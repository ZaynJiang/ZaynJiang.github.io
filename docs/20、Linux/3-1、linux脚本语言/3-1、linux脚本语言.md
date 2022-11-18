## shell概念

Shell是命令解释器，用于解释用户对操作系统的操作。

Shell有很多，我们可以查看cat /etc/shells。CentOS 7 默认使用的Shell是bas

### Linux启动过程

- BIOS
- MBR
- BootLoader(grub)
- kernel(内存版本)
- systemd(1号进程)
- 系统初始化
- shell

PS：Linux dd 命令用于读取、转换并输出数据。

#### MBR

在硬盘中，硬盘的0柱面0磁头第一个1扇区称为主引导扇区，也叫主引导记录-MBR(main boot record)，其中MBR是以下三个部分组成

* Bootloader，主引导程序（446个字节）
* Dpt（Disk Partition table），硬盘分区表（64个字节）
* 扇区结尾标志（55aa）（个字节）

总共512字节，前446个字节是主引导记录，是bios加电自检后要运行的代码，中间64字节为分区表。
简单的来说MBR=bootloader+dpt(64)+结尾标志(55aa)。其中dpt磁盘分区表(64字节，每16个字节为一组，一共4组)

示例：

* dd if=/dev/sda of=mbr.bin bs=446 count=1

  假设时sda分区（硬盘的主引导记录）

* hexdump -C mbr.bin

  用16进制的形式查看（没有文件系统无法直接查看）

* dd if=/dev/sda of=mbr2.bin bs=512 count=1

  假设时sda分区（硬盘的主引导记录和磁盘的分区表）

* hexdump -C mbr2.bin | more

  用16进制的形式查看（最后面是55 aa）

#### grub

* cd /boot/grub2 

* ls 

* grub -editenv list 

  查看默认引导内核

* uname -r 

  简要查看

#### 一号进程

* which init

  查看一号进程

* cd /etc/rc.d

  CentOS6和CentOS7在这一块有巨大区别

* cd /etc/systemd/system/

  默认启动级别

### Shell脚本

- UNIX的哲学：**一条命令只做一件事**
- 为了组合命令和多次执行，使用脚本文件来保存需要执行的命令
- 赋予该文件执行权限（chmod u+rx filename）

#### 基本语法

- Sha-Bang

  在编写shell之前需要声明使用的是shell类型

  ```
  #!/bin/bash
  ```

  `;`可用于连接两个命令，这两个命令是彼此独立的，没有任何关联关系。

- 命令"`#`"号开头的注释

- chmod u+rx filename 可执行权限

#### 执行方式

- bash ./filename.sh

  使用bash的子进程开始执行(可以不用赋予权限，脚本执行不影响当前路径)

- ./filename.sh

  使用开头声明的方式执行(必须有可执行权限，脚本执行不影响当前路径)

- source ./filename.sh

  在当前进程中开始执行的(离开脚本后影响当前路径)

- . filename.sh

  同上面一种

#### 示例

**方法1：**

* cd /var
* ls

**方法2：**

cd /var/ ; ls

**方法3：**

* vim 1.sh

  编写脚本

* chmod u+x 1.sh

  可执行

* ./1.sh 

  使用系统默认的shell类型

### 内建和外部命令

- 内建命令不需要创建子进程
- 内建命令对当前Shell生效

## 常见用法

### 管道与重定向

管道：方便两条命令之间的通信。
重定向：可以让程序将标准输出输到文件中；还可以将文件作为输入。

- 管道与管道符
- 子进程与子shell
- 重定向符号

#### 管道符

管道和信号一样，也是进程通信的方式之一。匿名管道（管道符）是Shell编程经常用到的通信工具。

管道符是"|"，将前一个命令执行的结果传递给后面的命令

- ps | cat

- echo 123 | ps

- cat | ps -f

  通过管道符可以将两侧的进程建立子进程并连接起来（可以通过查看进程对应文件夹的详细文件描述符）

PS：由于管道符是以子进程的形式运行的，故管道符中类似命令`cd`、`pwd`是无法获取结果的

#### 重定向

一个进程默认会打开标准输入、标准输出、错误输出三个文件描述符。

基本语法：

* 输入重定向符号 “`<`”(右侧一般是一个文件)

  示例：**read var < /path/to/a/file**

* 输出重定向符号

  "`>`“、”`>>`“、“`2>`”、”`&>`"

  (`>`清空输出、`>>`追加输出、`2>`错误重定向、`&>`无论是正确还是错误全部输出到文件中)

  示例：echo 123 > /path/to/a/file

* 输入和输出重定向组合使用
  - cat > /path/to/a/file << EOF
  - I am $USER
  - EOF

示例：

* wc -l

  统计输入的行数，ctrl d

* wc -l < /etc/passwd

  输入重定向, 用右侧的输入代替原本的输入。统计行数。

* vim a.txt

  创建a.txt,内容。输入123

* read var2 < a.txt

  读取文件内容到变量

* echo $var2

  输入到窗口

  echo $var2 > b.txt

   输入到文件中

  echo $var2 >> b.txt

  追加数据到文件中

* nocmd 2> c.txt

  不存在命令，执行错误，保存前面命令的错误提示

* nocmd &> c.txt 

  不知道执行的命令是否是对的还是错误的都进行记录

* ls &> d.txt

* 输入和输出进行组合使用

  一般使用在生成配置文件中

  * vim 3.sh

    编写脚本

  ```
  # 文件内容
  #! /bin/bash
  cat > /root/a.sh << EOF
  echo "hello bash"
  EOF
  ```

  * bash 3.sh

    执行脚本

  * cat a.sh

    查看内容

### 变量

变量名的命名规则为：字母、数字、下划线，不以数字开头

为变量赋值的过程，称为变量替换

#### 赋值

* a=123 

  注意shell变量赋值时，等号的左侧和右侧不能出现空格

* let a=10+20

  使用let为变量赋值。最好不好进行计算，性能低

* l=ls

  将命令赋值给变量

* letc=$(ls -l /etc)

  将命令结果赋值给变量，使用$()或者``

  注意，变量值有空格等特殊字符可以包含在" "或’ '中

示例：

```
ls /root
cmd1=`ls /root`
cmd2=$(ls /root)
string1=hello bash
echo $string1 # 无结果，执行了bash
string1="hello bash"
echo $string1
```

#### 引用

* ${变量名}称为对变量的引用
* echo ${变量名}查看变量的值
* ${变量名}在`部分情况下`可以省略为 $变量名

示例

```
string1 = "hello bash"
# 两种方式一致
echo ${string}
echo $string
# 存在问题的情况
echo ${string}23 # 正确输出
echo $string23
```

#### 作用范围

* 变量的导出export用于将变量传递给子进程

* unset

  用于删除变量

示例：

* 子进程更改的数据不影响父进程

  a=1
  bash  #开启新的bash进程
  echo $a # 为空(子进程中无法读取父进程的信息)
  a=2
  exit
  echo $a # 1(子进程更改的数据不影响父进程)

* 子进程影响父进程的书写方式

  demo_var1="hello subshell"

  定义变量

  vim 4.sh

  编写脚本

  ```
  # 书写内容
  #！/bin/bash
  
  # test echo
  echo $demo_var1
  ```

  chmod u+x 4.sh

  bash 4.sh # 为空
  ./4.sh # 为空
  source 4.sh # hello subshell
  . 4.sh # hello subshell

* export demo_var1

  传递变量到达子进程

  export demo_var1
  bash 4.sh

  输出hello subshell
  export demo_var1="hello subshell"

  也可以定义时直接声明

* unset demo_var1

  删除变量

#### 环境变量

环境变量是每个Shell打开都可以获取到的变量。

* set和env命令

  显示环境变量(set比env更详细)

* $? $$ $0

   预定义变量

* $PATH

  系统查找命令的路径

* $PS1

  当前显示的终端格式

* $1 $2 … $n

  位置变量，获取输入参数的变量

案例演示：

* env | more 

  读取环境变量

* echo $USER 

  读取用户

* echo $UID 

* echo $PATH 

  $PATH为命令的搜索路径

* 统计目录所占用的磁盘空间

  * vim 5.sh

    ```
    # 文件内容
    echo "hello bash"
    du -sh # 统计目录所占用的磁盘空间
    ```

  * chmod u+x 5.sh 

  * ./5.sh或者5.sh

    未找到命令（原因在于当前路径不存在PATH）

  * pwd 

    当前为/root

  * PATH=$PATH:/root

    添加当前目录（当然定义的变量仅影响当前shell以及子shell

  * 5.sh

    直接执行（且在任意目录下都可以执行，很好理解）

* $PS1

  当前显示的终端格式（可通过修改定义，例如时间格式、ip、路径）

* 预定义变量

  * echo $? 

    上一条命令是否正确执行

  * echo $$

    当前进程的PID

  * echo $0 

    当前进程的名称（不同执行方式显示不同）

* 位置参数

  $1 $2 ... $9 ${10}

  注意第10个参数需要使用{}

  * vim 7.sh

  编辑一个脚本

  ```
  # 脚本内容
  #!/bin/bash
  
  # $1 $2 ... $9 ${10}
  pos1=$1
  pos2=${2-_}# 小技巧：如果$2有值就是$2，否则输出_
  ```

  * echo $pos1 

  * echo ${pos2}

  * chmod u+x 7.sh

  * ./7.sh -a -l

    输出

    ```
    -a
    -l
    ```

#### 配置文件

这里是环境变量配置文件。可以为自己添加写好的环境变量。

一共相关的文件有：

- /etc/profile
- /etc/profile.d/
- ~/.bash_profile
- ~/.bashrc
- /etc/bashrc

PS：/etc/目录下环境变量所有用户共享，~(用户家目录)是用户独有的。

演示：

* 在各个配置文件添加日志变量

  ```
  # etc/profile # 系统启动或者终端启动时的系统环境
  # 在头文件首部加上
  echo etc/profile
  # /etc/bashrc
  # 在头文件首部加上
  echo etc/bashrc
  # ~/.bashrc
  # 在头文件首部加上
  echo .bashrc
  # ~/.bash_profile
  # 在头文件首部加上
  echo .bash_profile
  ```

* su -root

  执行测试命令。切换账号

  输出：

  ```
  /etc/profile
  .bash_profile
  .bashrc
  /etc/bashrc
  ```

* export PATH=$PATH:/new/path

  写入上述任意文件即可为命令添加新的路径

  bash

  不会立即生效（2种方式：1.关掉在打开；2.使用source更新配置文件）

* su root

  加载配置文件不完整，不建议使用这种方式

  输出结果：

  ```
  .bashrc
  /etc/bashrc
  ```

### 数组

- 定义数组

  IPTS=( 10.0.0.1 10.0.0.2 10.0.0.3 )

- echo $IPTS

  输出10.0.0.1

- 显示数组的所有元素

  echo ${IPTS[@]

  10.0.0.1   10.0.0.2    10.0.0.3

- 显示数组元素个数

  echo ${#IPTS[@]}

  输出3

- 显示数组的第一个元素

  echo ${IPTS[0]}

  输出

  10.0.0.3

### 转义

特殊字符：一个字符不仅有字面意义，还是元意：

- \# 注释

- ; 分号

- \ 转义符号

  如单个字符前的转义符号：

  - \n \r \t 单个字母的转义(特殊功能)
  - $ " \ 单个非字母的转义

- "和’引号

### 引用

- 常见的引用符号
- " 双引号
- ’ 单引号
- ` 反引号

注：单引号的引用不会被解释，双引号的引用会被解释。

### 运算符

- 赋值运算符
- 算数运算符
- 数字常量
- 双圆括号

#### 赋值运算符

- = 赋值运算符，用于算数赋值和字符串赋值
- 使用unset取消为变量的赋值
- = 除了作为赋值运算符还可以作为测试操作符

#### 算数运算符

* 基本运算符

  `+ - * / ** %`

* 使用expr进行运算(expr只能支持整数，不能支持浮点数)

  `expr 4 + 5`

#### 数字常量

数字常量的使用方法

- let “变量名=变量值”
- 变量值使用0开头为八进制
- 变量值使用0x开头为十六进制

#### 双圆括号

双圆括号是let命令赋值的简化

- ((a=10))
- ((a++))
- echo $((10+20))

#### 演示

* expr 4 + 5

  输出*9*

* expr 4 + 5.2

  *非整数参数*

* num1=\`expr 4 + 5\`

  num1赋值一个命令

  echo $num1

  输出9

* (( a=4+5 ))

  echo $a

  输出9

* b=4+5

  echo b

  输出4+5，因为当作字符串

* (( a++ ))

  echo $a

  输出10

### 特殊字符大全

#### 引号

- ’ 完全引用
- " 不完全引用
- ` 执行命令

#### 括号

- () 、(( ))、$() 圆括号
  - 单独使用圆括号会产生一个子shell (xyz=123)
  - 数组初始化 IPS=( ip1 ip2 ip3 )
- [ ]、 [[ ]] 方括号
  - 单独使用方括号是测试(test)或数组元素功能
  - 两个方括号表示测试表达式
- < > 尖括号 重定向符号
- { } 花括号
  - 输出范围 echo {0…9}
  - 文件复制 cp /etc/passwd{,.back}

示例：

```
( abc=123 )
echo $abc # 输出为空(父shell看不见)
ipt=( ip1 ip2 ip3 )
echo $(( 10+20 )) # 30
cmd1=$(ls) # 获取命令的执行结果
echo $cmd1
# [] 进行测试
[ 5 -gt 4 ] 
echo $? # 0 真
[ 5 -gt 6 ]
echo $? # 1 假
[[ 5 > 4 ]]
echo $? # 0 真
# {}
echo {0..9} # 0 1 2 3 4 5 6 7 8 9
cp -v /etc/passwd{,.back} # 前面后缀是空的(,)，后面是.back
```



#### 运算和逻辑符号

- \+ - * / % 算数运算符

- \> < = 比较运算符

  - (( 5 > 4 )) 

  - echo $?

    输出0

  * (( 5 < 4 ))

  * echo $?

    输出1

  * (( 5 > 4 && 6 > 5 ))

  * echo $?*

    输出0

  * (( 5 > 4 || 6 < 5 ))

  * echo $?

    输出0

- && || ! 逻辑运算符

#### 转义符号

- \n 普通字符转义之后有不同的功能
- ’ 特殊字符转义之后，当做普通字符来使用

#### 其他符号

- \# 注释符

- ; 命令分隔符

  - case 语句的分隔符要转义 ;;

- : 空指令

- . 和source命令相同

- ~ 家目录

  * cd ~ 

    回到家目录

  * cd -

    回到上次切换的目录

  * cd ..

    回到当前目录的上级目录

- , 分隔目录

- \* 通配符

- ? 条件测试 或 通配符

  * ls ?.sh

    ?代表一个字符

- $ 取值符号

- | 管道符

- & 后台运行

- _ 空格

### 测试命令

#### echo $? 

输出上一个命令的执行结果

示例：

* vim 8.sh

  ```
  # 文件内容
  #!/bin/bash
  
  # exit demo
  ppwd # 默认没有该命令
  exit # 自定义错误返回值 exit 127
  # 文件内容
  ```

  PS：exit可以自定义错误码

* bash 8.sh

* echo $?

  非0返回值

#### test

test命令用户检查文件或者比较值

具体的可用的功能如下，test可以做以下测试：

- 文件测试
- 整数比较测试
- 字符串测试

- test测试语句可以简化为[ ] 符号
- [ ] 符号还有扩展写法[[ ]]支持&&、||、<、>

示例：

* man test

  查看test命令的用法

* test -f /etc/passwd

  -f 判断文件是否存在

  然后使用echo $? 

  输出0 文件存在

* [ -d /etc/ ]

  判断目录是否存在，换种方式

* [ -e /etc/ ]

  判断目录、文件、链接是否存在

* [ "abc" = "abc" ]

  echo $? 

  输出0

* [ "abc" = "Abc" ]

  echo $? 

  1

* [$UID = 0]

  echo $? 

  判断用户是否为根用户

* [$USER = root]

  echo $?

  判断是否为根用户

### 逻辑分支

#### if-then语句

基本语法

```
if [测试条件成立] 或命令返回值是否为0
then 执行相应命令
fi 结束 
```

* 判断用户是否为根用户分支

  ```
  if [ $UID = 0 ]
  then
  	echo "root user"
  fi
  ```

  分开多行，会自动添加；

* 执行命令成功，则做出动作

  ```
  if pwd
  then
  	echo "pwd running"
  fi
  ```

  zhi输出pwd结果，否则输出then之后的内容

#### if-then-else语句

```
if [ 测试条件成立 ]
then 执行相应命令
else 测试条件不成立，执行相应命令
fi 结束
```

* vim 9.sh

  ```
  # 文件内容
  #!/bin/bash
  
  # if else demo
  if	[ $USER=root ] ;then
  	echo "user root"
  	echo $UID
  else
  	echo "other user"
  	echo $UID
  fi
  ```

* chmod u+x 9.sh

  授权

* ./9.sh

  执行

#### if-elif-else语句

```
if [ 测试条件成立 ]
then 执行相应命令
elif [ 测试条件成立 ]
then 执行相应命令
else 测试条件不成立，执行相应命令
fi 结束 
```

案例：

* vim 10.sh

```
# 文件内容 
#!/bin/bash

# root user1 other
if [ $USER=root ];	then
	echo "root"
elif [ $USER=user1 ]; then
	echo "user"
else
	echo "other"
fi
# 文件内容
```

#### 嵌套if语句

```
if [ 测试条件成立 ]
then 执行相应命令
	if [ 测试条件成立 ]
	then 执行相应命令
	fi
fi 结束 
```

案例

* vim 11.sh

```
# 文件内容 
#!/bin/bash

# root -x
if [ $USER=root ];	then
	echo "please run"
	if [-x /tmp/10.sh ];	then # 判断当前脚本是否可执行
		/tmp/10.sh
	fi
else
	echo "switch user root"
fi
```

#### case分支

case语句和select语句可以构成分支

```
case "$变量" in
	"情况1"	）
		命令...	;;
	"情况2"	）
		命令...	;;
	*	) # *是通配符，匹配其他情况
		命令...	;;
	esac # esac是case的反写
```

* vim 12.sh 

  ```
  # 文件内容 
  #!/bin/bash
  
  # case demo
  case "$1" in
  	"start"|"START") # 有两种情况
  	echo $0 start ....
  	;;
  	
  	"stop")
  	echo $0 stop ....
  	;;
  	
  	"restart"|"reload")
  	echo $0 restart ....
  	;;
  	*)
  	echo Usage：$0 {restart|stop|restart|reload}
  	;;
  esac
  ```

* chmod u+x 12.sh 

  授权执行

### 循环语句

- 使用for循环遍历命令的执行结果
- 使用for循环遍历变量和文件的内容
- C语言风格的for命令
- while循环
- 死循环
- unitl循环
- break和continue
- 使用循环对命令行参数的处理

#### 标准for循环

```
for 参数 in 列表
do 执行的命令
done 封闭一个循环
```

可以使用反引号或$()方式执行命令，命令的结果当做列表进行处理

案例：

* echo {1..9} 

  打印出：1 2 3 4 5 6 7 8 9 

* mkdir {1..9}

  创建目录1到9

* for循环遍历1到9

  ```
  for i in {1..9}
  do
  	echo $i
  done
  ```

* 批量改名

  touch a.mp3 b.mp3 c.mp3

  创建3个文件

  ```
  for filename in `ls *.mp3`##filename从ls命令种获取
  do
  	mv $filename $(basename $filename .mp3).mp4 
  done
  ls *.mp4
  ```

  **PS：basename $filename获取当前文件名称，后面加上.mp3即可去除后缀名**

#### C语言风格的for

这种风格不常用

```
for((变量初始化；循环判断条件；变量变化))
do
	 循环执行的命令
done
```

案例

```
# 输出1-10
for (( i=1; i<=10; i++))
do
	echo $i
done
```

#### while循环

```
while test测试是否成立
do
	命令
done
```

案例1:

```
a=1
while [ $a -lt 10 ] 
do
	((a++));
	echo $a
done
```

案例2：

死循环

```
while : # :为空指令
do
	echo always;
done
```

#### until循环

until循环与while循环相反，循环测试为假时，执行循环，为真时循环停止

```
# 死循环
unitl [5 -lt 4]; # 条件为假一直执行
do
	echo always;
done
```

while的使用频率远高于until。

#### 嵌套与跳转

break、continue

- 循环和循环可以嵌套
- 循环中可以嵌套判断，反过来也可以嵌套
- 循环可以使用break和continue语句在循环中退出

嵌套示例：

```
for sc_name in /etc/profile.d/*.sh
do
	if [-x $sc_name] ; then
		. $sc_name # 执行文件
	fi
done
```

break示例

```
for num in {1..9}
do
	if [$num -eq 5]; then
		break 
	fi;
	echo $num
done
```

continue示例

```
for num in {1..9}
do
	if [$num -eq 5]; then
		continue 
	fi;
	echo $num
done
```

#### 处理命令行参数案例

- 命令行参数可以使用$1 $2 …${10}…$n进行读取
- $0 代表脚本名称
- $*和$@ 代表所有位置参数
- $# 代表位置参数的数量

该脚本的设计步骤为：

* vim 13.sh

* 脚本内容

  ```
  # 文本内容
  #!/bin/bash
  
  # help display help help
  
  for pos in $*
  do
  	if [ "$pos" = "help" ] ; then
  		echo $pos $pos
  	fi
  done
  
  # while的书写方式
  while [$# -gt 1] # 参数大于等于1个
  do
  	echo $# # 参数的数量
  	echo "do something"
  	if [ "$1"="help" ]; then
  		echo $1 $1
  	fi
  	shift # 参数左移，相当于删除了当前的参数
  done
  ```

* chmod u+x 13.sh

  授权

* 验证

  * bash 13.sh
  * bash 13.sh a b c help
  * bash 13.sh help

  输出结果：./13.sh a b c d

### 函数

#### 自定义函数

函数用于"包含"重复使用的命令集合

其语法为：

```
function fname(){ # 关键字function可以删除
	命令
} 
```

- local 变量名

  声明函数内部的变量（局部变量）

* $1 $2 $3 … $n

  函数的参数

**示例1：**

* 定义函数

  ```
  function cdls() {
  	cd /var
  	ls
  }
  ```

* cdls

  使用该函数

* unset cdls

  删除函数声明

**示例2：**

* 定义函数

  ```
  cdls() {
  	cd $1
  	ls
  }
  ```

* cdls /tmp

  遍历函数

**示例3**

判断进程是否存在

* vim 14.sh

  定义函数脚本

  ```
  # 文本内容
  #!/bin/bash
  
  # functions
  checkpid() {
  # proc目录下记录着存活的进程号对应的文件夹
  	local i
  	
  	for i in $*; do
  		[ -d "/proc/$i" ] && return 0 # 存在即返回0
  	done
  	
  	return 1
  }
  ```

* chmod u+x 14.sh

  授权

* source 14.sh

  执行脚本，即定义了sh中的函数

* checkpid 1

  测试函数

* echo $?

  0 即证明进程1是存在的

* checkpid 65533

  1 即证明进程65533是不存在的

#### 系统函数库

系统自建了函数库，可以在脚本中直接引用，etc/init.d/functions。也可以使用source函数脚本文件"导入"函数

* vim /etc/init.d/functions 

  查看文件

* source /etc/init.d/functions 

  导入当前shell运行环境

* echo_success

  绿色确定

### 死循环

可以使用nice和renice调整脚本优先级。

避免出现"不可控的"死循环

* 死循环导致cpu占用过高
* 死循环导致死机

示例：

* ulimit -a

  查看用户的资源限制

* su - user1

  切换用户

* : | :& 

  两个空指令，控指令

* func() {func | func &}

  递归调用

  & 在后台运行，递归调用

  调用func()

  创建大量子进程

* .(){.|.&} 

  上面的递归调用的简写形式

  fork炸弹

### 信号

捕获信号脚本的编写。我们可以实现如下功能：

- kill默认会发送15号信号给应用程序

- ctrl + c 发送2号信号给应用程序

- 9号信号不可阻塞 

  用于可在备份的时候不会被打断

案例如下：

* vim 15.sh

  ```
  # 文本内容
  #!/bin/bash
  
  # signal demo
  
  trap "echo sig 15" 15 # trap命令捕获信号
  trap "echo sig 2" 2 # 捕获2号信号可避免ctrl+c
  
  echo $$
  
  while : # 注意该程序十分消费系统资源
  do
  
  done
  # 文本内容
  ```

  这是一个死循环，执行该脚本后，会捕获15号和2号信号

* chmod u+x 15
  ./15.sh

* 另开一个终端进行测试

* kill -15 PID

  上述进行pid(不可杀死)
  kill -9 PID

  不可阻塞，可杀死

### 任务

#### 一次性计划任务

让计算机在指定的时间运行程序。需要注意的是计算任务没有终端，需要进行重定向文件输出

* at 18:31 

* echo hello > /tmp/hello.txt 

* ctrl+d

  进行提交 

* atq 

  列出当前用户的at任务列表

#### 周期性计划任务

配置方式：

- crontab -e

  * \* * * * * /usr/bin/date >> /tmp/date.txt

    任意的分钟 小时 日期 月份 星期

  * \* * * * 1,5 /usr/bin/date >> /tmp/date.txt

    星期一和星期五都会执行

  * \* * * * 1-5 /usr/bin/date >> /tmp/date.txt

    星期一到星期五都会执行

  * 15 18 7 7 1-5 /usr/bin/date >> /tmp/date.txt

    如果7月7日是星期一至星期五，则18点的15分钟会执行

  * 30 3 * * 1 /usr/bin/date >> /tmp/date.txt

    星期一的3点30分都会执行

- 查看现有的计划任务

  - crontab -l

  - ls /var/spool/cron/

    会建立文件夹给每个用户保存计划任务

- 配置格式：

  - 分钟 小时 日期 月份 星期 执行的命令
  - 注意命令的路径问题

#### 计划任务加锁 flock

如果计算机不能按照预期时间运行

- anacontab 延时计划任务
- flock 锁文件

案例：

* vim /etc/cron.d/0hourly

  延迟计划任务路径(每小时都被运行一次)

* vim /etc/anacontab

  延时计划任务

* vim a.sh

  ```
  # 文本内容
  #!/bin/bash
  
  # long time
  sleep 100000
  # 文本内容
  ```

* chmod u+x a.sh

* flock -xn "/tmp/f.lock" -c "/root/a.sh"

  排它锁，/tmp/f.lock为产生的锁文件

* flock -xn "/tmp/f.lock" -c "/root/a.sh"

  新打开一个终端执行命令

  无法运行(只能运行一个任务)

