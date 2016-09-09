---
title: OpenWRT上使用haproxy对shadowsocks做负载均衡
date: 2016-01-27 22:22:34
tags:
- openwrt
- router
categories:
- 学习笔记
- OpenWRT
---

上篇[OpenWRT科学上网的问题及其优化](/2016/01/27/optimize-shadowsocks-on-openwrt/)中，提到使用haproxy对shadowsocks的远程TCP连接做负载均衡，本文将介绍其实现过程。

交叉编译haproxy
--------------
虽然haproxy有预编译版本的ipk包，但是作者的小米路由mini所使用的Pandorabox的内核版本与该包依赖的内核版本不一致，于是作者选择交叉编译。因为haproxy的源码简单，外部依赖少，编译过程比较简单，甚至不需要OpenWRT的完整SDK，只需要toolchain即可。

博主这里提供一个在MT7620平台编译好的binary:
[MT7620](/downloads/haproxy)

1. 获取haproxy源码
  ```
  $ git clone https://github.com/haproxy/haproxy.git
  ```
2. 修改Makefile
  只需要在Makefile中将编译器改为OpenWRT的toolchain即可
  ```
  $ cd haproxy
  $ vim Makefile
  ```
  找到
  ```
  #### Toolchain options.
  # GCC is normally used both for compiling and linking.
  ```
  下面的
  ```
  CC = XXX
  LD = XXX
  ```
  这两行，将`CC`的值改为路由器toolchain的gcc的完整路径，`LD`的值改为`$(CC)`
  例如，博主修改后是这样的：
  ```
  CC = /home/chiontang/development/OpenWrt-SDK-ramips-for-linux-x86_64-gcc-4.8-linaro_uClibc-0.9.33.2/staging_dir/toolchain-mipsel_24kec+dsp_gcc-4.8-linaro_uClibc-0.9.33.2/bin/mipsel-openwrt-linux-uclibc-gcc
  LD = $(CC)
  ```
3. 编译
  ```
  $ make TARGET=generic
  ```
  如不出意外，当前目录下已经有了一个编译好的`haproxy`。可以在电脑上直接运行它测试一下，如果一切正常，会返回如下错误，因为使用了OpenWRT ARM平台的toolchain进行编译，x86环境下无法直接运行：
  ```
  $ ./haproxy
  zsh: exec format error: ./haproxy
  ```
  如果能够成功运行，说明刚才toolchain的设置没有成功，编译时直接使用了当前系统的编译器（如x86下的gcc）
4. 将编译好的haproxy复制到路由器下的`/root/haproxy`
  ```
  $ scp haproxy root@192.168.1.1:/root/haproxy
  ```

配置haproxy
----------
现在ssh连接上OpwnWRT路由器：
1. 建立haproxy配置文件：`/etc/haproxy.cfg`，内容如下：
  ```
  global
      log         127.0.0.1 local2

      chroot      /root
      pidfile     /tmp/haproxy.pid
      maxconn     4000
      user        root
      daemon

  defaults
      mode                    tcp    #TCP模式
      log                     global
      option                  httplog
      option                  dontlognull
      option http-server-close
      option forwardfor       except 127.0.0.0/8
      option                  redispatch
      retries                 2
      timeout http-request    10s
      timeout queue           1m
      timeout connect         2s     #上游TCP服务器连接等待时间                                      
      timeout client          1m
      timeout server          1m
      timeout http-keep-alive 10s
      timeout check           10s
      maxconn                 3000

  listen test1
      bind 0.0.0.0:8388       #haproxy监听端口
      mode tcp
      server s1 X.X.X.X:8388
      server s2 X.X.X.X:8389
      server s3 X.X.X.X:8390
      server s4 X.X.X.X:8391
  ```
  其中，`X.X.X.X`为ss服务器，8388~8391这四个端口都为ss服务器上运行ssserver的端口（如果觉得不够你可以再加几个），稍后将讲到服务器端的配置。haproxy监听的端口为8388。
2. 建立启动脚本：`/etc/init.d/haproxy`，内容如下：
  ```
  #!/bin/sh /etc/rc.common

  START=99  #启动优先级设置为99，即最后启动

  start() {
          /root/haproxy -f /etc/haproxy.cfg
  }

  stop() {
          killall haproxy
  }

  restart() {
          stop
          sleep 1
          start
  }
  ```
  然后使其生效：
  ```
  # /etc/init.d/haproxy enable
  # /etc/init.d/haproxy start
  ```

服务器配置
--------
作者使用supervisor作为ssserver的daemon，本例中，服务器同时监听8388~8391这四个端口。
博主服务器上的supervisor配置文件如下：
```
[supervisord]

[program:ssserver1]
command=/usr/bin/ssserver -m rc4-md5 -p 8388 -k [KEY]

[program:ssserver2]
command=/usr/bin/ssserver -m rc4-md5 -p 8389 -k [KEY]

[program:ssserver3]
command=/usr/bin/ssserver -m rc4-md5 -p 8390 -k [KEY]

[program:ssserver4]
command=/usr/bin/ssserver -m rc4-md5 -p 8391 -k [KEY]
```
将`[KEY]`替换为ss密码即可

ShadowSocks配置
--------------
1. 编辑`/etc/shadowsocks/ignore.list`（如果你设置的忽略列表为其他文件，则编辑那个文件）,在最后添加一行，输入ss服务器的IP。
2. 进入路由器配置Web，在luci配置页中，将ss服务器IP改为`127.0.0.1`，端口改为`8388`。
