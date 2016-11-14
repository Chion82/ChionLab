---
title: "TCP keepalive的探究 (2) : 浏览器的Keepalive机制"
date: 2016-11-07 19:40:07
tags:
- network
- maintenance
- linux
categories:
- 学习笔记
- linux
---

上文介绍了TCP Keepalive机制以及其在linux中的编程实现，本文将继续介绍这种机制在浏览器中的运用，并以Chrome为例。

HTTP1.1中的Connection: Keep-Alive
--------------------------------
在介绍Chrome对TCP Keepalive的实现之前，我们先来了解一下第七层协议HTTP1.1中的Connection字段。注意，本章节讨论的Keepalive为七层协议(HTTP1.1)中的Keep-Alive机制。

HTTP1.1协议头(header)中的`Connection`字段可取这两个值的其中之一：`keep-alive`, `close`。
该字段在请求头(request header)和响应头(response header)中都可以存在，这说明，客户端可以申请开启Keep-Alive，而服务端可以接受Keep-Alive请求，或者拒绝并在响应头中告知客户端。

### 作用机理
这里以一次完整的HTTP1.1网站访问来说明。

1. 客户端浏览器向 `www.bilibili.com:80` 建立TCP连接，并在此TCP连接上传输七层报文，请求`GET /index.html`资源，在请求头中，`Connection`置为`keep-alive`。
2. 服务端向浏览器返回`index.html`的文件内容，响应报头中`Connection`置为`keep-alive`，随后，**不关闭和客户端的TCP连接**。
3. 客户端复用该TCP连接，并请求`GET /style.css`资源，请求头置`Connection`为`keep-alive`。
4. 服务器向浏览器返回`index.css`文件内容，仍然不关闭该TCP连接。
5. 客户端继续复用该TCP连接请求多个同域资源。
6. 客户端所需的各种资源都请求完毕，但是因为客户端的最后一次资源请求头中仍置`Connection`为`keep-alive`，该TCP连接仍未被关闭。
7. 如果在一段时间（通常是3分钟左右）内客户端没有使用该TCP连接请求资源，服务器可能会关闭该连接。连接被关闭后，客户端需要重新向该域建立TCP连接才能继续请求数据。

{% asset_img http1.1.png HTTP1.1的请求示意图 %}

{% asset_img 10.png 一次HTTP1.1的请求和响应报头 %}

### 几点细节

* HTTP1.1的Keep-Alive机制仅对同域下的网络请求有效。比如，对于`http://www.bilibili.com/index.html`和`http://www.bilibili.com/style.css`这两个资源请求，浏览器能够复用其TCP连接，而对于非同域下的`http://space.bilibili.com/index.html`，则需要重新建立一次TCP连接。

* 服务器有权拒绝客户端的Keep-Alive请求，在响应头中置`Connection`为`close`，并在传输一次完整的响应报文后主动关闭TCP连接，在这之后，客户端如需向该域请求资源，则需重新建立TCP连接。而事实上，即使客户端和服务端都开启了Keep-Alive，服务端一般会主动关闭非活动的连接，否则会造成资源浪费。

* Keep-Alive虽然可以在一定程度上通过复用TCP连接来提高页面资源加载性能，但是受HTTP1.1的max-connection限制，提高的性能很有限。很多时候，为了加快更多资源的加载，通常会使用多个不同域名的CDN。而在HTTP2中，通过二进制数据帧的方式来传输同域下多资源，可以解决这个问题。关于HTTP2的传输机制，可以参考[这篇文章](https://segmentfault.com/a/1190000006923359)。

Chrome对TCP连接的保活机制
----------------------
上篇章节中我们熟悉了七层协议中HTTP1.1的Keep-Alive机制，本章节我们介绍Chrome对四层协议的TCP Keepalive的实现。

**Chrome何时需要启用TCP Keepalive？**
假定服务器启用了HTTP1.1 Keep-Alive，浏览器与服务器建立TCP连接，并在该TCP连接上有序地传输多个HTTP1.1七层报文，以此来请求多个资源。对于同域下，在浏览器完成一次请求并获得对应资源后，若一段时间内暂时未有新的资源请求（资源请求可能由页面JavaScript发出，如Ajax），直至下次请求发出前，该TCP连接保持空闲状态。而在这段空闲时间内，浏览器需要对该TCP连接进行保活。

下面我们将通过Wireshark抓包来验证。

{% asset_img 9.png Wireshark抓到的Chrome发出的TCP keepalive探测包 %}

从上面的抓包结果中看到，在服务器返回完整HTTP 200报文的45秒后（Time=72），本地发出了第一个TCP Keepalive探测包并收到来自服务器的ACK。

这说明，Chrome对于可复用的TCP连接，采用的保活机制是TCP层（传输层）自带的Keepalive机制，通过TCP Keepalive探测包的方式实现，而不是在七层报文上自成协议来传输其它数据。

而实际上，由于HTTP1.1对时序和报文的约定，浏览器也不可在七层实现保活。假设，客户端在通过HTTP1.1获取一次资源后，若在这个TCP连接上发送一个`0x70`（无意义的数据，在七层实现保活的方式大多如此），服务器会在应用层接收到并缓存该数据，一段时间后客户端发送有效的HTTP请求报头，则服务端CGI应用程序收到的数据是`0x70`再接上一段HTTP请求头，这被认为是无效的HTTP报文，服务器则会返回400响应头，告知客户端这是坏的请求（Bad Request）。

所以，浏览器在处理HTTP1.1请求所对应的TCP连接的保活时，通过使用TCP Keepalive机制，来避免污染七层（应用层）的传输数据。

待续
----
本篇主要介绍浏览器对TCP Keepalive的运用，内容简单。结合本篇内容，作者将在下篇文章中详细说明作者在使用shadowsocks浏览web时遇到的问题、解决方案以及一点思考。
