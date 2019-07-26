---
title: "Python 中字符串的内置方法"
date: 2019-04-15T11:45:00Z
lastmod: 2019-04-15T16:47:31Z
draft: false
keywords: ["Python"]
description: "Python 中字符串的内置方法总结"
tags: ["Python"]
categories: ["Python"]
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
Python 中字符串的内置方法作用用例总结
<!--more-->
## Python 中字符串的内置方法

#### 1. 大小写转换
##### `capitalize()`

   把字符串第一个字符改为大写

   ```python
   >>> str='apple'
   >>> str.capitalize()
   'Apple'
   ```

##### `lower()`

   转换字符串中所有大写字符为小写

   ```python
   'ApP1E'.lower()
   'app1e'
   ```

##### `casefold()`

   把整个字符串的所有字符改为小写

   ```python
   >>> 'ApP1E'.casefold()
   'app1e'
   ```

##### `upper()`

   把整个字符串中所有字符改为大写

   ```python
   >>> "ApP1e".upper()
   'APP1E'   
   ```

##### `swapcase()`

   反转字符串中的大小写字符

   ```python
   >>> "ApP1E".swapcase()
   'aPp1e'
   ```


#### 2. 左右填充空格
##### `center(width)`

   把字符串居中显示，两边用空格填充至 width 指定长度

   ```python
   >>> str='apple'
   >>> str.center(10)
   '  apple   '
   ```

   width 小于字符串长度时，显示不变。优先把空格加在源字符串右边

##### `ljust(width)`

   返回一个左对齐的字符串，并在右边填充空格变为长度为 width 的新字符串

   ```python
   >>> "apple".ljust(10)
   'apple     '
   ```

##### `rjust(width)`

   返回一个右对齐的字符串，并在左边填充空格变为长度为 width 的新字符串

   ```python
   >>> "apple".rjust(10)
   '     apple'
   ```

#### 3. 删除左右空格

##### `strip([chars])`

   去掉字符串左右两边所有空格，chars 参数可以定制空格为其他待删除的字符

   ```python
   >>> "000 apple00 0".strip('0 ')
   'apple'
   ```

##### `lstrip()`

   去掉字符串左边的所有空格

   ```python
   >>> str = "apple".rjust(10)
   >>> str.lstrip()
   'apple'
   ```

##### `rstrip()`

   去掉字符串右边的所有空格

   ```python
   >>> str = "apple".ljust(10)
   >>> str.rstrip()
   'apple'
   ```

#### 4. 子字符串计数
##### `count(sub[,start[,end]])`

   返回 sub 在字符串内出现的次数，后面可选参数控制查找范围（0~n）

   ```python
   >>> str='apple'
   >>> str.count('p')
   2
   >>> str.count('p',2)
   1
   ```


#### 5. 编码字符串
##### `encode(encoding='uft-8',errors='strict')`

   以 encofing 指定的编码格式对字符串进行编码

   ```python
   >>> str = 'apple'
   >>> str.encode(encoding='utf8',errors='strict')
   b'apple'
   ```



#### 6. 查找匹配开头或者结尾
##### `endswitch(sub[,start[,end]])`

   检查字符串是否以 sub 字串结尾，如果是则返回 True，否则返回 False

   ```python
   >>> str = 'apple'
   >>> str.endswith('e')
   True
   ```

##### `startswith(prefix[,start[,end]])`

   检查字符串是否以 prefix 为开头，如果是则返回 True，否则返回 False

   ```python
   >>> "Apple".startswith('A')
   True
   ```

#### 7. 切换制表符(tab)
##### `expandtabs([tabsize=8])`

   把字符串中的制表符（`\t`）切换成空格，默认为 8 个空格。

   ```python
   >>> str
   'An\tapple'
   >>> str.expandtabs(tabsize=4)
   'An   apple'
   >>> print(str)
   An	apple
   ```


#### 8. `查找子字符串
##### `find(sub[,start[,end]])`

   检查 sub 是否存在，存在则返回找到的第一个索引值，不存在则返回"-1"

   ```python
   >>> str = "apple"
   >>> str.find('p',2)
   2
   >>> str.find('p',3)
   -1
   ```

##### `rfind(sub[,start[,end]])`

   类似上面办法，从右边开始查找

#### 9. 定位子字符串位置
##### `index(sub[,start[,end]])`

   同`find()`类似，但是当不存在 sub 时，会报错

   ```python
   >>> str = "apple"
   >>> str.index('p')
   1
   >>> str.index('p',3)
   Traceback (most recent call last):
     File "<stdin>", line 1, in <module>
   ValueError: substring not found
   ```

##### `rindex(sub[,start[,end]])`

   类似上面办法，从右边开始查找

#### 10. 匹配类型检查
##### `isalnum()`

   如果字符串非空，且都是由字母或者数字组成，返回 True，否则 false

   ```python
   >>> str="1 Apple"
   >>> str.isalnum()
   False
   >>> "1Apple".isalnum()
   True
   ```

##### `isalpha()`

   如果字符串非空，且所有字符都是字母则返回 True，否则 false

   ```python
   >>> "1Apple".isalpha()
   False
   >>> "Apple".isalpha()
   True
   ```

##### `isdecimal()`

   如果字符串非空，且只包含十进制数字，则返回 True，否则 false

##### `isdigit()`

   如果字符串非空，且只包含数字，则返回 True，否则 false

##### `isnumeric()`

   如果字符串非空，且只包含数字字符，则返回 True，否则 false

   ```
   isdigit()
   True: Unicode数字，byte数字（单字节），全角数字（双字节），罗马数字
   False: 汉字数字
   Error: 无

   isdecimal()
   True: Unicode数字，，全角数字（双字节）
   False: 罗马数字，汉字数字
   Error: byte数字（单字节）

   isnumeric()
   True: Unicode数字，全角数字（双字节），罗马数字，汉字数字
   False: 无
   Error: byte数字（单字节）
   ```

   测试：

   ```python
   >>> num = "0"
   >>> num.isdigit()
   True
   >>> num.isdecimal()
   True
   >>> num.isnumeric()
   True

   >>> num = b"0"  # byte
   >>> num.isdigit()
   True
   >>> num.isdecimal()
   Traceback (most recent call last):
     File "<stdin>", line 1, in <module>
   AttributeError: 'bytes' object has no attribute 'isdecimal'
   >>> num.isnumeric()
   Traceback (most recent call last):
     File "<stdin>", line 1, in <module>
   AttributeError: 'bytes' object has no attribute 'isnumeric'

   >>> num = "零"
   >>> num.isdigit()
   False
   >>> num.isdecimal()
   False
   >>> num.isnumeric()
   True
   ```
   罗马字测试失败



#### 11. 大小写检查
##### `islower()`

   如果字符串中至少包含一个区分大小写的字符，且这些字符都是小写，则返回 True，否则 false
   ```python
   >>> str = "apple"
   >>> str = "apple123pencil"
   >>> str.islower()
   True
   ```

##### `isupper()`

   如果字符串中至少包含一个区分大小写的字符，且这些字符都是大写，则返回 True，否则 false

#### 12. 匹配全空格
##### `isspace()`

   如果字符串中只包含空格，则返回 True，否则 false
   ```python
   >>> "".isspace()
   False
   >>> " ".isspace()
   True
   ```

#### 13. 匹配标题格式
##### `istitle()`

   如果字符串为标题格式（所有单词除了大写开头，其他都是小写），则返回 True，否则 false

   ```python
   >>> "An Apple".istitle()
   True
   >>> "An AppLe".istitle()
   False
   >>> "An App2e".istitle()
   False
   >>> "An Apple435".istitle()
   True
   ```

##### `title()`

   返回标题化的字符串

   ```python
   >>> 'An ApP1e'.title()
   'An App1E'
   ```

#### 14. 插入
##### `join(sub)`

   把 字符串插入到 子字符串 sub 中

   ```python
   >>> "apple".join('0 0')
   '0apple apple0'
   ```

#### 15. 查找分割成新序列
##### `partition(sub)`

   找到 子字符串 sub ，然后把字符串分成一个三元组（`'pre_sub'`,`'sub'`,`'fol_sub'`），如果不包含 sub ，则返回（`'原字符串`,`''`,`''`）

   ```python
   >>> "apple".partition('p')
   ('a', 'p', 'ple')
   ```

   中间一定是匹配到的 `sub` 或者 `''`

##### `rpartition(sub)`

   类似于 `partition(sub)`，从右开始匹配

#### 16. 替换子字符串
##### `replace(old,new[,count])`

   把字符串中的 old 子字符串 替换成  new 子字符串，count 控制替换的次数

   ```python
   >>> "apple".replace('a','A')
   'Apple'
   >>> "apple".replace('a','A',0)
   'apple'
   ```

#### 17. 分割字符串
##### `split(sep=None,maxplit=-1)`

   以 sep 分割字符串，不指定时为空格，maxsplit 可以设置分割次数，最后返回一个列表

   ```python
   >>> "Apple and pencil ".split()
   ['Apple', 'and', 'pencil']
   >>> "Apple and pencil ".split(sep='p')
   ['A', '', 'le and ', 'encil ']
   >>> "Apple and pencil ".split(sep='p',maxsplit=1)
   ['A', 'ple and pencil ']
   ```


##### `splitlines([keepends])`

   以换行符分割字符串，返回一个列表，keepends 控制是否显示换行符

   ```python
   >>> 'ab c\n\nde fg\rkl\r\n'.splitlines()
   ['ab c', '', 'de fg', 'kl']
   >>> 'ab c\n\nde fg\rkl\r\n'.splitlines(keepends=True)
   ['ab c\n', '\n', 'de fg\r', 'kl\r\n']
   ```


##### `translate(table[,deletechars=""])`

   按照 table 给出的表，转换字符串的字符，deletechars 可以指定需要过滤掉的字符列表（python2 和 3 版本不同）

```python
>>> intab = "abcdefghijklmnopqrstuvwxyz"
>>> outtab = intab.upper()
>>> tabtrans = str.maketrans(intab,outtab)
>>> "apple".translate(tabtrans)
'APPLE'
'''  过滤字符例子测试不成功 '''
```

#### 18. 填充0
##### `zfill(width)`

   返回字符串长度为 width 的字符串，原字符串右对齐，左边用 "0" 填充

   ```python
   >>> "apple".zfill(10)
   '00000apple'
   >>> type("0123".zfill(10))
   <class 'str'>
   ```

##### 参考链接

- [python中str函数isdigit、isdecimal、isnumeric的区别](https://www.cnblogs.com/jebeljebel/p/4006433.html)
- [Python splitlines()方法](http://www.runoob.com/python/att-string-splitlines.html)
- [Python3 字符串](http://www.runoob.com/python3/python3-string.html)
- [课时14：字符串：各种奇葩的内置方法](https://www.cnblogs.com/DC0307/p/9408848.html)
