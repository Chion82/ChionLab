---
title: 网易云音乐APP反代开发笔记
date: 2016-01-21 21:27:50
tags:
- python
- hack
categories:
- 开发笔记
- Python
---

前几个月网易云音乐下架了一批版权音乐，网页端和手机客户端都无法播放。后来博主通过逆向网页端的JavaScript，写了一个[Chrome插件](https://github.com/Chion82/163_music_cracker)替换页面JS，使得可以播放下架和收费音乐，但最近网易已修复，插件已失效。（不排除是因为网易更新前端代码时重新构建混淆了JS导致我写的patch失效）。本笔记主要记录移动客户端的反向代理破解方案。

项目地址
-------
https://github.com/Chion82/163-music-unlock

关于下架、收费音乐
---------------
网易云音乐上的每首歌都有唯一ID，通过歌曲ID调用API可获得每个资源文件的ID，将资源文件ID加密后可获取到mp3链接地址，从[网易云音乐命令行版](https://github.com/darknessomi/musicbox)源码中可获得链接拼接方式。经测试，这样拼接出来链接同样适用于下架和付费音乐。从歌曲ID获取资源文件URL的关键代码如下:
``` python
#代码来自darknessomi/musicbox
...
# 歌曲加密算法, 基于https://github.com/yanunon/NeteaseCloudMusic脚本实现
def encrypted_id(id):
    magic = bytearray('3go8&$8*3*3h0k(2)2')
    song_id = bytearray(id)
    magic_len = len(magic)
    for i in xrange(len(song_id)):
        song_id[i] = song_id[i] ^ magic[i % magic_len]
    m = hashlib.md5(song_id)
    result = m.digest().encode('base64')[:-1]
    result = result.replace('/', '_')
    result = result.replace('+', '-')
    return result
...
# song id --> song url ( details )
def song_detail(self, music_id):
    action = "http://music.163.com/api/song/detail/?id=" + str(music_id) + "&ids=[" + str(music_id) + "]"
    try:
        data = self.httpRequest('GET', action)
        return data['songs']
    except:
        return []
...
# 获取高音质mp3 url
def geturl(song):
    config = Config()
    quality = Config().get_item("music_quality")
    if song['hMusic'] and quality <= 0:
        music = song['hMusic']
...
    else:
        return song['mp3Url'], ''
...
    #拼接最终资源文件URL
    song_id = str(music['dfsId'])
    enc_id = encrypted_id(song_id)
    url = "http://m%s.music.126.net/%s/%s.mp3" % (random.randrange(1, 3), enc_id, song_id)
    return url, quality
```

移动客户端抓包
------------
在路由器上运行tcpdump抓包，使用wireshark分析出几个关键API，这些API用于网易云音乐移动客户端（IOS）获取歌曲信息。

1. 获取播放列表详细信息
  POST: http://music.163.com/eapi/v3/playlist/detail/
  该API返回JSON，经JSON viewer解析后部分内容如下：
  <img src="/images/netease-api-2.png" width="200px">
  上图privileges数组的每个元素中，"fee", "payed", "st"等字段分别标示每首音乐的状态。其中0号元素是下架音乐，3号元素的音乐可正常播放，存在不同的参数一目了然。其中"fee"和"payed"和付费歌曲有关。

2. 获取音乐资源URL
  ```
  POST: http://music.163.com/eapi/song/enhance/player/url
  =>
  {"data":[{"id":26117507,"url":"http://m8.music.126.net/20160121231817/b4980525dca8b023c187321115a2463d/ymusic/5e8a/e1af/5a94/bd16133f4ebc8d5a5aaa5a44c6111813.mp3","br":320000,"size":10722786,"md5":"bd16133f4ebc8d5a5aaa5a44c6111813","code":200,"expi":1200,"type":"mp3","gain":-6.15,"fee":0,"uf":null,"payed":0,"canExtend":false}],"code":200}
  ```

Hack the World
--------------
从抓包结果可知，只需更改几个播放列表的参数，客户端即可播放下架和付费音乐。因此需要架设一个反向代理服务器，使用nginx的sub_filter实现文本替换，nginx的关键配置如下：
``` shell
server {
        listen 80;
        server_name music.163.com;
        resolver 114.114.114.114;
        set $backend_upstream "http://music.163.com";

        location / {
               proxy_pass $backend_upstream;
               proxy_set_header Host $host;
               proxy_set_header X-Real-IP $remote_addr;
               proxy_set_header Accept-Encoding ""; #十分重要，默认为gzip encoding，sub_filter无法识别
               sub_filter '"st":-200' '"st":0';
               sub_filter '"st":-100' '"st":0';
               sub_filter '"st":-1' '"st":0';
               sub_filter '"st":-2' '"st":0';
               sub_filter '"st":-3' '"st":0';
               sub_filter '"pl":0' '"pl":320000';
               sub_filter '"dl":0' '"dl":320000';
               sub_filter '"fee":1' '"fee":0';
               sub_filter '"sp":0' '"sp":7';
               sub_filter '"cp":0' '"cp":1';
               sub_filter '"subp":0' '"subp":1';
               sub_filter '"fl":0' '"fl":320000';
               sub_filter_once off;
               sub_filter_types *;
        }

        location /eapi/song/enhance/player/url {  #这里是需要重新拼接歌曲URL的代理，稍后将说明
               proxy_pass http://localhost:5001;
        }
}
```
通过几个sub_filter，可将歌曲参数从下架／收费状态更改成普通状态。但是对于下架和收费歌曲，通过 http://music.163.com/eapi/song/enhance/player/url 这个API无法直接获取到歌曲播放地址，因此需要借助前文提到的URL拼接方法提供新的URL。在清楚歌曲ID加密算法后，在借助Flask的帮助下能快速写出一个API server，在nginx中将相应请求反代到这个API server即可。
