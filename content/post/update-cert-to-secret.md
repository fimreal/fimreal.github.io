---
title: "改进 lego 自动更新证书到 secret"
date: 2021-10-18T18:27:26+08:00
lastmod: 2021-10-18T18:27:26+08:00
draft: false
keywords: ["cert","kubernetes","rbac"]
description: "配置自动更新域名证书，同时更新到指定的 K8s 的 secret 中。"
tags: ["kubernetes","rbac","cronjob"]
categories: ["kubernetes"]
author: "Fimreal"

# You can also close(false) or open(true) something for this content.
# P.S. comment can only be closed
comment: true
toc: true
autoCollapseToc: true
postMetaInFooter: false
hiddenFromHomePage: false
# You can also define another contentCopyright. e.g. contentCopyright: "This is another copyright."
contentCopyright: false
reward: false
mathjax: false
mathjaxEnableSingleDollar: false
mathjaxEnableAutoNumber: false

# You unlisted posts you might want not want the header or footer to show
hideHeaderAndFooter: false

# You can enable or disable out-of-date content warning for individual post.
# Comment this out to use the global config.
#enableOutdatedInfoWarning: false

flowchartDiagrams:
  enable: false
  options: ""

sequenceDiagrams:
  enable: false
  options: ""

---

配置自动更新域名证书，同时更新到指定的 K8s 的 secret 中。 

<!--more-->

近期把服务器上内容统计丢到了 k3s 集群内，彻底用 ingress 替代了原本的 openresty 作为服务入口，很多用到证书的服务为了方便 POD 漂移，也都改造利用 secret 进行挂载。

由于要涉及到更新 secret 的步骤，之前更新证书使用的定时任务，或者特制容器做 daemon 的形式有点跟不上了，所以我又~~又又~~转回头测试了一把 `cert-manager`。不出意料，这越做越复杂的证书更新系统又一次没有跑起来。于是我坚定了我之前的想法，这工具门槛太高，就不是给个人用户使用的！一个基础的功能实现竟差到让我在两年内试了多次连 demo 都跑不起来，不看文档甚至不知道怎么去 debug。而它的文档结构又很机械化，根本不适合作为手册来使用，要我说是非常不推荐使用。

Ingress 其实有自动 reload 的功能，修改 secret 之后会自动更新证书配置。所以我只要利用 client-go 调 kubernetes 的 API 来自动化更新证书，剩下的找到一个支持 acme 的工具稍微改造下就好了。



## 1. 确定 ACME 工具

直接从老朋友 let‘sencrypt 文档下手，找到了各种语言写的客户端：

https://letsencrypt.org/docs/client-options/

bash 有之前用了很久的 `acme.sh`，如果统一用 shell 来写对运维人来说很简单，我只需要用 golang 多加一个读取证书并添加到 secret 的小程序就可以。

而Go 写的 `lego` 支持的 dns 很多看起来也不错，我能稍微改动下源码，添加上我想要的小功能，打包成一个二进制文件是最完美的。

nginx lua 的在 openresty 上测试了两个都没申请下来，想不到应该 debug 打印哪些信息，似乎需要对证书还有 http 等有比较深的理论知识，暂且放弃。

## 2. 尝试：利用 client-go 生成 secret 

先考虑万能方法，用 `client-go` 实现创建 secret，这样不管是使用 `acme.sh` 还是 `lego` 都可以很简单的整合在一起。

按照 client-go 的 example，读取 k8s 配置文件有两种方法，在集群内直接使用 `InClusterConfig()`，集群外要获取到 kubeconfig 配置。

简化一下代码如下：

```go
func NewKubeClient() (*kubernetes.Clientset, error) {
	// in-cluster
	config, err := rest.InClusterConfig()
	if err != nil {
		log.Println(err.Error())
	} else {
		return kubernetes.NewForConfig(config)
	}

	// read kubeconfig
	log.Println("Fallthrough...尝试读取本地配置文件")
	if Conf.Kubeconfig == "" {
		log.Fatalln("配置文件中未指定 kubeconfig 位置")
	}
	kubeconfig := &Conf.Kubeconfig

	config, err = clientcmd.BuildConfigFromFlags("", *kubeconfig)
	if err != nil {
		log.Fatalln(err.Error())
	}
	return kubernetes.NewForConfig(config)
}
```

创建和升级 secret，没有找到合适的例子，很简陋的代码：

```go

func UpdateSecret(clientset *kubernetes.Clientset, Conf *config.Config) (err error) {

	// create secret struct
	newSecret := v1.Secret{
		TypeMeta:   metav1.TypeMeta{Kind: "Secret", APIVersion: "v1"},
		ObjectMeta: metav1.ObjectMeta{Name: Conf.SecretName, Namespace: Conf.SecretNamespace},
		Data:       map[string][]byte{},
		StringData: map[string]string{},
		Type:       "kubernetes/tls",
	}
	newSecret.Data["tls.key"], err = ioutil.ReadFile("certificates/tls.key")
	if err != nil {
		return
	}
	newSecret.Data["tls.crt"], err = ioutil.ReadFile("certificates/tls.crt")
	if err != nil {
		return
	}

	// update secret
	secretOut, err := clientset.CoreV1().Secrets(Conf.SecretNamespace).Update(context.TODO(), &newSecret, metav1.UpdateOptions{})
	logrus.Debug(secretOut)
	if err != nil {
		logrus.Errorln(err)
	} else {
		return
	}

	// create secret
	secretOut, err = clientset.CoreV1().Secrets(Conf.SecretNamespace).Create(context.TODO(), &newSecret, metav1.CreateOptions{})
	logrus.Debug(secretOut)
	if err != nil {
		logrus.Errorln(err)
	}
	return

}
```

最后再加上调用外部命令比如 `lego` 来创建证书，把打包出来二进制文件添加到 `lego` 的镜像里，修改启动入口就完成了。

些许波折不谈，配置好权限，添加定时任务，更新成功。

## 3. 最终方案：修改 lego 添加创建 secret 部分

`lego` 文档写的的 library 例子，属实看不懂。简单点，直接 fork 了一份，在原来基础上添加功能。

`lego` 入口在 `cmd/lego/main.go`，里面使用 `urfave/cli`来生成配置 cli 的各种参数。

按照惯例，找到 `cmd/cmd_run.go` 文件，发现 `func run(ctx *cli.Context) error `也就是生成证书用到的子命令函数。

简单查了下 `urfave/cli` 的用法，在函数内直接用 `ctx.GlobalString("arg")`就可以获取到参数值，在 `run()`开头加入 `log.Fatal("hello, exit")`，执行 lego 创建证书，验证输出信息正常返回，证明这里就是要添加方法的位置。

#### 3.1 首先添加我们想要的参数

找到 `cmd/flags.go` 文件，在最后添加：

```go
		cli.StringFlag{
			Name:  "apply-to-secret",
			Usage: "Apply certificate to k8s secret. Format: {namespace}/{secretname}",
		},
```

也就是说，用 `--apply-to-secret` 来指定创建 secret 的命名空间和名字。

#### 3.2 添加生成 secret 方法的入口

回到之前的 `cmd/cmd_run.go` 文件，在 `run()` 函数最后返回前添加：

```go
	secretName := ctx.GlobalString("apply-to-secret")
	if secretName != "" {
		log.Infof("准备将证书部署到 secret[%s]...", secretName)
		if err := ksecret.DeployToSecret(&secretName, cert); err != nil {
			log.Fatal("部署到 secret 出现错误：", err)
		}
		log.Infof("部署成功！\n")
	}
```

`cert` 是`lego`自有的对象，里面包含了生成证书的所有信息。这里用`ksecret.DeployToSecret()`来部署证书到 secret。

#### 3.3 创建部署证书模块 ksecret

在项目目录下新建 `ksecret`目录，这里存放和 k8s API 交互的代码。

创建连接配置文件 `kubeClientSet.go`，这里只考虑创建 cronjob 的情况，启动的容器已经有相应的 RBAC 权限来修改生成 secret，所以不再添加配置文件的加载方式。

```go
package ksecret

import (
	"log"

	"k8s.io/client-go/kubernetes"
	"k8s.io/client-go/rest"
)

func NewKubeClient() (*kubernetes.Clientset, error) {
	config, err := rest.InClusterConfig()
	if err != nil {
		return nil, err
	} else {
		log.Println("成功获取到 kubernetes 配置")
		return kubernetes.NewForConfig(config)
	}
}
```

创建生成证书部分 `updateSecret.go`。万事具备，这里读取 `cert` 中的证书内容，拿到传参进来的 secret 名字就可以执行创建/更新操作了。

```go
package ksecret

import (
	"context"
	"strings"

	"github.com/go-acme/lego/v4/certificate"
	v1 "k8s.io/api/core/v1"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/client-go/kubernetes"
)

type Secret struct {
	SecretName      string
	SecretNamespace string
	Crt             []byte
	Key             []byte
}

func DeployToSecret(secretName *string, cert *certificate.Resource) (err error) {
	clientset, err := NewKubeClient()
	if err != nil {
		return err
	}
	sName := strings.Split(*secretName, "/")
	s := Secret{
		SecretName:      sName[1],
		SecretNamespace: sName[0],
		Crt:             cert.Certificate,
		Key:             cert.PrivateKey,
	}
  
	return UpdateSecret(clientset, &s)
}

func UpdateSecret(clientset *kubernetes.Clientset, s *Secret) (err error) {

	// create secret struct
	newSecret := v1.Secret{
		TypeMeta:   metav1.TypeMeta{Kind: "Secret", APIVersion: "v1"},
		ObjectMeta: metav1.ObjectMeta{Name: s.SecretName, Namespace: s.SecretNamespace},
		Data:       map[string][]byte{},
		StringData: map[string]string{},
		Type:       "kubernetes/tls",
	}
	newSecret.Data["tls.key"] = s.Key
	newSecret.Data["tls.crt"] = s.Crt

	// update secret
	_, err = clientset.CoreV1().Secrets(s.SecretNamespace).Update(context.TODO(), &newSecret, metav1.UpdateOptions{})
	if err == nil {
		return
	}

	// create secret
	_, err = clientset.CoreV1().Secrets(s.SecretNamespace).Create(context.TODO(), &newSecret, metav1.CreateOptions{})
	if err != nil {
		return err
	}
	return
}
```

最终修改保存在：

https://github.com/fimreal/lego/tree/a412c04c33e1e46f4df9d7a57774704fe068f039

#### 3.4 打包成容器镜像

自己使用，就不考虑 github action 和 makefile 来创建了，直接手动打包：

```bash
mkdir builds
cd builds
CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build -o ./lego -ldflags '-s -w' ../cmd/lego/main.go
```

按照原来的 Dockerfile，稍作修改

```dockerfile
FROM alpine
RUN apk update \
    && apk add --no-cache ca-certificates tzdata \
    && update-ca-certificates

COPY lego /usr/bin/lego

ENTRYPOINT [ "/usr/bin/lego" ]
```

打包上传

```bash
docker build -t epurs/updatecerts:n .
docker push epurs/updatecerts:n
```

至此，大功告成。

## 4. 创建 cronjob 定时更新证书

除了按照 `lego` 原命令创建证书方式添加参数，还要创建对应的 RBAC 权限，yaml 文件可参考：

```yaml
# 不同命名空间需要额外修改 namespace 的 value
# 创建 sa 给 cronjob 使用

apiVersion: v1
kind: ServiceAccount
metadata:
  name: secrets-admin
  namespace: default

---

kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: secrets-admin
  namespace: default
rules:
- apiGroups: [""]
  resources: ["secrets"]
  # 添加需要的权限，可以有 get list watch create delete deletecollection patch update
  verbs: ["get", "list", "create", "update"]		

---

apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: secrets-admin
  namespace: default
subjects:
# 绑定给前面创建的 sa
- kind: ServiceAccount
  name: secrets-admin
  namespace: default
roleRef:
# 使用前面创建 role
  kind: Role
  name: secrets-admin
  apiGroup: rbac.authorization.k8s.io
  
---
apiVersion: batch/v1
kind: CronJob
metadata:
  name: update-cert-n
  namespace: default
spec:
  # 30 天执行一次
  schedule: "10 1 */30 * *"
  jobTemplate:
    spec:
      template:
        spec:
          tolerations:
          - key: "node-role.kubernetes.io/master"
            operator: "Exists"
          # 使用前面创建的 sa
          serviceAccountName: secrets-admin
          restartPolicy: Never
          containers:
          - name: update-cert-n
            image: epurs/updatecerts:n
            imagePullPolicy: Always
            args:
              - --dns.resolvers
              - "223.5.5.5"
              - -m
              - "acme@epurs.com"
              - --dns
              - "dnspod"
              - -d
              - "niml.ml"
              - -d
              - "*.n.niml.ml"
              - "--apply-to-secret"
              - "default/niml-ml"
              - -a
              -  run
            env:
            - name: DNSPOD_API_KEY
              valueFrom:
                secretKeyRef:
                  name: dnspod
                  key: userToken
---
apiVersion: v1
kind: Secret
metadata:
  name: dnspod
type: Opaque
data:
  userToken:  xxxxxxxxxx
---
```

apply 配置

```bash
# kubectl apply -f crt.yaml
serviceaccount/secrets-admin created
clusterrole.rbac.authorization.k8s.io/secrets-admin created
clusterrolebinding.rbac.authorization.k8s.io/secrets-admin created
cronjob.batch/update-cert-n created
secret/dnspod unchanged
```

手动触发定时任务，测试结果：

```bash
# kubectl create job --from=cronjob/update-cert-n update-crt-manually
job.batch/update-crt-manually created
# kubectl get job
NAME                   COMPLETIONS   DURATION   AGE
update-crt-manually    0/1           4s         4s
```

查看定时任务日志：

```bash
# kubectl logs -f update-crt-manually-kx9w7
2021/10/19 03:37:44 No key found for account acme@epurs.com. Generating a P256 key.
2021/10/19 03:37:44 Saved key to /.lego/accounts/acme-v02.api.letsencrypt.org/acme@epurs.com/keys/acme@epurs.com.key
2021/10/19 03:37:45 [INFO] acme: Registering account for acme@epurs.com
!!!! HEADS UP !!!!

Your account credentials have been saved in your Let's Encrypt
configuration directory at "/.lego/accounts".

You should make a secure backup of this folder now. This
configuration directory will also contain certificates and
private keys obtained from Let's Encrypt so making regular
backups of this folder is ideal.
2021/10/19 03:37:45 [INFO] [niml.ml, *.n.niml.ml] acme: Obtaining bundled SAN certificate
2021/10/19 03:37:47 [INFO] [*.n.niml.ml] AuthURL: https://acme-v02.api.letsencrypt.org/acme/authz-v3/41253422090
2021/10/19 03:37:47 [INFO] [niml.ml] AuthURL: https://acme-v02.api.letsencrypt.org/acme/authz-v3/41253422100
2021/10/19 03:37:47 [INFO] [*.n.niml.ml] acme: use dns-01 solver
2021/10/19 03:37:47 [INFO] [niml.ml] acme: Could not find solver for: tls-alpn-01
2021/10/19 03:37:47 [INFO] [niml.ml] acme: Could not find solver for: http-01
2021/10/19 03:37:47 [INFO] [niml.ml] acme: use dns-01 solver
2021/10/19 03:37:47 [INFO] [*.n.niml.ml] acme: Preparing to solve DNS-01
2021/10/19 03:37:48 [INFO] [niml.ml] acme: Preparing to solve DNS-01
2021/10/19 03:37:48 [INFO] [*.n.niml.ml] acme: Trying to solve DNS-01
2021/10/19 03:37:48 [INFO] [*.n.niml.ml] acme: Checking DNS record propagation using [223.5.5.5:53]
2021/10/19 03:37:50 [INFO] Wait for propagation [timeout: 1m0s, interval: 2s]
2021/10/19 03:37:51 [INFO] [*.n.niml.ml] acme: Waiting for DNS record propagation.
2021/10/19 03:37:53 [INFO] [*.n.niml.ml] acme: Waiting for DNS record propagation.
2021/10/19 03:37:55 [INFO] [*.n.niml.ml] acme: Waiting for DNS record propagation.
2021/10/19 03:37:57 [INFO] [*.n.niml.ml] acme: Waiting for DNS record propagation.
2021/10/19 03:37:59 [INFO] [*.n.niml.ml] acme: Waiting for DNS record propagation.
2021/10/19 03:38:15 [INFO] [*.n.niml.ml] The server validated our request
2021/10/19 03:38:15 [INFO] [niml.ml] acme: Trying to solve DNS-01
2021/10/19 03:38:15 [INFO] [niml.ml] acme: Checking DNS record propagation using [223.5.5.5:53]
2021/10/19 03:38:17 [INFO] Wait for propagation [timeout: 1m0s, interval: 2s]
2021/10/19 03:38:57 [INFO] [niml.ml] The server validated our request
2021/10/19 03:38:57 [INFO] [*.n.niml.ml] acme: Cleaning DNS-01 challenge
2021/10/19 03:38:57 [INFO] [niml.ml] acme: Cleaning DNS-01 challenge
2021/10/19 03:38:58 [INFO] [niml.ml, *.n.niml.ml] acme: Validations succeeded; requesting certificates
2021/10/19 03:38:59 [INFO] [niml.ml] Server responded with a certificate.
2021/10/19 03:38:59 [INFO] 准备将证书部署到 secret[default/niml-ml]...
2021/10/19 03:38:59 成功获取到 kubernetes 配置
2021/10/19 03:38:59 [INFO] 部署成功！

```

确认证书：

```bash
# kubectl get secret niml-ml
NAME      TYPE             DATA   AGE
niml-ml   kubernetes/tls   2      61s
```

搞定！

