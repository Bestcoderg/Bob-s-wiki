# Linux 命令

[linux命令大全](Linux%20%E5%91%BD%E4%BB%A4%20a88759794f3a4447b311e590a4a97ce7/linux%E5%91%BD%E4%BB%A4%E5%A4%A7%E5%85%A8%20f8b2b5737b1c4257918e8815ea48ccb4.csv)

[Bash快捷键](Linux%20%E5%91%BD%E4%BB%A4%20a88759794f3a4447b311e590a4a97ce7/Bash%E5%BF%AB%E6%8D%B7%E9%94%AE%20c97fa83c2f57463d914d264ba932a247.csv)

1、**xargs**

xargs 是给命令传递参数的一个过滤器，也是组合多个命令的一个工具。

xargs 可以将管道或标准输入（stdin）数据转换成命令行参数，也能够从文件的输出中读取数据。

xargs 也可以将单行或多行文本输入转换为其他格式，例如多行变单行，单行变多行。

xargs 默认的命令是 echo，这意味着通过管道传递给 xargs 的输入将会包含换行和空白，不过通过 xargs 的处理，换行和空白将被空格取代。

xargs 是一个强有力的命令，它能够捕获一个命令的输出，然后传递给另外一个命令。

之所以能用到这个命令，关键是由于很多命令不支持|管道来传递参数，而日常工作中有有这个必要，所以就有了 xargs 命令，例如：

xargs -n <num> 后面加次数，表示命令在执行的时候一次用的argument的个数，默认是用所有的。

**xargs -i  cat {}** 

i 或者是-I，这得看linux支持了，将xargs的每项名称，一般是一行一行赋值给 {}，可以用 {} 代替

2、sed

Linux sed 命令是利用脚本来处理文本文件。

sed 可依照脚本的指令来处理、编辑文本文件。

Sed 主要用来自动编辑一个或多个文件、简化对文件的反复操作、编写转换程序等。

3、netstat

从整体上看，netstat的输出结果可以分为两个部分：

>>一个是Active Internet connections，称为有源TCP连接，其中"Recv-Q"和"Send-Q"指%0A的是接收队列和发送队列。这些数字一般都应该是0。如果不是则表示软件包正在队列中堆积。这种情况只能在非常少的情况见到。

另一个是Active UNIX domain sockets，称为有源Unix域套接口(和网络套接字一样，但是只能用于本机通信，性能可以提高一倍)。Proto显示连接使用的协议,RefCnt表示连接到本套接口上的进程号,Types显示套接口的类型,State显示套接口当前的状态,Path表示连接到套接口的其它进程使用的路径名。

4、 |

管道，把第一条命令的输出作为第二条的输入

5、sync

将内存中没有写入硬盘的数据写入硬盘，在reboot、shutdown前会先执行这条命令

6、eval&exec

eval

- bashshell中内建的一个命令，命令后面所跟的内容都认为是参数，但是会两次扫描其参数：第一次扫描会将参数中的变量进行替换；第二次扫描会将后面的参数当作一个shell中的命令组合来执行命令。
- 实际使用中，可以将任意组合的命令赋值给一个变量，然后在需要的位置通过 eval $variable 来执行这个命令。
- 常见用法：
    1. 直接组合命令 ： eval ls -al
    2. 替换变量
    3. 可以执行任何值为命令组合的变量
    4. 变量替换赋值

exec

- 也是shell内建的一个命令。类似 eval、source，不同的是exec执行后面的命令会替换当前shell进程，而前两者不会。

- 常见用法：
    1. 用于分离执行脚本，并退出子脚本的shell进程
    2. 用于设置描述符重定向输入文件内容
    3. 用于设置描述符重定向输出内容至文件

7、&&和 ||

- command1 && command2 [&& command3 ...]
    - 左边的命令返回真后，右边的命令才能够被执行
    - 只要有一个命令返回假，后面的命令就不会被执行
- command1 || command2
    - 只有左边的命令返回假（$? ==1），右边的命令才能被执行，即实现短路逻辑或操作。
    - 只要有一个命令返回真，后面的命令就不会被执行

8、查看端口

netstat -tnlp | grep 30000

9、创建user

useradd -G <groupname> <name>

passwd <name>

vim /etc/sudoers

10、获取内存占用详细信息：

cat /proc/pid/statm

ssh-keygen -t rsa -C ["your_email@example.com](mailto:%22your_email@example.com)"

11、

for x in `ls game-943-*-2021-08-17-*.log`; do echo $x; cat $x | grep Reconnect; donegame-943-1-2021-08-17-00.log