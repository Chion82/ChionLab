---
title: 在CentOS6上编译安装Python2.7
date: 2016-01-22 02:11:53
tags:
- linux
- maintenance
categories:
- 干货
- linux
---

参考了友站的博文[CentOS6上的Python2.7问题](https://www.starduster.me/2016/01/04/py27-on-centos6/)，正如所言，在CentOS6上安装Python2.7是非常头疼的问题。友站的这篇博文阐述了如何了从源安装Python2.7，本站则讲述从源码编译安装要注意的问题。
编译依赖参考[How to install Python 2.7 and Python 3.3 on CentOS 6](http://toomuchdata.com/2014/02/16/how-to-install-python-on-centos/)。

依赖
---
**十分重要:** 编译Python2.7之前务必安装齐必须依赖。在configure过程中，若缺少依赖则不会报错，编译也可顺利通过，但编译出的Python将缺少几个必要模块，导致在运行`ez_setup.py`时出错。

``` shell
# yum groupinstall "Development tools"
# yum install zlib-devel bzip2-devel openssl-devel ncurses-devel sqlite-devel readline-devel tk-devel gdbm-devel db4-devel libpcap-devel xz-devel
```

编译和安装
--------
``` shell
$ wget http://python.org/ftp/python/2.7.6/Python-2.7.6.tar.xz
$ tar xf Python-2.7.6.tar.xz
$ cd Python-2.7.6
$ ./configure --prefix=/usr/local --enable-unicode=ucs4 --enable-shared LDFLAGS="-Wl,-rpath /usr/local/lib"
# make && make altinstall
```
这将会把Python2.7安装在`/usr/local/bin/python2.7`

将默认Python版本从2.6改为2.7
-------------------------
首先将`/usr/bin/python`这个软链接指向刚刚安装的Python2.7
```shell
# rm /usr/bin/python
# ln -s /usr/local/bin/python2.7 /usr/bin/python
```
**重要：** 进行这步操作后，yum会失效，运行即报错。这是因为`/usr/bin/yum`其实是个python2.6脚本，刚刚安装的python2.7缺少yum的相关依赖。因此需要改动`/usr/bin/yum`的解释器。
```shell
# vim /usr/bin/yum
```
将第一行
```
#!/usr/bin/python
```
改为：
```
#!/usr/bin/python2.6
```
现在运行`yum --version`应该不会再报错

安装pip
------
```shell
$ wget https://bitbucket.org/pypa/setuptools/raw/bootstrap/ez_setup.py
# python ez_setup.py
# easy_install-2.7 pip
```

替换默认pip为pip2.7
-----------------
```
# vim /usr/bin/pip2.6 #第一行改为#!/usr/bin/python2.6
$ which pip2.7  #应该返回/usr/local/bin/pip2.7
# rm /usr/bin/pip
# ln -s /usr/local/bin/pip2.7 /usr/bin/pip
```
