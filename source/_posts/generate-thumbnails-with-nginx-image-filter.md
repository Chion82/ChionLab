---
title: 使用nginx image filter动态生成缩略图
date: 2016-02-02 01:05:24
tags:
- nginx
- backend
categories:
- 学习笔记
- 后端
---

nginx提供`ngx_http_image_filter_module`模块，可用来动态生成图片的缩略图。当然，最好的办法是在后端进行图片压缩。但是当不方便修改后端代码时，在牺牲些许性能的代价下，使用image filter生成缩略图还是很方便的。

编译安装nginx
------------
大部分预编译的nginx默认不带`ngx_http_image_filter_module`模块，这时需要手动编译nginx。
在执行`configure`时带上参数`--with-http_image_filter_module`。
在水果上编译可参考[OSX上编译安装nginx](/2016/02/02/compile-nginx-on-osx/)

在配置文件中使用image_filter生成缩略图
----------------------------------
示例：
```
location /images {
    image_filter resize 200 200;
    image_filter_buffer 10M;
    image_filter_jpeg_quality 90;
    root /path/to/website;
    index index.html;
}
```
其中：
* `image_filter resize 200 200`表示按比例缩放图片，长和宽中较大者为200。比如，原图大小为1000x500，处理后为200x100。
* `image_filter_buffer 10M`表示处理图片的缓冲区最大为10M。
* `image_filter_jpeg_quality 90`设置jpeg压缩质量为90%。

经过这样的配置后，访问`HOSTNAME/images/XXX.jpg|png|gif`即可得到经过压缩的缩略图。
