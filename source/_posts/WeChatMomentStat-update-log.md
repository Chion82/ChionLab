---
title: WeChatMomentStat：微信朋友圈导出工具开发记录
date: 2016-03-31 10:32:52
tags:
- android
- wechat
- hack
categories:
- 开发笔记
- 安卓
---

GitHub repo
-----------
https://github.com/Chion82/WeChatMomentStat-Android

关于WeChatMomentStat-Android
---------------------------
博主之前开发过[WeChatMomentExport](https://github.com/Chion82/WeChatMomentExport)，借助Xposed实现了导出微信朋友圈数据。该项目在GitHub上获得了不少Star，被应用平台收录之后也有几千的下载量，可见这个需求是存在的。但是，对于WeChatMomentExport，还存在以下问题：  
* 作为Xposed模块，必需依赖Xposed才能运行  
* 因为数据抓取方式为hook，故用户需要在微信朋友圈页面手动下滑加载
* 微信版本每更新一次会导致源码被重新混淆，相应的本项目也需要更新钩子逻辑
* 项目的定位是将导出数据作为开发者二次开发所需的数据源，但从酷安网的用户评论看，普通用户不能理解需求

对于上述问题，博主考虑了以下相应对策：
* 从[上次的逆向分析结果](https://blog.chionlab.moe/2016/02/20/wechat-sns-reflect-classes/)看，只要想办法调用到这几个类（以下称为parser），就可以解析微信SQLite缓存中的blob数据，这样就不需要借助Xposed的hook了，也能实现一键导出
* 考虑到blob格式不会经常变更，因此可在项目中整合parser，这样本项目就无需经常更新
* 博主在开发WeChatMomentExport之后随手写的[朋友圈数据统计脚本](https://github.com/Chion82/WeChatMomentStat)也获得了少量star，因此认为，对于普通用户，生成这样的简易统计数据更有吸引性

于是，决定整合WeChatMomentExport和统计脚本，做一个功能稍完善的工具。

几个技术难点
----------
要做这样的一个独立的APP，而不是一个Xposed模块，需要解决以下问题：
1. 如何在APP中整合parser？parser的逻辑代码被混淆在微信的dex中，直接分析其算法难度太大。
2. 如何越权获得微信的SQLite缓存数据？
3. 如何确保从SQLite缓存中取得的朋友圈数据足够齐全？

经过查阅各种文档和亲自实验，还是找到了解决方案。

## 使用DexClassLoader直接加载微信apk中的parser
DexClassLoader可直接解析apk中的classes.dex，并从中取得所需类，通过java反射，可以获得所需的parser方法。因此，无需再分析parser算法，而是直接调用就可以了。
通过DexClassLoader取得parser方法的关键代码如下：
```java
DexClassLoader cl = new DexClassLoader(
                    apkFile.getAbsolutePath(),  //apkFile为微信apk文件
                    context.getDir("outdex", 0).getAbsolutePath(),
                    null,
                    ClassLoader.getSystemClassLoader());

Class SnsDetailParser = cl.loadClass("com.tencent.mm.plugin.sns.f.i");
Class SnsDetail = cl.loadClass("com.tencent.mm.protocal.b.atp");
Class SnsObject = cl.loadClass("com.tencent.mm.protocal.b.aqi");
//之后只需使用java反射即可取得所需方法
```
还需要提供一个微信的apk文件。因此将微信apk放在assets中，首次运行本工具的时候释放到外部存储中。

## 通过su调用，拷贝微信的SQLite数据库文件
需要越权操作的话，获取root权限是很难避免的。通过调用su，可以复制出微信的SQLite数据库文件到本工具可读写的目录下。
微信朋友圈的SQLite文件在`/data/data/com.tencent.mm/MicroMsg/XXXXXXXXXXXXX/SnsMicroMsg.db`。其中，`XXXXXXXXXXXXX`是微信生成的hash值，每台设备上都可能不一样。由于在Android的shell中没有`find`或类似的命令，需要复制出这个`SnsMicroMsg.db`还得费一点功夫。最终，博主采用`ls`列目录并循环尝试`cp`的方法强行取得`SnsMicroMsg.db`。
```java
public void copySnsDB() throws Throwable {
    String dataDir = Environment.getDataDirectory().getAbsolutePath();
    String destDir = Config.EXT_DIR;
    Process su = Runtime.getRuntime().exec("su");
    DataOutputStream outputStream = new DataOutputStream(su.getOutputStream());
    outputStream.writeBytes("mount -o remount,rw " + dataDir + "\n");
    outputStream.writeBytes("cd " + dataDir + "/data/" + Config.WECHAT_PACKAGE + "/MicroMsg\n");
    outputStream.writeBytes("ls | while read line; do cp ${line}/SnsMicroMsg.db " + destDir + "/ ; done \n");
    outputStream.writeBytes("sleep 1\n");
    outputStream.writeBytes("chmod 777 " + destDir + "/SnsMicroMsg.db\n");
    outputStream.writeBytes("exit\n");
    outputStream.flush();
    outputStream.close();
    Thread.sleep(1000);
}
```
其中，还需要修改db文件的权限为`777`，否则工具无权读取数据库。另外，`sleep`是为了避免稍后偶然性出现的读取数据库失败的情况（可能文件复制不完整或未被去锁？）。

## 关于SQLite中数据完整性的问题
经过测试，微信的SQLite数据库中缓存了几乎所有加载过的朋友圈，理论上应当不会漏数据。

题外话
-----
本来这个app计划于2月中旬就写出来的，由于博主不是安卓开发者，没有系统地学过安卓开发，当时还不知道有`DexClassLoader`，写的第一个demo用的依然是Xposed，但是不同于WeChatMomentExport，这里用Xposed仅仅是为了取得那几个parser的类而已。2月底开学后，通过各种渠道了解到了`DexClassLoader`，才有现在的这个思路。
博主现在读大二，这学期开学后课程比较紧张，再者在工作室外包项目的压力下（团队管理问题，还有涉及的利益问题出现冲突的时候，处理起来非常棘手），一时失去了搞开源轮子的动力，甚至连续一个月都没有更新博客，于是才导致了这个项目拖到现在才基本完成。
看到了GitHub上的项目star和follower每隔几天就多一个，本站也陆续有网友来评论，每日UV也保持在100以上，就重拾了动力去继续折腾。
非常感谢前来光临本站和GitHub profile的各位！
