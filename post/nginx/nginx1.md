---
title: "Nginx学习笔记(一)"
date: 2017-02-13T21:07:16+08:00
draft: false
---
### 运行中的Nginx进程间的关系
在正式提供服务的产品环境下，部署nginx都是使用一个master进程来管理多个worker进程，
一般情况下，worker进程的数量与服务器上的CPU核心数相等。
每个worker进程都是繁忙的，他们真正提供互联网服务，master进程则很清闲，只负责监控管理
worker进程。
Nginx是支持单进程(master进程)提供服务的，那么为什么产品环境下要按照master-worker方式配置同时
启动多个进程呢？这样做的好处主要有以下两点：
- 由于master进程不会对用户请求提供服务，只用于管理真正提供服务的worker进程，所以master进程可以是唯一的，它仅专注于自己的纯管理工作，为管理员提供命令行服务，包括诸如启动服务、停止服务、重载配置文件、平滑升级程序等。master进程需要拥有较大的权限，例如，通常会利用root用户启动master进程。worker进程的权限要小于或等于master进程，这样master进程才可以完全地管理worker进程。当任意一个worker进程出现错误从而导致coredump时，master进程会立刻启动新的worker进程继续服务。
- 多个worker进程处理互联网请求不但可以提高服务的健壮性（一个worker进程出错后，其他worker进程仍然可以正常提供服务），最重要的是，这样可以充分利用现在常见的SMP多核架构，从而实现微观上真正的多核并发处理。因此，用一个进程（master进程）来处理互联网请求肯定是不合适的。另外，为什么要把worker进程数量设置得与CPU核心数量一致呢？这正是Nginx与Apache服务器的不同之处。在Apache上每个进程在一个时刻只处理一个请求，因此，如果希望Web服务器拥有并发处理的请求数更多，就要把Apache的进程或线程数设置得更多，通常会达到一台服务器拥有几百个工作进程，这样大量的进程间切换将带来无谓的系统资源消耗。而Nginx则不然，一个worker进程可以同时处理的请求数只受限于内存大小，而且在架构设计上，不同的worker进程之间处理并发请求时几乎没有同步锁的限制，worker进程通常不会进入睡眠状态，因此，当Nginx上的进程数与CPU核心数相等时（最好每一个worker进程都绑定特定的CPU核心），进程间切换的代价是最小的。

![Nginx 进程间的关系](https://images-cdn.shimo.im/ObBmOvEpWRo3wk2O/nginx_process.png!thumbnail)

### nginx配置相关
#### location模块中root和alias的区别
root方式的配置:
```bash
location /download/ {
    root /opt/web/html/;
}
```
如果请求的URI是/download/index/test.html，那么web服务器将会返回服务器上/otp/web/html/download/index/test.html文件的内容。

alias方式的配置:
```bash
location /conf {
    alias /usr/local/nginx/conf;
}
```
在URI向实际文件路径的映射过程中，已经把location后配置的/conf这部分字符串丢弃，因此，/conf/nginx.conf请求将根据alias path映射为
**/usr/local/nginx/conf**/nginx.conf (conf -> /usr/local/nginx/conf)<br/>
root可以放置到http, server, location或if块中，而alias只能放置在location块中。<br/>
alias后面还可以添加正则表达式，例如：<br/>
```bash
location .~ ^/test/(\w+)\.(\w+)$ {
    alias /usr/local/nginx/$2/$1.$2;
}
```
这样，请求在访问/test/nginx.conf时，nginx会返回/usr/local/nginx/conf/nginx.conf文件中的内容。

#### try_files
语法： try_files path1 [path2] uri;
配置块： server、location
try_files后要跟若干路径，如path1 path2...，而且最后必须要有uri参数，意义如下：尝试按照顺序访问每一个path,如果可以有效地读取，就直接返回这个path对应的文件结束请求，否则继续向下访问。如果所有path都找不到有效的文件，就重定向到最后的参数uri上。如：
```bash
try_files /system/maintenance.html $uri $uri/index.html $uri.html @other;
location @other {
    proxy_pass http://backend;
}
```
#### 文件操作的优化
- sendfile 系统调用
语法: sendfile on|off; <br.>
默认：sendfile off; <br/>
配置快： http、server、location <br/>
可以启用Linux上的sendfile系统调用来发送文件，它减少了内核态与用户态之间的两次内存复制，这样就会从
磁盘中读取文件后直接在内核态发送到网卡设备，提高了发送文件的效率。
- AIO系统调用
此配置项表示是否在FreeBSD或Linux上启用内核级别的异步文件I/O功能。注意，它与sendfile功能是互斥的。
- directio
语法：directio size|off;<br/>
默认：directio off;<br/>
配置快： http、server、location <br/>
此配置项在FreeBSD和Linux系统上使用O_DIRECT选项去读取文件，缓冲区大小为size, 通常对大文件的读取速度有优化作用.注意，它与sendfile功能是互斥的。
- directio_alianment
- 打开文件缓冲
语法：open_file_cache max = N [inactive=time] | off;<br/>
默认：open_file_cache off; <br/>

#### nginx反向代理配置
参考文章：

https://www.chenyudong.com/archives/nginx-reverse-proxy.html

http://natumsol.github.io/2016/03/16/nginx-basic/
