---
title: 在GitHub Pages上使用CloudFlare https CDN
date: 2016-01-28 00:49:18
tags:
- maintenance
categories:
- 运维
---

本站就是使用[hexo](https://hexo.io)搭建的静态Web站点，托管在[GitHub repo](https://github.com/Chion82/Chion82.github.io)，并使用GitHub Pages。
另外，阿里云最近也提供https的CDN服务，更适合用在国内链路质量要求高的站点。

GitHub Pages应用自定义域名
------------------------
默认情况下，访问GitHub Pages页面的域名为`username.github.io`，如果需要使用自己的域名（以下简称“你的域名”），可[参考官方的帮助文档](https://help.github.com/articles/setting-up-a-custom-domain-with-github-pages/)，其实非常简单：
1. 在repo根目录下创建`CNAME`文件，内容为你的域名。[本站的CNAME文件](https://github.com/Chion82/Chion82.github.io/blob/master/CNAME)
2. 在你的域名管理中心，添加一条`CNAME`记录，指向`username.github.io`。（将username替换为你的GitHub用户名）

现在，访问`http://你的域名` ，已经可以访问到站点首页了。而如果访问`http://username.github.io` （即原来的地址），将被302跳转到`http://你的域名`。

https的问题
----------
尝试直接访问`https://你的域名`，浏览器会报SSL_DOMAIN_NOT_MATCHED警告。因为GitHub Pages默认提供的SSL证书的根域名是`github.io`，和你的域名不相同。
而且，GitHub Pages不支持上传SSL证书。

使用CloudFlare
-------------
[CloudFlare](https://www.cloudflare.com)（以下简称CF）是一家CDN提供商，它的free plan里面就提供https服务（免费计划不能上传SSL）。现在可以通过CF实现：从用户到CDN服务器的连接为https，而CDN服务器到GitHub Pages服务器的连接为http。
1.注册并登录到CF。按照提示，在你的域名的管理中心，将域名的name server改为CF的name server。CF提供的NS如下：

|Type|Value                  |
|:--:|:---------------------:|
|NS  |bob.ns.cloudflare.com  |
|NS  |jamie.ns.cloudflare.com|

2.在CF的DNS设置页中，检查对应的子域名记录。博主的DNS记录如下：
  {% asset_img dns-config.png blog.chionlab.moe的DNS记录 %}
  其中，右侧的橙色云图标代表该条记录将经过CF的CDN加速。
  在这里设置的DNS记录，如果是CNAME记录或者A记录，若右边的STATUS为连通状态，CF都会在name server中将其设置为A记录并指向CF的CDN服务器（并根据用户所在地选择最优CDN），当用户通过该域名访问CF的CDN时（仅限http或https），CDN再转发到刚才填写的真实目的主机（即username.github.io）
  {% asset_img dig-result.png 博主的blog.chionlab.moe虽在CF中设置为CNAME到gh-pages，但dig结果是A指向CF的CDN %}
  CF正是通过这种动态DNS的方式实现CDN加速的。

设置https
--------
1. 在CF的Crypto页中，SSL设置为Flexible。这将允许CDN到github pages之间的访问为http。
2. 现在，通过`https://你的域名`已经可以访问站点首页了。

强制https
--------
CF提供Page Rules功能，可设置路由规则。通过规则中的`Always use https`选项，可以将用户强制跳转到https。博主的设置如下：
{% asset_img page-rules.png %}
