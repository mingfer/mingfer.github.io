---
layout: post
title: "Linux 常用命令"
subtitle: "关于 Linux 命令的花式用法"
date: 2019-05-13 20:03:00
author: "mingfer"
header-img: "img/post/bg-1.jpg"
catalog: true
tags: 
    - "Linux"
---

> 技巧类的合集更多的是方便自己查阅所用，毕竟每次忘了都要去 google 或者 baidu 太影响效率了。



## Shell 脚本

### if 条件

**字符串判断**

```shell
str1 = str2      当两个字符串有相同内容、长度时为真
str1 != str2     当两个字符串不相等的时候为真
-n str           非空，当字符串长度大于 0 的时候为真
-z str           为空，当字符串长度等于 0 的时候为真
str              当字符串非空的时候为真
```

**数字大小判断**

```shell
num1 -eq num2     两数相等为真 equal
num1 -ne num2     两数不等为真 not equal
num1 -gt num2     num1 大于 num2 为真 great than
num1 -ge num2     num1 大于等于 num2 为真 great or equal
num1 -lt num2     num1 小于 num2 为真 less than
num1 -le num2     num1 小于等于 num2 为真 less or equal
```

**逻辑判断**

```shell
-a               逻辑与
-o               逻辑或
!                逻辑非
```

**文件判断**

```shell
-a FILE             如果 FILE 存在则为真。  
-b FILE             如果 FILE 存在且是一个块特殊文件则为真。  
-c FILE             如果 FILE 存在且是一个字特殊文件则为真。  
-d FILE             如果 FILE 存在且是一个目录则为真。  
-e FILE             如果 FILE 存在则为真。  
-f FILE             如果 FILE 存在且是一个普通文件则为真。  
-g FILE             如果 FILE 存在且已经设置了 SGID 则为真。 
-h FILE             如果 FILE 存在且是一个符号连接则为真。  
-k FILE             如果 FILE 存在且已经设置了粘制位则为真。  
-p FILE             如果 FILE 存在且是一个名字管道则为真。  
-r FILE             如果 FILE 存在且是可读的则为真。  
-s FILE             如果 FILE 存在且大小不为 0 则为真。  
-t FD               如果文件描述符 FD 打开且指向一个终端则为真。  
-u FILE             如果 FILE 存在且设置了 SUID (set user ID) 则为真。  
-w FILE             如果 FILE 存在且是可写的则为真。  
-x FILE             如果 FILE 存在且是可执行的则为真。  
-O FILE             如果 FILE 存在且属有效用户 ID 则为真。  
-G FILE             如果 FILE 存在且属有效用户组则为真。  
-L FILE             如果 FILE 存在且是一个符号连接则为真。  
-N FILE             如果 FILE 存在并且在最后一次打开后修改过则为真。  
-S FILE             如果 FILE 存在且是一个套接字则为真。  
FILE1 -nt FILE2     如果 FILE1 比 FILE2 要新（new than）, 或者 FILE2 存在且 FILE1 不存在则为真。  
FILE1 -ot FILE2     如果 FILE1 比 FILE2 要老（old than）, 或者 FILE2 存在且 FILE1 不存在则为真。  
FILE1 -ef FILE2     如果 FILE1 和 FILE2 指向相同的设备和节点号则为真。  
```

### 函数

shell 脚本的函数声明如下：

```shell
function func() {
	// do something
}
```

需要注意 shell 脚本的函数只能 `return` 数字，返回值使用 `$?` 获取。对于需要返回字符串类型的函数可以通过一个全局变量接收字符串值，使用 `$1`，`$2` 依次接收传入的参数。如：

```shell
#!/bin/bash

test=""
function func() {
	echo "the argument 1: $1"
	echo "the argument 2: $2"
	test="$1 $2"
	return 100
}

func "hello" "world"
echo "func return $?"
echo "the value of test: $test"
```

输入结果为：

```shell
the argument 1: hello
the argument 2: world
func return 100
the value of test: hello world
```

## 网络操作

### nc 建立和监听 TCP/UDP 连接

nc 是一个用于简单的建立和监听 TCP 和 UDP 连接的 Linux 命令行工具。

**常用参数**：

```shell
-k           保持监听，部分版本的 Linux 不含本参数
-l 			 开启监听
-p port      指定端口，部分版本不需要该参数
-v           打印详细的交互信息
-w           连接和读超时的时间间隔
-x           代理服务器地址
-z           用于端口扫描时候的参数，指发送零字节的数据
```

**查看指定主机的网络状况**

查看某个端口是否打开：

```shell
$ nc -vz -w 2 host port
```

扫描一个范围内的端口是否打开：

```shell
$ nc -vz -w 2 host port1-port2
```

**进行文件传输**

文件接收端，开启一个监听端口 `8080`，并将数据重定向到 `test.txt` 文件：

```shell
$ nc -lp 8080 > test.txt
# 部分版本不需要 -p
$ nc -l 8080 > test.txt
```

文件发送端，将文件 `test.txt` 输入到标准输入，发往 `8080` 端口：

```shell
$ nc localhost 8080 < test.txt
```

如果需要传输文件夹，可以将文件夹打包成 tar 包进行传输。

**进行聊天**

聊天的双方任一一方开启一个端口监听：

```shell
$ nc -lp 8080
# 部分版本不需要 -p
$ nc -l 8080
# 部分 Linux 版本支持 -k，客户端关闭之后，监听端不会跟随关闭
$ nc -lk 8080
```

另一端连接对应的主机端口：

```shell
$ nc localhost 8080
```

输入文本之后，按 `Enter` 键发送到对端，按 `Ctrl + C` 可以关闭连接。

### nmap 渗透测试工具

**网络扫描**

```shell
# 扫描单个主机
$ nmap 192.168.1.2

# 扫描多个主机
$ nmap 192.168.1.2 192.1.1.2

# 扫描一个范围内的主机
$ nmap 192.1.68.1.2-100

# 扫描整个子网
$ nmap 192.168.1.1/24

# 查看所有被扫描的主机列表
$ nmap -sL 192.168.1.1/24

# 扫描一个 IP 列表文件中指定的所有 IP
$ nmap -iL ip.file

# 扫描除某一主机的外的所有 IP
$ nmap 192.168.1.1/24 -exclude 192.168.1.1

# 扫描除过某一个文件中的ip外的子网主机
$ nmap 192.168.1.1/24 -exclude file exclude-ip.file

# 扫描特定主机上的特定端口
$ nmap -p80,21,23 192.168.1.1
```

**TCP SYN 扫描**

nmap 默认的扫描方式，也叫做半开放扫描。这种扫描方式 nmap 不需要通过完整的握手，就能获得远程主机的信息。nmap 发送 SYN 包到远程主机，但不会产生任何会话，因此不会在目标主机上产生任何日志记录。

```shell
$ nmap -sS 192.168.1.1
```

**TCP connect 扫描**

TCP connect 扫描要求调用系统的 connect() 函数完成三次握手，这种扫描方式使用与查找 TCP 或 UDP 端口。

```shell
$ nmap -sT 192.168.1.1
```

**UDP 扫描**

这种扫描方式主要用于寻找开放的 UDP 端口，UDP 扫描通过发送 UDP 数据包到目标主机，并等待响应。如果返回 ICMP 不可达的错误消息，说明端口是关闭的；如果得到正确的适当的回应，说明端口是开放的。

```shell
$ nmap -sU 192.168.1.1
```

**FIN 扫描**

当目标主机存在 IDS 或 IPS 系统的时候，防火墙会阻止掉 SYN 数据包。FIN 扫描发送一个 FIN 标志的数据包，不需要完成 TCP 握手，所以也不会在目标主机产生任何日志。

```shell
$ nmap -sF 192.168.1.1
```

**PING 扫描**

PING 扫描只能用于找出主机是否是存在在网络中的，不能用于发现端口是否开放。PING 扫描需要ROOT权限，如果用户没有 ROOT 权限，PING 扫描将会使用 connect() 调用。

```shell
$ nmap -sP 192.168.1.1
```

**IDLE 扫描**

IDLE 扫描通过另一个目标主机向目标主机发送数据包，这是一种匿名扫描的技术。

```shell
$ nmap -sL 192.168.1.6 192.168.1.1
```

**软件版本扫描**

软件版本扫描用来扫描目标主机和端口上运行的软件的版本，它通过从开放的端口获取信息来判断软件的版本。使用该扫描之前需要先用 TCP SYN 扫描目标主机开放了哪些端口。

```shell
$ nmap -sV 192.168.10.1
```

## 资源监控

### pmap 查看进程的内存映像信息

使用设备格式查看具体线程（替换 $pid 为具体的进程号）的内存信息：

```shell
$ pmap -d $pid
```

扫描出来的结果的最后一行说明如下：

- **mapped** 表示该进程映射的虚拟地址空间大小，也就是该进程预先分配的虚拟内存大小
- **writeable/private**  表示进程所占用的私有地址空间大小，也就是该进程实际使用的内存大小     
- **shared** 表示进程和其他进程共享的内存大小

循环扫描内存信息：

```shell
$ while true; do pmap -d $pid | tail -1; sleep 2; done
```

## 文件操作

### readlink 打印软连接的目标路径

readlink 用于打印软连接文件的目标文件路径。

使用 `readlink filename` 获取到软连接文件的指向文件的路径。一些系统下目标文件可能是以相对路径存在的，可以使用 `readlink -f filename` 获取目标文件的绝对路径。**注意**：在 macOS 中，readlink 只支持 `-n` 选项，即不在输出内容末尾加换行符

### 去除文件特殊字符

**使用 VI 或 VIM 去除**

- 去除空格：`:%s/ //g`
- 去除换行：`:%s/\n//g`
- 去除制表符：`:%s/\t//g`

这里其实是做了正则替换，`: %s/str1/str2/g` 是指在全文中将 `str2` 替换成 `str1`。

**使用 tr 命令替换**

tr 是将一个字符转换为另一个字符的意思，我们可以通过 `cat test.txt | tr '\n' ' ' ` 的方式将 test.txt 文件内容中的 `\n` 替换成空格之后再打印到标准输出中。

### dos 和 unix 文本转换

由于 Windows 和 Linux 下文本文件的换行符有差异，在 Windows 下编写的 shell 脚本因为包含非法字符 `\r` 往往在 Linux 上无法执行。此时我们需要通过对文件进行转码：

```shell
$ dos2unix file1 file2
```

类似的命令还有：**unix2dos**，**unix2mac**，**mac2unix**

