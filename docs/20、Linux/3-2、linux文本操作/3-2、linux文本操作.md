## 开头

对于文本操作可以进行搜索，主要涉及到命令为grep和find

对于文本编辑有vim、sed、awk。

Vim和Sed、AWK的区别主要有：

* 交互式与非交互式
* 文件操作模式与行操作模式



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

* 遍历文件列表

  示例：touch /tmp/{1..9}.txt

  创建1到9的命令

  ls /tmp/*.txt

  cd /tmp

  find *txt

  同ls /tmp/\*.txt查找结果类似

* 查询并删除

  find *txt -exec rm -v { } \;

* grep pass /root/anconda-ks.cfg | cut -d " " -f 1

  对grep找到的内容进行剪切以空格为分割并找到第一个字段 

  查找的内容 auth

* cut -d ":" -f7 /etc/passwd

  按照:进行切割输出第7列

* cut -d ":" -f7 /etc/passwd | sort | uniq -c

  按照:进行切割输出第7列排序

* cut -d ":" -f7 /etc/passwd | sort | uniq -c | sort -r

  并排序

sed

sed替换的概念分为：

- sed的模式空间
- 替换命令s

基本工作方式是，将文件以行为单位读取到内存(模式空间)。然后使用sed的每个脚本对该行进行操作。最终处理完成后输出该行

### sed单行

sed 一般用于对文本内容做替换。sed ‘/user1/s/user1/u1/’ /etc/passwd

#### 标准命令

* sed ‘s/old/new/’ filename

  sed只替换了一次

* sed 's!/!abc!' afile

  由于要替换/，因此需要更换分隔符

* sed -e ‘s/old/new/’ -e ‘s/old/new/’ filename 

  -e 接收多个指令

* sed 's/a/aa/;s/aa/bb/' afile 

  多个指令的简写方式

* sed -i ‘s/old/new’ ‘s/old/new’ filename

   -i 替换并写回写入原文件

* sed 's/a/aa/;s/aa/bb/' afile > bfile

  新建另一文件

* head -5 /etc/passwd | sed 's/...//'

  相当于删除每行的前三个字符

* head -5 /etc/passwd | sed 's/s*bin//'

  删除s*bin匹配的字符

* grep root /etc/passwd | sed 's/^root//'

  删除以root开头的root

#### 正则表达式

新建文件

```
b
a
aa
aaa
ab
abb
abbb
```

* sed 's/ab*/!/'

  前面一个字符出现零次或多次

  输出

  ```
  b
  !
  !a
  !aa
  !
  !
  !
  #
  ```

* sed -r 's/ab+/!/' bfile

  前面一个字符出现一次或多次

  ```
  b
  a
  aa
  aaa
  !
  !
  !
  ```

* sed -r 's/ab?/!/' bfile

  前面出现零次或一次

  ```
  b
  !
  !a
  !aa
  !
  !b
  !bb
  ```

* sed -r 's/a|b/!/' bfile

  | 或者

  ```
  !
  !
  !a
  !aa
  !b
  !bb
  !bbb
  ```

* sed -r 's/(aa)|(bb)/!/' bfile

  使用|时多个字符需要使用()

  ```
  b
  a
  !
  !a
  ab
  a!
  a!b
  ```

#### 回调功能

sed -r 's/(a.*b)/\1:\1/' cfile

sed回调功能 \1表示分组的第一个元素

源文件

```
axyzb
```

生成

```
axyzb:axyzb
```

#### 全局替换

**s/old/new/g**

g 为全局替换，用于替换所有出现的次数

/ 如果和正则匹配的内容冲突可以使用其他符号，如：s@old@new@g

* head -5 /etc/passwd | etc 's/root/!!!!/'

  默认替换第一个

* head -5 /etc/passwd | etc 's/root/!!!!/g

  全局替换

* head -5 /etc/passwd | etc 's/root/!!!!/2'

  只匹配第2次

* head -5 /etc/passwd | etc 's/root/!!!!/n'

  只匹配第n次

#### 标志位

基本语法：s/old/new/标志位

- 数字，第几次出现才进行替换
- g，每次出现都进行替换
- p 打印模式空间的内容
  - sed -n ‘script’ filename 阻止默认输出
- w file 将模式空间的内容写入到文件

示例：

* head -5 /etc/passwd | sed 's/root/!!!!/p'

  处理后的输出，不处理的原本输出

* head -5 /etc/passwd | sed -n 's/root/!!!!/p'

  只输出替换后的内容

* head -5 /etc/passwd | sed -n 's/root/!!!!/w tmp/a.txt'

  保存到文件

#### 行筛选

默认对每行进行操作，增加寻址后对匹配的行进行操作（筛选一些行）。

- /正则表达式/s/old/new/g
- 行号s/old/new/g
  - 行号可以是具体的行，也可以是最后一行$符号
- 可以使用两个寻址符号，也可以混合使用行号和正则地址

注：正则表达式和行号是可以混合使用的

示例：

* head -6 /etc/passwd | sed '1s/adm/!/'

  第一行替换

* head -6 /etc/passwd | sed '1,3s/adm/!/'

  第一行到第三行替换

* head -6 /etc/passwd | sed '1,$s/adm/!/'

  第一行到最后一行替换

* head -6 /etc/passwd | sed '/root/s/adm/!/' 

  root行的bash进行替换

* head -6 /etc/passwd | sed '/^bin/,$s/nologin/!/'

#### 分组

- 寻址可以匹配多条命令
- /regular/{s/old/new/;s/old/new/}

#### 脚本文件

- 可以将选项保存为文件，使用-f加载脚本文件
- sed -f sedscript filename

#### 删除

[寻址]d

删除模式空间内容，改变脚本的控制流，读取新的输入行（d匹配后整行都会被删除）

输入文件示例

```
b
a
aa
aaa
ab
abb
abbb
```

* sed '/ab/d' bfile

  ```
  b
  a
  aa
  aaa
  ```

  匹配的行都删除了

* sed '/ab/d;s/a/!/' bfile

  删除之后的内容不会被改变(改变控制流)

  ```
  b
  !
  !a
  !aa
  ```

* sed '/ab/d;=' bfile

  =为打印行号

  ```
  1
  b
  2
  a
  3
  aa
  4
  aaa
  ```

#### 追加插入和更改

- 追加命令 a
- 插入命令 i
- 更改命令 c

输入文件

```
b
a
aa
aaa
ab
abb
abbb
```

示例：

* sed 'ab/i hello' bfile

  只要匹配到ab就会插入hello,上一行插入

  ```
  b
  a
  aa
  aaa
  hello
  ab
  hello
  abb
  hello
  abbb
  ```

* sed 'ab/a hello' bfile

  只要匹配到ab就会插入hello,下一行插入

  ```
  b
  a
  aa
  aaa
  ab
  hello
  abb
  hello
  abbb
  hello
  ```

* sed 'ab/c hello' bfile

  只要匹配到ab就会改写成hello

  ```
  b
  a
  aa
  aaa
  hello
  hello
  hello
  ```

* sed 'ab/r afile' bfile

  当遇到bfile中的ab时添加afile里面的内容

  ```
  b
  a
  aa
  aaa
  ab
  bb a a
  abb
  bb a a
  abbb
  bb a a
  ```

#### 读文件和写文件

* 读文件命令r
* 写文件命令w

#### 下一行

- 下一行命令 n
- 打印行号命令 =

#### 打印

打印命令 p

* sed '/ab/p' bfile

  把匹配的行进行输出(匹配到的行再输出一次)

  ```
  b
  a
  aa
  aaa
  ab
  ab
  abb
  abb
  abbb
  abbb
  ```

* sed -n '/ab/p' bfile

  只输出匹配的行

#### 退出命令

比较命令效率

- sed 10q filename

  扫描到10行就退出

  在不完全扫描整个文本文件就可以退出

- sed -n 1,10p filename

q的指令性能高于p，q只读取对应的行数到内存中

示例：

* seq 1 10

  产生1至10的数字

* seq 1 1000000 > lines.txt

* wc -l lines.txt 

  1000000 lines.txt

* sed -n 1,10p lines.txt

* time sed -n '1,10p' lines.txt

  0.118s

* time sed -n '10q' lines.txt

  0.003s

#### sed多行

配置文件一般为单行出现，但也有使用XML或JSON格式的配置文件，为多行出现。

多行匹配命令

- N 将下一行加入到模式空间
- D 删除模式空间中的第一个字符到第一个换行符
- P 打印模式空间中的第一个字符到第一个换行符

示例：

a.txt文本：

```
hel
lo
```

* sed 'N;s/hel\nlo/!!!/' a.txt

  输出!!!

* sed 'N;s/hel.lo/!!!/' a.txt

  输出!!!

  因为使用.来匹配换行符

* cat > b.txt << EOF

  输入重定向,控制台输入：

  ```
  > hell
  > o bash hel
  > lo bash
  > EOF
  ```

* cat b.txt

  ```
  hell
  o bash hel
  lo bash
  ```

* sed 'N;s/\n//;s/hello bash.hello sed\n/;P;D' b.txt

  将换行符删除

* sed 'N;N;s/\n//;s/hello bash.hello sed\n/;P;D' b.txt

  每三行进行处理

  ```bash
  a.txt 内容如下
  1
  2
  3
  4
  5
  6
  7
  8
  9
  
  # ------------------- 示例1 ---------------------#
  sed 'N;N;s/\n/\t/g;' a.txt
  1    2    3
  4    5    6
  7    8    9
  ```

* sed 's/^\s*//;N;s/\n//;s/hello bash/hello sed\n/;P;D;' b.txt

  输出：

  ```
  hello sed
  hello sed
  ```

  示例 hello bash 替换成 hello sed 

* sed -n 'P;N;s/\n/\t/;s/^/\n/;D' a.txt

  利用 D 改变控制流

  ```bash
  1
  1    2
  1    2    3
  1    2    3    4
  1    2    3    4    5
  1    2    3    4    5    6
  1    2    3    4    5    6    7
  1    2    3    4    5    6    7    8
  1    2    3    4    5    6    7    8    9
  ```

#### sed保持空间

保持空间也是多行的一种操作方式，将内容暂存在保持空间，便于做多行处理，即在模式空间的同时再开辟一段内存空间

基本命令：

- h和H将模式空间内容存放到保持空间
- g和G将保持空间内容取出到模式空间
- x交换模式空间和保持空间内容

**小写的h、g是覆盖模式，大写的H、G是追加模式。可应用在行之间顺序的互换。**

示例1：

实现行数翻转

方法1：

* head -6 /etc/passwd | cat -n | tac

方法2：

* cat -n /etc/passwd | head -6 | sed -n '1h;1!G;$!x;$p'

  翻转第一行到第六行

* cat -n /etc/passwd | head -6 | sed -n 'G;h'

  上述一样的功能实现

* cat -n /etc/passwd | head -6 | sed -n 'G;h;$p'

* cat -n /etc/passwd | head -6 | sed -n '1!G;h;$p'

* cat -n /etc/passwd | head -6 | sed '1!G;h;$!d'

示例2：

```bash
# 下述多个都实现了 tac 倒序显示的效果
# 思路: 每次将本轮正确的结果保存在保持空间
cat -n /etc/passwd | head -n 6 | sed -n '1!G;$!x;$p'
//第一行不获取,最后一行不交换，只打印最后一行即可
cat -n /etc/passwd | head -n 6 | sed -n '1!G;h;$p'
cat -n /etc/passwd | head -n 6 | sed '1!G;h;$!d'
cat -n /etc/passwd | head -n 6 | sed '1!G;$!x;$!d'
cat -n /etc/passwd | head -n 6 | sed -n '1h;1d;G;h;$p';
cat -n /etc/passwd | head -n 6 | sed -n '1h;1!G;h;$p';

sed '=;6q' /etc/passwd | sed 'N;s/\n/\t/;1!G;h;$!d'

# --------------------- 显示结果 --------------------#
6    sync:x:5:0:sync:/sbin:/bin/sync
5    lp:x:4:7:lp:/var/spool/lpd:/sbin/nologin
4    adm:x:3:4:adm:/var/adm:/sbin/nologin
3    daemon:x:2:2:daemon:/sbin:/sbin/nologin
2    bin:x:1:1:bin:/bin:/sbin/nologin
1    root:x:0:0:root:/root:/bin/bash
```

## awk

AWK一般用于对文本内容进行统计、按需要的格式进行输出

- cut 命令 ：cut -d : -f 1 /etc/passwd
- AWK命令: awk -F: ‘/wd$/{print $1}’ /etc/passwd

### awk 和 sed 的区别

- awk 更像是**脚本语言**
- awk 用于**"比较规范"**的文本处理, 用于**统计数量**并调整顺序, **输出指定字段**.
- 使用 sed 将不规范的文本处理为"比较规范"的文本

### awk 脚本的流程控制

* BEGIN{}` 输入数据前例程, 可选

* `{}` 主输入循环

* `END{}` 所有文件读取完成例程

上述流程并非都需要完整写完，一般可直接书写主输入循环。

### 字段切割

- 每行称作AWK的记录
- 使用空格、制表符分隔开的单词称作字段
- 可以自己指定分隔的字段

#### 字段引用

- awk中使用$1、$2…$n表示每一个字符

  awk ‘{ print $1,$2,$3}’ filename

- awk可以使用-F选项改变字段分隔符

  - awk -F ‘,’ ‘{ print $1,$2,$3}’ filename
  - 分隔符可以使用正则表达式

#### 引用示例

* awk -F "'" '/^menu/{ print $2 }' /boot/grub2/grub.cfg

  测试取出指定的内核

* awk -F "'" '/^menu/{ print x++,$2 }' /boot/grub2/grub.cfg

  显示行号，默认变量为0

### awk表达式

### 数组