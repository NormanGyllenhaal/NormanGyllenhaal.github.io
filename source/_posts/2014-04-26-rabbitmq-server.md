---
layout: post
title: "CentOS6.4安装rabbitmq-server"
date: 2014-04-26 10:55:13 +0800
comments: true
categories: rabbitmq
---

### 在 CentOS 6.4上安装python
自己手动安装python2.7.5，不要动系统上面其他的版本

**1,先安装GCC，用如下命令yum install gcc gcc-c++**
```
yum install zlib
yum install zlib-devel
```
**2,下载 [python-2.7.5.tar.gz][] 文件，修改文件权限**
```
chmode +x python-7.5.tar.gz
```
**3,解压tar文件**
```
tar -xzvf python-2.7.5.tar.gz
```
**4,编辑Setup.dist**
```
cd python-2.7.5
vim Python-2.7.5/Modules/Setup.dist
```
找到<!--more-->

    #SSL=/usr/local/ssl
    #_ssl _ssl.c \
    #       -DUSE_SSL -I$(SSL)/include -I$(SSL)/include/openssl \
    #       -L$(SSL)/lib -lssl -lcrypto
    ......
    #zlib zlibmodule.c -I$(prefix)/include -L$(exec_prefix)/lib -lz

把注释去掉后开始执行安装
```
./configure --prefix=/usr/local/python27 --with-zlib=/usr/include
make &amp;&amp; make install
```
**5、建立软连接，使系统默认的python指向python27**
```
mv /usr/bin/python /usr/bin/python2.6.6.old
ln -s /usr/local/python27/bin/python2.7 /usr/bin/python
```
已经安装完成python的安装或升级的全部操作了，我们再来看一下现在的python的版本：
```
python -V
Python 2.7.5
```
虽然现在python已经安装完成，但是使用yum命令会有问题——yum不能正常工作。

这是因为yum默认使用的python版本是2.6.6，到哪是现在的python版本是2.7.5，
故会出现上述问题，只需要该一下yum的默认python配置版本就行了：
```
vi /usr/bin/yum
```
将文件头部的`#!/usr/bin/python` 改为`#!/usr/bin/python2.6`

### 在 CentOS 6.4上安装Erlang
在本节中，我们将来学习如何在CentOS 6.4上安装erlang，具体的Erlang版本是R16B02。

在安装之前，需要先要安装一些其他的软件，否则在安装中间会出现一些由于没有其依赖的软件模块而失败。

**1、首先要先安装GCC GCC-C++ Openssl等以来模块：**
```
yum -y install make gcc gcc-c++ kernel-devel m4 ncurses-devel openssl-devel
```
**2、再安装ncurses模块**
```
yum -y install ncurses-devel
yum install ncurses-devel
```
**3、下载Erang源代码文件文件，并对其付权限和解压文件：**
```
wget http://www.erlang.org/download/otp_src_R16B02.tar.gz
chmod +x otp_src_R16B02.tar.gz
tar -xzvf otp_src_R16B02.tar.gz
mv otp_src_R16B02 erlang_R16B #重命名解压厚的文件
```
**4、下面是安装erlang的重头戏，依次执行以下操作：**
```
cd erlang_R16B/
#不用java编译，故去掉java避免错误
./configure --prefix=/usr/local/erlang --with-ssl --enable-threads --enable-smp-support --enable-kernel-poll --enable-hipe --without-javac
make &amp;&amp; make install #编译后安装
```
**5、配置erlang环境：**
```
vi /etc/profile
ERL_HOME=/usr/local/erlang
export PATH=$PATH:$ERL_HOME/bin
```
好了，现在erlang的已经配置好了，现在我们来测试一下是否安装成功,在控制台输入命令erl，
如果在erlang shell里出现下图所示就说明安装成功了：
此处省略截图了…

### 在CentOS上安装rabbitmq-server-3.1.5
在本节中我们来看一下如何在CentOS上安装RabbitMQ。
我们使用的rabbitmq的版本是rabbitmq-server-3.1.5.tar.gz，CentOS的版本是CentOS 6.4。

在安装rabbitmq之前需要先安装python和erlang，
这两部分的安装过程请参看在CentOS 6.4上安装python和在 CentOS 6.4上安装Erlang，这里不再赘述。

安装rabbitmq的具体步骤如下：

**1、下载rabbitmq-server-3.1.5.tar.gz文件，并解压之：**
```
cd /usr/local
wget http://www.rabbitmq.com/releases/rabbitmq-server/v3.1.5/rabbitmq-server-3.1.5.tar.gz
chmod +x rabbitmq-server-3.1.5.tar.gz
tar -xzvf rabbitmq-server-3.1.5.tar.gz
```
**2、在编译rabbitmq源码之前先要安装其需要依赖包：**
```
yum -y install xmlto
```
否则会编译不通过：
```
/bin/sh: line 1: xmlto: command not found
```
**3、开始编译源代码：**
```
cd rabbitmq-server-3.1.5
make
#将rabbitmq编译到/opt/mq/rabbitmq目录
make install TARGET_DIR=/opt/mq/rabbitmq SBIN_DIR=/opt/mq/rabbitmq/sbin MAN_DIR=/opt/mq/rabbitmq/man
```
**4、安装web插件管理界面**
```
cd /opt/mq/rabbitmq/sbin
mkdir /etc/rabbitmq/
rabbitmq-plugins enable rabbitmq_management
```
**5、好了，到这里rabbitmq已经配置好了，可以启动了：**
```
./rabbitmq-server start &
```
我运行的时候报错了，ERROR: epmd error for host “springzoo”: timeout (timed out)

更改下/etc/hosts:

    127.0.0.1   localhost springzoo
    ::1         localhost springzoo

接下来我们查看下端口
```
ps aux | grep rabbitmq #查看端口，默认就是5672
netstat -tnlp | grep 5672
```
应该是下面的结果

    tcp        0      0 0.0.0.0:15672               0.0.0.0:*                   LISTEN      30435/beam.smp
    tcp        0      0 0.0.0.0:55672               0.0.0.0:*                   LISTEN      30435/beam.smp
    tcp        0      0 :::5672                     :::*                        LISTEN      30435/beam.smp

如果看到下面的信息就表明已经启动成功了！
省略截图….

最好我们就可以在浏览器上输入http://127.0.0.1:15672/登录管理界面了

使用登录的名户名和密码默认都算guest，登录后的页面如下：

截图再次省略…


[python-2.7.5.tar.gz]: https://www.python.org/ftp/python/2.7.5/Python-2.7.5.tgz
