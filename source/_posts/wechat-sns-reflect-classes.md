---
title: 逆向纪录：微信朋友圈相关的几个类
date: 2016-02-20 00:16:42
tags:
- wechat
- android
- hack
categories:
- 逆向
---

本纪录针对微信安卓端版本`6.3.13`。本文纪录逆向微信过程中找到的几个朋友圈内容相关的数据结构类。

1. 朋友圈详细内容
类名: `com.tencent.mm.protocal.b.atp`
[方法]添加属性（可作为hook的方法）: `protected final int a(int paramInt, object... objectArray)`
[方法]从BLOB数据导入：`public a am(byte[])`

2. 可将`com.tencent.mm.protocal.b.atp`实例格式化为XML的类
类名: `com.tencent.mm.plugin.sns.f.i`
[方法]输出朋友圈内容XML: `static public String a(com.tencent.mm.protocal.b.atp atpObject)`

3. 朋友圈评论和点赞数据
类名: `com.tencent.mm.protocal.b.aqi`
[方法]添加属性（可作为hook的方法）: `protected final int a(int paramInt, object... objectArray)`
[方法]从BLOB数据导入：`public a am(byte[])`
[属性]用户ID：`String iYA`
[属性]用户昵称：`String jyd`
[属性]时间戳：`long fpL`
[属性]评论列表：`LinkedList<com.tencent.mm.protocal.b.apz> jJX`
[属性]点赞列表：`LinkedList<com.tencent.mm.protocal.b.apz> jJU`

4. 评论或点赞数据详情
类名: `com.tencent.mm.protocal.b.apz`
[属性]用户ID： `String iYA`
[属性]用户昵称：`String jyd`
[属性]评论回复给谁(对方用户ID)：`String jJM`
[属性]评论内容：`String fsI`
