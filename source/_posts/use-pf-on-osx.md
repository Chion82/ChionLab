---
title: OSX上pf的简单配置笔记
date: 2016-02-01 01:16:55
tags:
- OSX
categories:
- 学习笔记
- OSX
---

水果的OSX上没有iptables，在10.10以后以pf取代ipfw。相比于iptables，pf一般使用配置文件保存防火墙规则，语法规范上更严谨，但是配置也更复杂、规则冗长。本文记录pf的简单配置方法。

`cat /etc/pf.conf`，可看到以下已有内容：（忽略注释部分）
```
scrub-anchor "com.apple/*"
nat-anchor "com.apple/*"
rdr-anchor "com.apple/*"
dummynet-anchor "com.apple/*"
anchor "com.apple/*"
load anchor "com.apple" from "/etc/pf.anchors/com.apple"
```
`anchor`可理解为一组规则的集合。默认情况下，这里的几行anchor都是苹果留的place holder，实际上没有active的规则。
`/etc/pf.conf`在以后的OSX更新中可能会被覆盖，最好可以另外建立一个自定义的`pf.conf`。
配置文件必须按照`Macros`, `Tables`, `Options`, `Traffic Normalization`, `Queueing`, `Translation`, `Packet Filtering`的顺序。
更详细的说明参考[pf.conf man page](https://developer.apple.com/library/mac/documentation/Darwin/Reference/ManPages/man5/pf.conf.5.html)

1. 添加一个`anchor`。修改`/etc/pf.conf`如下：
  ```
  scrub-anchor "com.apple/*"
  nat-anchor "com.apple/*"
  nat-anchor "custom"
  rdr-anchor "com.apple/*"
  rdr-anchor "custom"
  dummynet-anchor "com.apple/*"
  anchor "com.apple/*"
  anchor "custom"
  load anchor "com.apple" from "/etc/pf.anchors/com.apple"
  load anchor "custom" from "/etc/pf.anchors/custom"
  ```
2. 建立`anchor`规则文件`/etc/pf.anchors/custom`，内容为具体规则。
  常用的规则：
  * 屏蔽IP入站TCP连接并记录：
  ```
  block in log proto tcp from 192.168.1.136 to any
  ```
  * 转发入站TCP连接到另一本地端口：
  ```
  rdr inet proto tcp from any to any port 8081 -> 127.0.0.1 port 80
  ```
  经测试，rdr无法转发到另一台外部主机上（man page的示例，只可以转发到internal network），内核开启`net.inet.ip.forwarding=1`也无效。如需转发到另一个外网IP，需要配合[mitmproxy的透明代理](http://mitmproxy.org/doc/transparent/osx.html)
  * NAT，路由vlan12接口上(192.168.168.0/24)的出口包，经由非vlan12的接口转换到外部地址(204.92.77.111)，并允许vlan12之间的互相访问:
  ```
  nat on ! vlan12 from 192.168.168.0/24 to any -> 204.92.77.111
  ```
3. 使配置文件生效
  ```
  pfctl -evf /etc/pf.conf
  ```
