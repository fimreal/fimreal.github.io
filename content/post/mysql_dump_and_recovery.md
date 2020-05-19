---
title: "MySQL备份还原笔记"
date: 2020-05-14T03:30:00Z
draft: false
keywords: ["MySQL"]
description: "工作中常用到 MySQL 备份还原的一些命令，脚本，以及遇到的问题记录"
tags: ["MySQL"]
categories: ["database"]
author: "Fimreal"

# You can also close(false) or open(true) something for this content.
# P.S. comment can only be closed
comment: true
toc: true
autoCollapseToc: false
postMetaInFooter: false
hiddenFromHomePage: false

# You can also define another contentCopyright. e.g. contentCopyright: "This is another copyright."
contentCopyright: true
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
MySQL 备份时常用到的命令，脚本笔记。

<!--more-->
### 一、mysqldump 备份常用命令参数

#### 1. 基础备份命令

```bash
mysql [-h <host_address>] [-P <host_port>] <-u <mysql_username>> [-p[<mysql_password>]] [dump_args] ...[scheme_name] [table_name] > filename-$(date +%F).sql
```

`dump_args`可以为如下参数，不指定时默认参数受到变量控制，全部详情使用`mysqldump --help`查看：

1. `-A, --all-databases, --databases all`备份所有数据，默认为`FALSE`
2. `-B, --databases <db_name> [db_name1 db_name2...]`指定备份数据库，参数后面所有名字都被看作数据库名
3. `-C, --compress`客户端和服务端传输协议中启用压缩选项，默认为`FALSE`
4. `--default-character-set=utf8`修改默认字符集为`utf8`，注意 MySQL 里该字符集没有"-"
5. `--add-drop-database`导出语句中每个数据库结构创建前添加 DROP DATABASE 语句，默认为`FALSE`
6. `--add-drop-table`导出语句中每个表结构创建前添加 DROP TABLE 语句，默认为`TRUE`
7. `--skip-add-drop-table`导出语句中每个表结构创建前不添加 DROP TABLE 语句，相比 6 具有相反效果
8. `-d`只导出数据结构，不导出数据
9. `-t`只导出数据，不导出数据结构

#### 2. 其他

##### 查看数据库大小

```mysql
select
TABLE_SCHEMA,
concat(truncate(sum(data_length)/1024/1024,2),' MB') as data_size,
concat(truncate(sum(index_length)/1024/1024,2),'MB') as index_size
from information_schema.tables
group by TABLE_SCHEMA
ORDER BY data_size desc;
```

##### 查看数据库表空间大小

```mysql
select
TABLE_NAME,
concat(truncate(data_length/1024/1024,2),' MB') as data_size,
concat(truncate(index_length/1024/1024,2),' MB') as index_size
from information_schema.tables
where TABLE_SCHEMA = 'mysql'
group by TABLE_NAME
order by data_length desc;
```

### 二、MySQL 导入SQL 备份文件

#### 1. 从 MySQL 全库备份中恢复单个表的方法

##### 重命名当前表（可选）

```mysql
ALTER TABLE <table_name> RENAME to <table_name>_bak7777;
```

##### 从全备份文件中找出单个 **scheme SQL**

```mysql
sed -ne '/-- Current Database: `${scheme_name}`/,/-- Current Database/ p' alldump.sql > scheme.sql
```

##### 从单个 **scheme SQL**文件中找出单个表 **SQL**

为了避免从不同库中有同名表存在，尽量从单个 scheme 中找到表数据恢复

```mysql
sed -n -e '/-- Table structure for table `${table_name}`/,/-- Table structure for table/p' scheme.sql > table.sql
```

##### 恢复表

```mysql
mysql -h $host -P $port -u$username -p ${scheme_name} --max_allowed_packet=64M < table.sql
```



#### 2. MySQL 备份脚本

```bash
# filename: mysqldump_cron.sh
#!/usr/bin/env bash

# init
now_Date=$(date +%Y%m%d)
LOGFILE=${LOGFILE:-/tmp/mysqldump.log}
WORKDIR=${WORKDIR:-/data/mysqldump}

LOGSTATUS() {
  if [ $? -eq 0 ]; then
      echo $1 OK >> $LOGFILE
    # 删除旧的日志
	  # echo \
      find $WORKDIR -name "$1*.gz" -mtime +$2 -delete
  else
      echo $1 FAILD >> $LOGFILE
    # mail to somebody
  fi
}

DUMPSQL() {
  DUMPFILENAME=${2}-${1}
  if [ ${1} == "all" ]; then
      SCHEME_NAME="-A"
  fi
  shift
  MYSQL_HOST=${1}
  MYSQL_USERNAME=${2}
  MYSQL_PASSWORD=${3}
  MYSQL_PORT=${4:-3306}
  MYSQL_ARGS=${5}


  echo "Strating dump ${DUMPFILENAME} SQL at $(date)" >> ${LOGFILE}
  mysqldump -h ${MYSQL_HOST} -u ${MYSQL_USERNAME} -p${MYSQL_PASSWORD} -P${MYSQL_PORT} ${MYSQL_ARGS} ${SCHEME_NAME} |gzip > ${DUMPFILENAME}-${now_Date}.sql.gz 2>/dev/null
}

cd $WORKDIR

DUMPSQL all 172.16.225.47 root woshimima
LOGSTATUS 172.16.225.47 3


echo "======================" >> ${LOGFILE}
```





### 三、一些异常错误的解决办法

#### 1. 遇到报错：[MySQL ERROR 1231 (42000):Variable 'character_set_client' can't be set to the value of 'NULL'](https://stackoverflow.com/questions/29112716/mysql-error-1231-42000variable-character-set-client-cant-be-set-to-the-val) 时

客户端字符集设置问题，可以修改 SQL 文件内的 [最后]

`/*!40101 SET character_set_client = @saved_cs_client */;`

改成

`/*!40101 SET character_set_client = 'utf8' */;`

默认 dump 文件环境配置可以参考：

/*!40101 SET @OLD_CHARACTER_SET_CLIENT=@@CHARACTER_SET_CLIENT */;

/*!40101 SET @OLD_CHARACTER_SET_RESULTS=@@CHARACTER_SET_RESULTS */;

/*!40101 SET @OLD_COLLATION_CONNECTION=@@COLLATION_CONNECTION */;

/*!40101 SET NAMES utf8 */;

/*!40103 SET @OLD_TIME_ZONE=@@TIME_ZONE */;

/*!40103 SET TIME_ZONE='+00:00' */;

/*!40014 SET @OLD_UNIQUE_CHECKS=@@UNIQUE_CHECKS, UNIQUE_CHECKS=0 */;

/*!40014 SET @OLD_FOREIGN_KEY_CHECKS=@@FOREIGN_KEY_CHECKS, FOREIGN_KEY_CHECKS=0 */;

/*!40101 SET @OLD_SQL_MODE=@@SQL_MODE, SQL_MODE='NO_AUTO_VALUE_ON_ZERO' */;

/*!40111 SET @OLD_SQL_NOTES=@@SQL_NOTES, SQL_NOTES=0 */;



### Reference

https://stackoverflow.com/questions/29112716/mysql-error-1231-42000variable-character-set-client-cant-be-set-to-the-val

https://www.cnblogs.com/chenmh/p/5300370.html


