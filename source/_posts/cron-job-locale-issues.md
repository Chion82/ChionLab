---
title: cron任务的locale问题
date: 2016-04-29 21:15:58
tags:
- linux
- maintenance
categories:
- 学习笔记
- linux
---

原文：[Linux - cron での locale の挙動！ - mk-mode BLOG](http://www.mk-mode.com/octopress/2013/11/26/linux-cron-locale-behavior/)

> こんばんは。
> Linux で、自分が作成したスクリプトがコンソール上では正常に動作するのに、 cron で定時起動させようとすると文字コードの関係でうまく日本語出力ができないことがあります。
> 以下、それについての備忘録です。

晚上好。
在Linux下，自己编写的（shell）脚本，在终端下手动运行是一切正常的。但是，由于字符编码的关系，当cron在试图以定时任务来执行该脚本时，日语文字却不能被正常输出。
以下是解决这一问题的备忘录。

> 0\. 前提条件
> CentOS 6.4 (32bit) での作業を想定。
> cron は crontab -e ではなく、 /etc/cron.d/ ディレクトリ配下にファイルを設置する方法。
> 文字化けが起こるスクリプトは “UTF-8” でエンコードされていて、日本語出力を伴うことを想定。
> （当然、日本語出力を伴わないのならロケールの心配もない）

## 0\. 条件
* 假定操作系统是CentOS 6.4 (32bit) （译者注：6.X, 64位同样适用）
* 不使用cron的`crontab -e`，而是在`/etc/cron.d/`目录下建立配置文件来设置cron任务（译者注：同样适用于通过`crontab -e`设置的任务）
* 脚本使用UTF-8编码，并假定脚本的执行将伴随有日语文字输出，且（由cron执行时）出现了乱码。
（当然，如果日语输出不受locale影响，则无需担心。）

> 1\. cron 外（コンソール）でのロケール
> 普通にコンソールで locale コマンドでロケールを確認してみる。

## 1\. 在cron外部（用户终端）的locale
在一般的用户终端（console）中，尝试通过`locale`命令来确认当前环境的locale。
```python
# locale
LANG=ja_JP.UTF-8
LC_CTYPE="ja_JP.UTF-8"
LC_NUMERIC="ja_JP.UTF-8"
LC_TIME="ja_JP.UTF-8"
LC_COLLATE="ja_JP.UTF-8"
LC_MONETARY="ja_JP.UTF-8"
LC_MESSAGES="ja_JP.UTF-8"
LC_PAPER="ja_JP.UTF-8"
LC_NAME="ja_JP.UTF-8"
LC_ADDRESS="ja_JP.UTF-8"
LC_TELEPHONE="ja_JP.UTF-8"
LC_MEASUREMENT="ja_JP.UTF-8"
LC_IDENTIFICATION="ja_JP.UTF-8"
LC_ALL=
```

> 2\. cron 内でのロケール
> 次に cron 内で locale コマンドを実行させてみる。
> 例えば、以下のようなファイル /etc/cron.d/locale_test を作成してみる。

## 2\. cron内的locale
接下来，我们尝试在cron内执行`locale`命令。（译者注：其实就是在cron job中运行`locale`命令）
如下例，尝试创建一个文件`/home/hoge/work/locale.log`
```
* * * * * root locale > /home/hoge/work/locale.log
```
> 毎分 “/home/hoge/work/” ディレクトリ内に “locale.log” というファイルが作成されるので、内容を確認してみる。

每分钟，`/home/hoge/work/`下的`locale.log`文件都会被写入新数据，我们来尝试确认该文件内容。
```python
LANG=
LC_CTYPE="POSIX"
LC_NUMERIC="POSIX"
LC_TIME="POSIX"
LC_COLLATE="POSIX"
LC_MONETARY="POSIX"
LC_MESSAGES="POSIX"
LC_PAPER="POSIX"
LC_NAME="POSIX"
LC_ADDRESS="POSIX"
LC_TELEPHONE="POSIX"
LC_MEASUREMENT="POSIX"
LC_IDENTIFICATION="POSIX"
LC_ALL=
```
> “ja_JP.UTF-8” でなく “POSIX” となっている。
> これでは、UTF-8 でエンコードされているスクリプトは日本語表示で不具合を起こすでしょう。

`ja_JP.UTF-8`并不在`POSIX`集合内。
因此，使用UTF-8编码的脚本在遇到日语输出时会出错。

> 3\. 対処方法
> cron 内で UTF-8 でデンコードされたスクリプトを実行させる場合は、以下のように LC_CTYPE, LANG を設定してやる。

## 3\. 解决方法
要在cron中运行通过UTF-8编码的脚本，需要设定`LC_CTYPE`和`LANG`。如下：
```python
LC_CTYPE="ja_JP.utf8"
LANG="ja_JP.utf8"

* * * * * root locale > /home/hoge/work/locale.log
```
> 再度 “/home/hoge/work/” ディレクトリ内の “locale.log” の内容を確認してみる。

再次确认`/home/hoge/work/`目录下的`locale.log`文件的内容。
```python
LANG="ja_JP.utf8"
LC_CTYPE="ja_JP.utf8"
LC_NUMERIC="ja_JP.utf8"
LC_TIME="ja_JP.utf8"
LC_COLLATE="ja_JP.utf8"
LC_MONETARY="ja_JP.utf8"
LC_MESSAGES="ja_JP.utf8"
LC_PAPER="ja_JP.utf8"
LC_NAME="ja_JP.utf8"
LC_ADDRESS="ja_JP.utf8"
LC_TELEPHONE="ja_JP.utf8"
LC_MEASUREMENT="ja_JP.utf8"
LC_IDENTIFICATION="ja_JP.utf8"
LC_ALL=
```

> “ja_JP.utf8” になりました。（UTF-8 と utf8 の違いはあるが問題ない）
> これで、日本語出力で文字化けすることがなくなります。

现在是`ja_JP.utf8`了。（UTF-8和utf8的区别并不是个问题）
现在，（cron job任务的）的日语输出不会再乱码了。

> 4\. 参考
> 上記では任意のスクリプトについて話したが、UTF-8 エンコードの Ruby スクリプト（日本語出力を伴うもの）を cron 起動させるには以下のように -Ku オプションで文字コードを指定することでも対処可能である。

## 4\. 参考
上面的记录是针对任意的脚本。若需通过cron运行含有日语输出的Ruby脚本，可以通过`-Ku`选项指定字符编码。如下：
```
* * * * * root /usr/local/bin/ruby -Ku test_script.rb
```

> 5\. 後始末
> 当然、テストで作成した cron スクリプトは不要なので削除しておく。
>
> 以上。

## 5\. 后续清理
当然，在刚才的测试中添加的cron任务脚本（locale命令）是不再需要的，请删除它。

## 译者注
本文locale问题的解决方案对于简体中文也是同样适用的，只需将本文中的`ja_JP`替换成`zh_CN`即可。
