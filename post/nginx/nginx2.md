---
title: "nginx日志切分方案"
date: 2017-02-18T21:07:16+08:00
draft: false
---
nginx的日志切分问题一直是运维nginx时需要重点关注的。本文将简单说明下nginx支持的两种日志切分方式。
#### 定时任务切分
所谓的定时任务切分，是指通过定时任务（比如crontab)，发送信号给nginx，让其重新打开文件。该方法也是nginx官网上面比较推荐的,原文说明比较清楚，这里在说明下：<br/>
发送USR1 信号会让nginx主动重新打开日志文件，故操作如下：
```bash
$ mv access.log access.log.0
$ kill -USR1 `cat master.nginx.pid`
$ sleep 1
$ gzip access.log.0    # do something with access.log.0
```
总结 ：优点是思路较为简单，但效果明显，而且对error_log 同样适用；缺点是有外部依赖（比如 crontab)

#### 自切分
自切分是指让nginx自身实现日志切分功能，不依赖crontab等东西。 其主要原理是依赖access_log的强大功能---- 可以用变量定义请求的log路径。<br/>
nginx的acess_log 功能非常强大，其完整指令说明如下，这里主要说明定义日志路径的功能；关于syslog还有gzip, buffer等特性，后续再说明。

access_log指令Syntax:<br/>
access_log path [format [buffer=size [flush=time]] [if=condition]];
access_log path format gzip[=level] [buffer=size] [flush=time] [if=condition];
access_log syslog:server=address[,parameter=value] [format [if=condition]];
access_log off;

默认：access_log logs/access.log combined;<br/>
Context:    http, server, location, if in location, limit_except

注意path部分是支持nignx变量的，这也就意味这我们只要通过配置正确的nginx变量，就可以实现小时等级别的日志自动拆分了。

一个简单的问题就出现了，假设nginx要实现这个机制，那岂不是每打印一个请求log就得打开文件，写日志，关闭文件？ 这样显然效率太差了，为了解决这个问题，nginx又引入了一个机制，叫做 open_file_cache，简单的说，这个东西的功能就是会缓存打开的文件，只有满足一定条件的时候才会重新去check当前fd对应的文件是否合法，是否需要重新打开。 open file cache的指令如下：

Syntax:     open_log_file_cache max=N [inactive=time] [min_uses=N] [valid=time];
open_log_file_cache off;
Default:     open_log_file_cache off;
Context:     http, server, location

open_log_file_cache 里面几个参数的含义为：

max : 设置缓存中描述符的最大数量；如果缓存被占满，最近最少使用（LRU）的描述符将被关闭。

inactive : 设置缓存文件描述符在多长时间内没有被访问就关闭； 默认为10秒。

min_uses : 设置在inactive参数指定的时间里， 最少访问多少次才能使文件描述符保留在缓存中；默认为1。

valid :设置一段用于检查超时后文件是否仍以同样名字存在的时间； 默认为60秒。

off :禁用缓存。
综上，要让nginx自切分，需要两个步骤，其一，配置合理的access_log;其二，开启open_log_file_cache提升性能； 下面是用实现小时级别日志切分的配置demo

提取nginx变量
```bash
if ($time_iso8601 ~ "^(\d{4})-(\d{2})-(\d{2})T(\d{2}):(\d{2}):(\d{2})")
{
    set $year $1;
    set $month $2;
    set $day $3;
    set $hour $4;
    set $minutes $5;
    set $seconds $6;
}
```

配置access_log ；以  hour 为界
```bash
 access_log  logs/access.log.$year$month$day$hour;
```
配置open_log_file_cache
open_log_file_cache max=10 inactive=60s valid=1m min_uses=2;

总结 :
自切分可一定程度上面满足日志切分的需求；但是对性能会有一定的影响； 另外，并不支持error_log的切分，个人更推荐产品线采用方式一的方法切。
