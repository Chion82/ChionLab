---
title: OpenWRT自动科学上网的问题及优化
date: 2016-01-27 01:45:48
tags:
- openwrt
- router
categories:
- 干货
- OpenWRT
---

在[OpenWRT路由器配置ShadowSocks+ChinaDNS](/2016/01/23/openwrt-bypass-gfw-solution/)虽然使用起来十分优雅和方便，但是因为ISP质量问题，可能随之会带来很多稳定性问题，包括但不限以下作者遇到过的问题：
* ISP封杀境外UDP流量，导致SS的UDP隧道非常不稳定，ChinaDNS查询失败（比如广州电信部分地区）
* 链路拥堵时，到ss服务器的多并发TCP连接无法建立成功，新建立的连接容易卡在`SYN_SENT`，最终导致超时
* ChinaDNS长时间运行后可能出现的解析不正常的情况，表现为始终返回国外DNS结果，导致国内网站访问缓慢
针对以上问题，作者琢磨出一些优化方法。

优化UDP稳定性
-----------
由于ISP封杀境外UDP包，那么UDP只可以走境内。那么可以将一台境内服务器作为UDP跳板。TCP包直接发到境外的ss服务器，UDP包则经过境内的跳板服务器中转到ss服务器。跳板服务器可以选择阿里云、腾讯云等，带宽不需要太大，1M都完全够了，主要是用来中转DNS查询包。作者用的是腾讯云的1元／月的学生优惠。

一、在UDP跳板服务器上进行以下配置以实现中转。
1. 开启内核的ip_forward
  ``` shell
  # vim /etc/sysctl.conf
  ```
  将`net.ipv4.ip_forward=0`修改为`net.ipv4.ip_forward=1`。如果文件为空，则添加一行`net.ipv4.ip_forward=1`即可。
  运行这条命令即时生效：
  ```
  # sysctl -p
  ```
2. 用iptables实现TCP+UDP中转
  ``` shell
  # iptables -t nat -A PREROUTING -p tcp --dport [PORT] -j DNAT --to-destination X.X.X.X:X
  # iptables -t nat -A PREROUTING -p ucp --dport [PORT] -j DNAT --to-destination X.X.X.X:X
  # service iptables save #永久保存修改
  ```
  其中，`X.X.X.X:X`为ss服务器IP和端口，`[PORT]`为跳板服务器上用于中转的监听端口（为了方便，可以和ss服务器的端口相同）。

二、修改OpenWRT上的配置
  1. ssh连接上路由器，建立一个跳板服务器的ss配置:
  ``` shell
  # cp /var/etc/shadowsocks.json /root/shadowsocks_udp.json #复制当前的ss配置文件
  # vim /root/shadowsocks_udp.json
  ```
  然后修改`/root/shadowsocks_udp.json`，将远程服务器IP改为跳板服务器的IP，远程端口改为跳板服务器的监听端口（即刚才的`[PORT]`）
  2. 然后需要修改shadowsocks的启动脚本，让ss隧道使用跳板服务器的配置文件。
  ```
  # vim /etc/init.d/shadowsocks
  ```
  在`CONFIG_FILE=/var/etc/shadowsocks.json`下增加一行：
  ```
  UDP_CONFIG_FILE=/root/shadowsocks_udp.json
  ```
  找到：
  ```
  /usr/bin/ss-tunnel \       
                -c $CONFIG_FILE \        
                -l $tunnel_port \          
                -L $tunnel_forward \
                -f /var/run/ss-tunnel.pid \
  ```
  将其修改为：
  ```
  /usr/bin/ss-tunnel \       
                -c $UDP_CONFIG_FILE \        
                -l $tunnel_port \          
                -L $tunnel_forward \
                -f /var/run/ss-tunnel.pid \
  ```
  保存后重启shadowsocks
  ```
  # /etc/init.d/shadowsocks restart
  ```

优化TCP稳定性
-----------
1. 最好的方法是选择链路质量好的服务器作为ss服务器，比如香港的vps，将不容易出现多并发TCP连接无法建立的情况。
2. 如果不能选择更换ss服务器，可在ss服务器上同时监听多个端口，同时在路由器上使用haproxy进行负载均衡，把ss服务器的多个端口加入到upstream列表中，由于haproxy超时自动更换upstream的特性，可大大降低连接失败的概率。详细操作方法作者将在下篇博文中介绍。

ChinaDNS偶然解析不正常的解决
------------------------
尚不明确为何ChinaDNS长时间运行后偶然地可能会始终返回国外DNS结果（重启ChinaDNS即恢复），作者之前尝试[修改了一下ChinaDNS](https://github.com/Chion82/ChinaDNS)，将判断某个IP是否为国内IP的逻辑修改为借助ipset来判断，但是作者没有测试过其稳定性。
本文介绍的解决方法很无脑，即定时重启ChinaDNS。使用cron，在每天的零时重启一次ChinaDNS。经作者测试，这招确实有效。
在路由器上运行：
```
# crontab -e
```
添加一行：
```
0 0 * * * /etc/init.d/chinadns restart
```
保存并重启路由器生效。
