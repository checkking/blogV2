---
title: "tcp自连接问题"
date: 2017-03-12T21:07:16+08:00
draft: false
---
#### 现象重现
在linux主机下运行下面的python脚本，等待一会即可出现。

```python
import socket
import time

connected=False
while (not connected):
    try:
        sock = socket.socket(socket.AF_INET,socket.SOCK_STREAM)
        sock.setsockopt(socket.IPPROTO_TCP,socket.TCP_NODELAY,1)
        sock.connect(('127.0.0.1',55555))
        connected=True
    except socket.error,(value,message):
        print message

        if not connected:
            print "reconnect"

print "tcp self connection occurs!"
print "try to run follow command : "
print "netstat -an|grep 55555"
time.sleep(1800)
```
截图如下：
![tcp自连接](https://images-cdn.shimo.im/updE9v1RXnQem98L/20170312.png!thumbnail)

tcp自连接出现了！

#### 原因分析
从上面的python脚本中，可以看到它只是在不断地尝试连接55555这个端口，并且是没有socket监听这个端口，那么为何最后却建立连接了呢？原因在于客户端在连接服务端时，如果没有指定端口号，系统会随机分配一个。随机就意味着可能分配一个和目的端口一样的数字，此时就会出现自连接情况了。因为对于tcp协议来讲，连接的流程是走的通，三次握手整个阶段都合法，连接自然可以建立。

自连接的坏处显而易见，当程序去connect一个不处于监听的端口时，必然期待其连接失败，如果自连接出现，就意味着该端口被占用了，那么：

1. 真正需要监听该端口的服务会启动失败，抛出端口已被占用的异常。
2. 客户端无法正常完成数据通信，因为这是个自连接，并不是一个正常的服务。

#### 解决思路
解决办法也很简单，只要保证客户端随机的端口不会和服务监听的端口相同就可以了。那么我们得先了解随机的范围，这个范围对应linux的`/etc/sysctl.conf的net.ipv4.ip_local_port_range`参数，其默认值是`32768 61000`。也就是说随机端口会在这个范围内出现，试验中我们选定了`55555`这个端口，所以出现了自连接现象。此时只要限定服务监听在`32768`端口以下，就不会出现自连接现象了。当然，你可以修改这个配置，只要注意保证监听端口不再配置范围内就可以避免自连接问题了。
t