## SUSE的一些问题

[TOC]

在使用**Peach**/**secFuzz**工具中的**SCTP发包器**时必须要在Linux环境下执行，在使用

![SUSE版本](images/suse版本.png)

版本的机器时，遇到了一些小问题，在这里Mark一下。

### mysctp.so报错

mysctp.so是工具在执行SCTP发包动作的时候会调用的文件，如果使用了SCTP发包器很可能产生与之相关的错误。例如：**Test Default error: mysctp.so**	

解决办法如下

第一种原因可能是环境变量的问题，可以尝试执行如下命令

> export PATH=/opt/peach:/opt/mono/bin:$PATH
>
> export LD_LIBRARY_PATH=/opt/peach111/:$LD_LIBRARY_PATH:
>
> export TERM=xterm 
>
> export TEMCAP=$INFORMIXDIR/etc/termcap

第二种原因可能是SUSE版本过高，需要重新编译mysctp.so。这篇[重新编译mysctp.so](http://3ms.huawei.com/hi/group/2030893/file_7771109.html)的文章中讲的很详细，照着做即可。

安装Peach

这篇[Peach安装指导（linux版）](http://3ms.huawei.com/hi/group/1502875/wiki_4163993.html)介绍的很详细，其中有一些小地方我来做一些补充。

#### 0x00 环境变量

我按照网上的教程在使用

```shell
PATH=/root/ayh/python/bin
export PATH
```

命令将python的环境变量永久写入**/etc/profile**文件时，每次重新登陆会话都会覆盖掉本来的环境变量，会报这种错误

![环境变量报错](images/没路径.png)

需要重新输入命令

```shell
 export PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin:/root/bin
```

将基础环境变量导入才可以正确执行如 ls 等基本命令。原因是使用赋值语句将默认的PATH覆盖掉了，避免麻烦的写法是在**/etc/profile文件写入

```shell
PATH=<你要加入的路径>:$PATH
```

#### 0x01 权限报错

解压Peach进入pits目录下之后正常的步骤应该是输入

```shell
../peach -h
```

来测试Peach是否能用，但是在我使用的过程中出现了权限的问题

![没权限](images/没权限.png)

回到上级目录输入给执行文件赋予执行权限

```shell
chmod +x peach
```

![赋予权限](images/赋予权限.png)

在linux中绿色表示可执行文件，灰色表示普通（其他）文本文件。看到赋权前peach只是一个普通文本权限，赋权后变成了可执行文件

此时可以正常使用peach

![正常使用peach](images/正常使用peach.png)

### 使用secFuzz

发现公司的secFuzz使用手册并不是很详细，在这里做一些补充

#### 0x00 环境变量

第一个问题和Peach中的第一个问题一样，python的环境变量问题。可以参考前面的进行设置。

#### 0x01 测试套存放位置

secFuzz的工具有些目录和Peach相似, example目录相当于Peach的pits目录

不同的是，Peach中将测试套中的状态模型（State.xml）和数据模型（Data.xml）放在 pits/_Common/Models/Net目录下；虽然在secFuzz工具中也有相似目录，但是状态模型和数据模型必须放在example目录下，测试套中的四个文件都放在example目录下

![测试套文件存放](images/测试套文件存放.png)

#### 0x02 目录报错

secFuzz工具和Peach工具的执行命令很相似，但还是有细微的不同，可能会导致报错，首先看一下Peach的执行命令，在pits目录下

```shell
../peach Net/测试套.xml
```

如果照搬到secFuzz中，在secFuzz根目录下

```shell
python secfuzz example/测试套.xml
```

这样执行是**错误的！**会报如下的错误

![找不到路径](images/找不到路径.png)

虽然路径没有问题，但还是找不到指定路径下的测试套文件。原因是在secFuzz执行命令传参时，路径参数是example根目录下的相对路径。secFuzz根目录下**正确的**执行语句如下

```shell
python secfuzz.py 测试套.xml
```

### 搭建VPN

申请的虚拟机默认安装了ppp和pptp，可以使用

```shell
rpm -qa | grep ppp
```

查看配置情况

#### 0x00 配置链接

首先配置VPN连接信息

```shell
vim /etc/ppp/chap-secrets
```

可以按照我的配置修改用用户名和密码

![vpn连接信息](images/vpn连接信息.png)

connectvpn使此条vpn连接名称，可以自己起名字；

#### 0x01 配置拨号

配置VPN拨号信息

```shell
vim /etc/ppp/peers/connectvpn
```

此文件是不存在的，需要自己创建

可以参考我的文件内容

```
pty "pptp 10.185.34.*** --nolaunchpppd"
lock
nobsdcomp
nodeflate
name a50001978
remotename connectvpn
ipparam vpn
#require-mppe-128
refuse-pap
refuse-chap
refuse-eap
refuse-mschap

```

第一行是申请到VPN服务器的IP地址；name后面是自己的用户名；remotename后面是自己自定义的连接名称

#### 0x02 拨号

VPN拨号

```shell
pppd call connectvpn &
```

连接成功会显示

> SR5S3:~ # Using interface ppp0 
>
> Connect: ppp0 <--> /dev/pts/1 
>
> local IP address 191.132.254.81 
>
> remote IP address 191.132.254.80
>
> Script /etc/ppp/ip-up finished (pid 20320), status = 0x0

点击回车，使用ifconfig命令可以查看多了一块虚拟网卡ppp0

![ppp0](images/ppp0.png)

#### 0x03 添加路由

添加路由

添加路由的意义是告诉系统从某个网段来的信息走ppp0网卡建立的VPN通道进行处理

```shell
route add -net <目标网段> netmask 255.0.0.0 dev ppp0
```

-net后面的IP地址就是要使用VPN访问的网段

#### 0x04 保持连接

因为一段时间不用之后VPN会自动关闭，可以参照我的操作修改一下

![保持链接](images/保持连接.png)

注释掉

> idle 600

或者也可以将600改成自定义的时间，单位为秒

#### 0x05 断开连接

执行以下命令关掉VPN

```shell
killall pppd
```

