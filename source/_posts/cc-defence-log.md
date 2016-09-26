---
title: 运维纪录：遭遇CC攻击，防御与查水表
date: 2016-01-27 03:28:34
tags:
- maintenance
- linux
categories:
- 运维
---

博主之前完成了一个外包项目，最近两个月在负责这个项目的运维。这是一个web，主营不良资产催收O2O。由于可能存在竞争对手，有人试图攻击服务器。

事件回顾
-------
24日下午3点，博主正在去拜访亲戚家的路上，这时公司的菜鸟开发者突然从QQ上发消息过来，问我服务器是不是被黑了。我认为这个可能性不大。这个项目由我亲手带领团队开发，后端使用的是Python+Flask+PostgreSQL，前端使用nodejs+express实现的midway，服务器部署也是由博主亲手完成。这类技术栈，已公布的可直接利用的漏洞十分有限，再者，博主在领队开发时已多次强调安全的重要性，具体到每个API都对用户权限进行了严格认证，编码过程中也不存在可能被注入、被远程执行等低级的危险代码，于是博主认为服务器被web渗透的可能性非常小。当然，不排除黑客从web之外的服务渗透进入，但是服务器上除了web只有ssh服务（强密码+公钥认证），除非公司的开发者部署了其他服务，否则能渗进来的可能性不大。

博主于是立即用手机访问网站，网站返回了504，这说明nginx的上游没有响应了，node midway或者Python后端，肯定有一个处于freeze状态。

到达亲戚家后，经过简短的问候，我即问道有没有能用的电脑。朋友让我使用一台08年的笔记本，运行的XP系统，只有IE8和360安全浏览器，但是已经够用了。下载Putty后ssh连接上服务器，立即`killall supervisord && supervisord`。因为node midway和Python后端都处于开发中状态，为了调试方便，所以直接是用supervisor作为daemon的。结果是，网站首页仍然返回504。

下意识地`tail /var/log/nginx/access.log -n 100`，出来的结果让我目瞪口呆.jpg
{% asset_img cc_access_log_1.jpg nginx access log:图中/api开头的URL全是短信API %}
我立即就知道是怎么一回事了：黑客在flood发送短信的API。由于当时开发急促，没有对短信API加入图形验证码或者reCaptcha之类的验证，使得可以通过软件实现模拟请求，并且由于项目处于开发中，为方便调试没有使用wsgi容器调度请求和超时处理，再者，由于发送短信需要服务器向第三方短信平台请求，这个请求将比较费时，同时的大量请求使得Python后端完全被阻塞，难怪nginx报504。从log上看，flood来源自多个不同的IP，这是分布式的攻击，算得上是一场小型的CC攻击。后来发现参与这次CC的肉鸡大概有700～800台。

**出于保密原则，本文以下内容中，发送短信API的URI均由[SMS_API]代替**

应急防御
------
运行了一下`cat /var/log/nginx/access.log | grep '[SMS_API]' | wc -l`，返回的数字超过了30万，这时公司购买的短信平台套餐肯定已经用光了。但是现在首先要考虑恢复网站的正常访问。

对于这种小型的CC防御，除了ban ip之外我没有想到更好的解决方法。于是，我用ipset+iptables将当天访问过短信API的IP全部ban了。
```
# ipset create blacklist hash:net
# cat /var/log/nginx/access.log | grep '[SMS_API]' | awk '{print $1}' | while read line;do ipset add blacklist $line;done  #将访问过短信API的IP全部加入ipset的blacklist集合
# iptables -I INPUT -m set --match-set blacklist src -j DROP
```
> 笔记： iptables -m set --match-set [SET_NAME] [src|dst]

执行后，再查看access log，flood马上就停下来了。但是现在遇到了新问题：后端跑不起来了。

修复后端
-------
手动运行后端Python脚本，Peewee报不能连接上数据库。
跑了一下psql，发现正常读取数据，再查看PostgreSQL的log，没有发现异常。没有头绪，通知公司的后端开发者检查后端代码。
公司的开发者没有回应，我折腾了很久找不到问题所在，直到我想到会不会是刚才添加iptables过滤规则时把本机也过滤了。
试着运行了一下
```
# ipset test blacklist 127.0.0.1
```
返回
```
127.0.0.1 is in set blacklist.
```
再次目瞪口呆.jpg。突然想起来，刚才我为了测试短信接口，在服务器上跑了一下`curl localhost/[SMS_API]`，于是nginx access log中就有了127.0.0.1，然后在跑脚本的时候就把127.0.0.1加入到blacklist中了。立即运行了一下
```
# ipset del blacklist 127.0.0.1
```
再次重启后端，一切正常，网站也能够访问了。

nginx中添加访问限制
-----------------
目前后端是从session判断唯一用户的，并限制每个用户每分钟只能调用短信API一次。但是如果黑客手动清空cookie，服务器将允许再次请求。在nginx的文档中快速查找了一下，发现nginx支持从IP上request limit。现在需要限制1 request/min per IP，为此修改nginx配置：
1. 添加limit_req_zone
  ```
  # /etc/nginx/nginx.conf
  http {
    limit_req_zone $binary_remote_addr zone=sms:10m rate=1r/m;
    ...
  }
  ```
2. location中应用limit_req_zone
  ```
  server {
    ...
    location ~ ^([SMS_API]) {
        limit_req zone=sms nodelay;
        proxy_pass http://127.0.0.1:5000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
    ...
  }
  ```
经过这样的配置，同一IP在一分钟内只能访问该URL一次，否则返回503 server unavailable。

脚本实现自动Ban IP
----------------
之后发现源源不断地还有更多IP试图发起CC，不可能人工一个一个的ban，于是写了一个简单shell脚本实现：当天access log中，访问短信API超过30次的IP，将被加入黑名单。当然，这只是临时的，生产环境中，对于同一内网中的多个真实用户可能会出现误ban的情况，因此攻击过后要将脚本关闭。
``` shell
#!/bin/sh

while [ True ]
do
        cat /var/log/nginx/access.log | grep '[SMS_API]' | awk '{print $1}' | sort | uniq -c | awk '$1>30{print $2}' | while read line;do echo 'Blocking IP:'$line && ipset add blacklist $line;done
        sleep 10
done
```

找出攻击发起者
-----------
由于CC分布式的特征，很难找出真正的攻击发起者。但是，往往可以找到第一个嫌疑者。通过翻看当天上午的access log，发现如下有趣的信息:
```
113.232.156.* - - [23/Jan/2016:11:19:13 +0800] "GET /register.html HTTP/1.1" 200 8383 "https://www.google.com/" "Mozilla/5.0 (Windows NT 5.1) AppleWebKit/537.1 (KHTML, like Gecko) Chrome/21.0.1180.89 Safari/537.1" "-" #使用Chrome进入网站注册页
#...
#下面若干行纪录均为页面静态资源请求
#...
113.232.156.* - - [23/Jan/2016:11:19:20 +0800] "GET [SMS_API]?phone=1584059XXXX HTTP/1.1" 200 46 "http://网站域名.com/register.html" "Mozilla/5.0 (Windows NT 5.1) AppleWebKit/537.1 (KHTML, like Gecko) Chrome/21.0.1180.89 Safari/537.1" "-"
#在Chrome中点击“发送短信验证码”按钮
```
正常的UA（Chrome 21.0.1180.89），并且有静态资源访问记录，基本可以确定是人工操作。
继续翻：
```
113.232.156.* - - [23/Jan/2016:11:19:27 +0800] "GET /register.html HTTP/1.1" 200 8383 "-" "Mozilla/5.0 (compatible; MSIE 9.0; Windows NT 6.1; WOW64; Trident/5.0)" "-"
#注意，这个人在1分钟内使用了IE9重新进入网站注册页
#...
#下面若干行纪录均为页面静态资源请求
#...
113.232.156.* - - [23/Jan/2016:11:19:35 +0800] "GET [SMS_API]?phone=1552442XXXX HTTP/1.1" 200 46 "http://网站域名.com/register.html" "Mozilla/5.0 (compatible; MSIE 9.0; Windows NT 6.1; WOW64; Trident/5.0)" "-"
#这个人在1分钟内使用IE9再次点击“发送短信验证码”按钮
```
普通用户是不会同时使用两款浏览器登录同一个网站并点击发送短信按钮的。除非——你想要验证这个网站是否根据session判断同一用户是否在一分钟内调用了多次发送短信API。再往后翻记录：
```
113.232.156.* - - [23/Jan/2016:11:19:51 +0800] "GET [SMS_API]?phone=1504032XXXX HTTP/1.1" 200 46 "-" "-" "-"
```
果不其然，这个人用模拟请求调用了发送短信API（因为没有正常的UA）
在这的几分钟后，来自全国各地的肉鸡就开始flood服务器了。

人肉攻击发起者
------------
换位思考一下，如果我是黑客，在开始CC之前，是否需要测试一下这个API，然后再在肉鸡上配置随机手机号，最后再进行CC？
再次翻log，发现flood开始后，来自肉鸡的请求中，手机号码来自全国各地，但是都每个号码重复了很多次，并且每台肉鸡都有自己的手机号。据此可判断，肉鸡用的手机号码一定不是黑客本人或相关者的号码，而应该是随机生成的或者是通过非法渠道获取到的“受害者”的手机号。但是，113.232.156.* （即黑客嫌疑者）一开始在Chrome和IE9中用的号码在后面的记录中都没有找到，并且号码所属地和IP所属地吻合（都为辽宁沈阳），据此，怀疑黑客一开始在Chrome中会用真实的手机号先进行测试，然后再实施CC。
将黑客IP和他第一次在浏览器中提交的手机号码（1584059XXXX）告诉了公司，公司立即拨打了这个手机号码。
对方一开始不承认。后来对方打回来，问我们是什么网站做什么的，并且听到对面几个人在偷着乐。因此，对方很有可能就是这次攻击的发起者，并且可能是黑客团伙，专职外包。（其实博主认为，国内这种组织根本算不上真正意义上的黑客，只是非常低级的为了图利的非法技术组织，并且自身技术也是很菜...）
公司随后开始通过手机号码查询该人身份信息。由于公司本身性质的关系，有后台可以调查某些信息。
之后的事情我就没有多问了。
