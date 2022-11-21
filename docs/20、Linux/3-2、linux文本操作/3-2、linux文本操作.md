## 文本搜索

### 正则表达式

#### 元字符

- . 匹配除换行符外的任意单个字符

- \* 匹配任意一个跟在它前面的字符

  PS：通配符中*表示零个、单个或多个字符，?表示任意单个字符；在元字符中.*相当于通配符中的*

- [ ] 匹配方括号中的字符类中的任意一个

- ^ 匹配开头

- $ 匹配结尾

- \ 转义后面的特殊字符

#### 扩展元字符

- \+ 匹配前面的正则表达式至少出现一次
- ? 匹配前面的正则表达式出现零次或一次
- | 匹配它前面或后面的正则表达式

### grep

文本内容的过滤(查找)

* grep password /root/anaconda-ks.cfg

  匹配到的整行输出

* grep pass.... /root/anaconda-ks.cfg

  其中.代表单个字符

* grep pass....$ /root/anaconda-ks.cfg

  $表示字符匹配结束

* grep pass.* /root/anaconda-ks.cfg

  表示pass后匹配任意字符

* grep pass.*$ /root/anaconda-ks.cfg 

  $表示字符匹配结束

* grep "\\." /root/anaconda-ks.cfg

  避免.被shell解析

### find

文件名查找命令 find

* find password

  当前目录，查找内容 passwod

* find /etc -name password

  指定目录查找指定名称

  查询结果为：

  /etc/pam.d/passwd 

  /etc/passwd

* find /etc -name pass*

  通配符前缀

* find /etc -regex .*wd

  通配符后缀

* find /etc -type f -regex .*wd

  指定文件类型，使用regex正则表达式查询，.*wd表示任意字符加上wd后缀

* find /etc/ -atime 8

  按时间查找(atime 访问时间)

  还有时间*atime mtime ctime* 

* 