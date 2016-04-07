---
title: 网易云音乐反向代理163-music-unlock更新记录
date: 2016-04-06 23:40:45
tags:
- hack
- nginx
- python
- maintenance
categories:
- 开发笔记
- Python
---

这几天，博主去年开发的[163-music-unlock](https://github.com/Chion82/163-music-unlock)（[开发笔记](https://blog.chionlab.moe/2016/01/21/netease-music-app-reversed-proxy/)）用户数急剧增加，GitHub repo获得了160多个star（甚至上了GitHub的Trending列表，作者还是有点小激动的），服务器每日请求数达到10万、600IP。随着用户增多，项目也收到了不少Issue反馈，总结如下：
* 歌曲加载速度慢（特别是版权歌曲）
* 无法下载付费歌曲
* 无法加载其他用户的个人资料和歌单信息
* 部分付费歌曲无法播放

提高反代性能
----------
检查nginx access log，发现其中有多个请求状态为499。499为nginx特有的状态码，含义是客户端未等待服务器回应而主动关闭连接。
经过测试，发现网易云音乐客户端调用API接口时都有超时重试机制，并且超时时间比较短，大概在3～5秒左右，若服务器未在该时间内响应，客户端会直接关闭连接而重试，导致服务器上有大量499请求记录。
首要目标是提高服务器反代性能。其实在这之前，反代服务器基本上只有博主在使用，性能问题不明显，歌曲很快就可以加载出来了，而最近用户数量上升后，明显感觉到歌曲加载速度非常慢。
由于反代服务器架设在SLHK节点的VPS上，经过测试，发现瓶颈在SLHK到`music.163.com`的链路上。由于nginx的优化，nginx直接到网易的反代性能还能接受。但是python脚本调用网易云音乐API时速度很慢，甚至很多时候会直接导致gunicorn主动关闭超时请求。
python脚本主要处理歌曲播放地址API和歌曲下载地址API这两种请求，其他请求都直接由nginx直接反代到网易了。
由于网易云音乐主服务器在国内，使用国内云服务器当然是最佳的。博主测试发现，反代python脚本运行在腾讯云或阿里云上时，调用网易云音乐API速度非常快。
但是如果直接将反代服务器架设在腾讯云或阿里云上，有个问题：客户端是使用`music.163.com`这个域名访问80端口web服务的，而国内云平台会拦截未备案域名的web请求。但是，直接通过IP访问web服务（即HTTP头中，Host的值为服务器IP）时，不会被拦截。
SLHK VPS到腾讯云或阿里云的链路情况也很好。因此，我将python脚本放在腾讯云上运行，而反代服务器依然使用SLHK VPS，只不过在SLHK VPS上，将歌曲播放地址API和歌曲下载地址API的URL反代到腾讯云上（直接使用腾讯云的IP），其他URL请求维持原样。
将python脚本部署在腾讯云上，并在SLHK VPS上修改nginx配置如下：
```
upstream balanced_backend {
        server music.163.com;
}

server {
        listen 80;
        server_name music.163.com;
        resolver 114.114.114.114;

        #一般请求直接反代到music.163.com
        location / {
               proxy_pass http://balanced_backend;
               proxy_next_upstream     error timeout invalid_header http_500;
               proxy_connect_timeout   6s;
               proxy_send_timeout       6s;
               proxy_read_timeout       6s;
               proxy_set_header Host $host;
               proxy_set_header X-Real-IP $remote_addr;
               proxy_set_header Accept-Encoding "";
               subs_filter_types *;
               subs_filter '"st":-.+?,' '"st":0,' ir;
               subs_filter '"pl":0' '"pl":320000';
               subs_filter '"dl":0' '"dl":320000';
               subs_filter '"sp":0' '"sp":7';
               subs_filter '"cp":0' '"cp":1';
               subs_filter '"subp":0' '"subp":1';
               subs_filter '"fl":0' '"fl":320000';
               subs_filter '"fee":.+?,' '"fee":0,' ir;
      	       subs_filter '"abroad":1,' '';
        }

        #歌曲播放地址API
        location /eapi/song/enhance/player/url {
               proxy_pass http://<腾讯云IP>:5001;
        }

        #歌曲下载地址API
        location /eapi/song/enhance/download/url {
               proxy_pass http://<腾讯云IP>:5001;
        }

}
```
为了进一步提高python反代脚本的性能，增加gunicorn的进程数到16。当某个请求使得某个服务进程（或者说是worker）调用网易API而发生io阻塞时，整个进程都会被阻塞而无法接手下一个请求，因此理论上，这种网络io瓶颈型的服务，进程数越多越好。但是单个gunicorn进程内存占用大，经测试，在博主的1G内存腾讯云上，开50个gunicorn进程已接近极限，而性能甚至不如16个进程。
通过gunicorn运行反代服务的启动脚本如下：
```
#!/bin/sh
cd /root/163-music-unlock/server    #替换为本项目下的server目录
/usr/local/bin/gunicorn -w 16 runapi:app -b 0.0.0.0:5001 --access-logfile /var/log/gunicorn.access.log --error-logfile /var/log/gunicorn.error.log --log-file /var/log/gunicorn.log
```

增加下载地址API反代
----------------
之前的版本中，python脚本只处理歌曲在线播放地址的API，所以下架歌曲或付费歌曲只能试听，无法下载。经过抓包发现，在线播放API和下载地址API只有细微的差异：
```JavaScript
//在线播放API，服务器返回格式：
{
  "code" : 200,
  "data" : [
    {
      "id" : 123456,
      "url": "http://m1.music.net/music.mp3",
      "br" : 64000,
      "size": 12345,
      "md5" : "11111111111111111111111111111111",
      "code": 200,
      "expi": 1200,
      "type": "mp3",
      "gain": 0,
      "fee": 0,
      "canExtend": False
    }
  ]
}
//下载地址API，服务器返回格式：
{
  "code" : 200,
  "data" : {
    "id" : 123456,
    "url": "http://m1.music.net/music.mp3",
    "br" : 64000,
    "size": 12345,
    "md5" : "11111111111111111111111111111111",
    "code": 200,
    "expi": 1200,
    "type": "mp3",
    "gain": 0,
    "fee": 0,
    "canExtend": False
  }
}
```
python脚本只需要做稍微的调整就可以同时兼容下载地址API了。
经过测试，IOS和OSX客户端都可以下载付费音乐了。但是，安卓客户端在下载付费音乐时，虽然有下载速度，但是最后会报网络错误下载失败，怀疑是因为安卓客户端会校验下载文件的md5，而python反代脚本获取到的付费音乐信息不含文件md5（python脚本所调用的网易云音乐API不返回文件md5信息），直接给客户端返回`"md5":null`所致。

解决https反代的问题
-----------------
Issue中反映的无法查看其他用户资料和歌单的问题，经过抓包发现，是因为这部分API是https的，而服务器上只有http反代。博主尝试使用自签证书在nginx上实现https反代，但是IOS客户端不接受自签证书。虽然客户端可以通过PAC配置文件或者iptables，实现只将http请求转发到反代服务器，而https请求直接到网易云音乐服务器，但是在IOS或安卓上，这样的配置对于用户而言是十分繁琐的，因此还是在服务器上实现https反代。
通过在TCP层的转发，是可以实现免SSL证书反代https请求的。因此，在反代服务器上通过iptables实现443端口转发：
```
#先在/etc/sysctl.conf中设置net.ipv4.ip_forward=1
iptables -t nat -A PREROUTING -p tcp --dport 443 -j DNAT --to-destination 223.252.199.7:443
iptables -t nat -A POSTROUTING -j MASQUERADE
```
其中，`223.252.199.7`即`music.163.com`指向的网易云音乐服务器IP。
另外，据github网友 [Max-Sum](https://github.com/Max-Sum) 在 [issue#11](https://github.com/Chion82/163-music-unlock/issues/11) 中提到，使用`SNI Proxy`可实现根据域名转发，即可在反代服务器443端口上架设多个https服务。
若使用SNI Proxy，配置文件如下：
```
user daemon
pidfile /var/run/sniproxy.pid

error_log {
    syslog daemon
    priority notice
}

listen <YOUR_SERVER_IP>:443 {
    proto tls
    table https_hosts

    access_log {
        filename /var/log/sniproxy/https_access.log
        priority notice
    }
    fallback 127.0.0.1:443
}

table https_hosts {
    music.163.com 223.252.199.7:443
}
```

部分付费歌曲无法播放
----------------
免费午餐不会永久。前段时间，网易已经封了一部分付费歌曲，python脚本目前使用的API`http://music.163.com/api/song/detail/`，一部分付费歌曲已经不返回文件信息了。博主发现，部分付费专辑／付费歌曲（无论是否包月会员都需要付费的音乐）已经无法获取到音乐文件信息，而大部分包月付费包中的音乐还可以获取到最低音质的音乐文件信息。
