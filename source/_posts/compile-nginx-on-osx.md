---
title: 在OSX上编译安装nginx
date: 2016-02-02 00:31:21
tags:
- nginx
- OSX
- backend
categories:
- 学习笔记
- OSX
---
在OSX上，一般情况下，使用`brew`安装nginx，再链接一个plist到`/Library/LaunchDaemons`即可。但是有时候brew中的nginx缺少某些模块，比如上文提到的`ngx_http_image_filter_module`，这时就需要重新编译nginx。

安装依赖
------
```
$ brew brew pcre
$ brew install gd #image filter依赖gd
```

编译
---
```
#进入nginx源码目录
$ ./configure --with-http_image_filter_module --with-http_ssl_module --with-cc-opt="-I /usr/local/include" --with-ld-opt="-L /usr/local/lib" --sbin-path=/usr/local/Cellar/nginx/nginx --conf-path=/usr/local/etc/nginx/nginx.conf --pid-path=/usr/local/var/run/nginx.pid --http-log-path=/usr/local/var/log/nginx/access.log --error-log-path=/usr/local/var/log/nginx/error.log
$ make
```
其中，`--with-http_image_filter_module`选项加入`ngx_http_image_filter_module`模块（如果不需要该模块可去除该选项），`--with-cc-opt="-I /usr/local/include" --with-ld-opt="-L /usr/local/lib"`可避免报`Undefined symbols for architecture x86_64`错误。

安装
---
如果之前未安装过nginx，运行这条命令来安装：
```
$ make install
```
如果已使用brew安装nginx，可以通过替换文件的方式换成刚才编译的版本：
```
#备份原来的binary
$ cp /usr/local/opt/nginx/bin/nginx /usr/local/opt/nginx/bin/nginx.bak
#进入nginx源码目录
$ sudo cp objs/nginx /usr/local/opt/nginx/bin/nginx
$ rm /usr/local/bin/nginx
$ ln -s /usr/local/opt/nginx/bin/nginx /usr/local/bin/nginx
```
现在，可以查看nginx版本：
```
$ nginx -V
```

强迫症患者可以像我这样在`Cellar`中建立一个新的版本目录：
```
$ cp -r /usr/local/Cellar/nginx/1.8.0 /usr/local/Cellar/nginx/1.9.10
#恢复1.8.0中的binary
$ rm /usr/local/Cellar/nginx/1.8.0/bin/nginx
$ mv /usr/local/Cellar/nginx/1.8.0/bin/nginx.bak /usr/local/Cellar/nginx/1.8.0/bin/nginx
#更新/usr/local/opt/nginx
$ rm /usr/local/opt/nginx
$ ln -s /usr/local/Cellar/nginx/1.9.10 /usr/local/opt/nginx
```

将nginx加入LaunchDaemons
-----------------------
编辑`/Library/LaunchDaemons/homebrew.mxcl.nginx.plist`内容如下：（brew的nginx自带的版本）
```
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
  <dict>
    <key>Label</key>
    <string>homebrew.mxcl.nginx</string>
    <key>RunAtLoad</key>
    <true/>
    <key>KeepAlive</key>
    <false/>
    <key>ProgramArguments</key>
    <array>
        <string>/usr/local/opt/nginx/bin/nginx</string>
        <string>-g</string>
        <string>daemon off;</string>
    </array>
    <key>WorkingDirectory</key>
    <string>/usr/local</string>
  </dict>
</plist>
```
然后：
```
$ launchctl load -F /Library/LaunchDaemons/homebrew.mxcl.nginx.plist
```
