## shell

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