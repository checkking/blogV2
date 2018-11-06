---
title: "Websocket协议总结"
date: 2018-11-05T13:46:09+08:00
draft: false
---

### 概述
WebSocket协议被设计来取代现有的使用HTTP作为传输层的双向通信技术，并受益于现有的基础设施（代理、过滤、身份验证）。协议有两部分：握手和数据传输。
来自客户端的握手看起来像如下形式：
```python
GET /chat HTTP/1.1
Host: server.example.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Origin: http://example.com
Sec-WebSocket-Protocol: chat, superchat
Sec-WebSocket-Version: 13

```

重点请求首部意义如下：
* Connection: Upgrade：表示要升级协议
* Upgrade: websocket：表示要升级到 websocket 协议。
* Sec-WebSocket-Version: 13：表示 websocket 的版本。如果服务端不支持该版本，需要返回一个 Sec-WebSocket-Versionheader ，里面包含服务端支持的版本号。
* Sec-WebSocket-Key：与后面服务端响应首部的 Sec-WebSocket-Accept 是配套的，提供基本的防护，比如恶意的连接，或者无意的连接。

来自服务器的握手看起来像如下形式：
```python
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
Sec-WebSocket-Protocol: chat
```

* Sec-WebSocket-Accept 根据客户端请求首部的 Sec-WebSocket-Key 计算出来。 计算公式为：
```
toBase64(sha1(Sec-WebSocket-Key + 258EAFA5-E914-47DA-95CA-C5AB0DC85B11))
```

一旦客户端和服务器都发送了它们的握手，且如果握手成功，接着开始数据传输部分。 这是一个每一端都可以的双向通信信道，彼此独立，随意发生数据。

### 数据帧
下面给出了 WebSocket 数据帧的统一格式。熟悉 TCP/IP 协议的同学对这样的图应该不陌生。
从左到右，单位是比特。比如 FIN、RSV1各占据 1 比特，opcode占据 4 比特。内容包括了标识、操作代码、掩码、数据、数据长度等。
```
  0                   1                   2                   3
  0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
 +-+-+-+-+-------+-+-------------+-------------------------------+
 |F|R|R|R| opcode|M| Payload len |    Extended payload length    |
 |I|S|S|S|  (4)  |A|     (7)     |             (16/64)           |
 |N|V|V|V|       |S|             |   (if payload len==126/127)   |
 | |1|2|3|       |K|             |                               |
 +-+-+-+-+-------+-+-------------+ - - - - - - - - - - - - - - - +
 |     Extended payload length continued, if payload len == 127  |
 + - - - - - - - - - - - - - - - +-------------------------------+
 |                               |Masking-key, if MASK set to 1  |
 +-------------------------------+-------------------------------+
 | Masking-key (continued)       |          Payload Data         |
 +-------------------------------- - - - - - - - - - - - - - - - +
 :                     Payload Data continued ...                :
 + - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - +
 |                     Payload Data continued ...                |
 +---------------------------------------------------------------+
```

### 抓包分析
请求:
![websocket 建立链接](https://images2018.cnblogs.com/blog/1078987/201803/1078987-20180315190546389-362401249.png)

这是一次特殊的Http请求，为什么是一次特殊的Http请求呢？Http请求头中Connection:Upgrade Upgrade:websocket,Upgrade代表升级到较新的Http协议或者切换到不同的协议。很明显WebSocket使用此机制以兼容的方式与HTTP服务器建立连接。WebSocket协议有两个部分：握手建立升级后的连接，然后进行实际的数据传输。首先，客户端通过使用Upgrade: WebSocket和Connection: Upgrade头部以及一些特定于协议的头来请求WebSocket连接，以建立正在使用的版本并设置握手。服务器，如果它支持协议，回复与相同Upgrade: WebSocket和Connection: Upgrade标题，并完成握手。握手完成后，数据传输开始。这些信息在前面的Chrome控制台中也可以看到。

响应：
响应状态码 101 表示服务器已经理解了客户端的请求，在发送完这个响应后，服务器将会切换到在Upgrade请求头中定义的那些协议。
![websocket 响应](https://images2018.cnblogs.com/blog/1078987/201803/1078987-20180315190558077-197672425.png)

通信协议格式是WebSocket格式，服务器端采用Tcp Socket方式接收数据，进行解析，协议格式如下：
![websocket格式](https://images2018.cnblogs.com/blog/1078987/201803/1078987-20180315190609151-1724857868.png)

首先我们需要知道数据在物理层，数据链路层是以二进制进行传递的，而在应用层是以16进制字节流进行传输的。

第一个字节：
![第一个字节](https://images2018.cnblogs.com/blog/1078987/201803/1078987-20180315190618338-1359735849.png)

FIN:1位，用于描述消息是否结束，如果为1则该消息为消息尾部,如果为零则还有后续数据包;
RSV1,RSV2,RSV3：各1位，用于扩展定义的,如果没有扩展约定的情况则必须为0
OPCODE:4位，用于表示消息接收类型，如果接收到未知的opcode，接收端必须关闭连接。

Webdocket数据帧中OPCODE定义：
0x0表示附加数据帧
0x1表示文本数据帧
0x2表示二进制数据帧
0x3-7暂时无定义，为以后的非控制帧保留
0x8表示连接关闭
0x9表示ping
0xA表示pong
0xB-F暂时无定义，为以后的控制帧保留

第二个字节：
![第二个字节](https://images2018.cnblogs.com/blog/1078987/201803/1078987-20180315190625548-1649312784.png)
MASK:1位，用于标识PayloadData是否经过掩码处理，客户端发出的数据帧需要进行掩码处理，所以此位是1。数据需要解码。
PayloadData的长度：7位，7+16位，7+64位
如果其值在0-125，则是payload的真实长度。
如果值是126，则后面2个字节形成的16位无符号整型数的值是payload的真实长度。
如果值是127，则后面8个字节形成的64位无符号整型数的值是payload的真实长度。

![websocket响应](https://images2018.cnblogs.com/blog/1078987/201803/1078987-20180315190640152-1549042973.png)

上图是客户端发送给服务端的数据包，其中PayloadData的长度为二进制：01111110——>十进制：126；如果值是126，则后面2个字节形成的16位无符号整型数的值是payload的真实长度。也就是圈红的十六进制：00C1——>十进制：193 byte。所以PayloadData的真实数据长度是193 bytes；

![websocket帧图](https://images2018.cnblogs.com/blog/1078987/201803/1078987-20180315190647548-379204831.png)

我们再来抓包分析一下服务器到客户端的数据包：

![客户端收到包](https://images2018.cnblogs.com/blog/1078987/201803/1078987-20180315190656357-234229738.png)

可以发现服务器发送给客户端的数据包中第二个字节中MASK位为0，这说明服务器发送的数据帧未经过掩码处理，这个我们从客户端和服务端的数据包截图中也可以发现，客户端的数据被加密处理，而服务端的数据则没有。（如果服务器收到客户端发送的未经掩码处理的数据包，则会自动断开连接；反之，如果客户端收到了服务端发送的经过掩码处理的数据包，也会自动断开连接）。

### websocket包大小限制
理论上websocket包大小没有限制，但是为了高效利用带宽，建议一个websocket包不宜过大。如果一个websocket包过大，可以分片处理，让产生数据和发送数据能够并行进行, 防止整块数据在用户空间的缓存。

### 数据掩码的作用
WebSocket协议中，数据掩码的作用是增强协议的安全性。但数据掩码并不是为了保护数据本身，因为算法本身是公开的，运算也不复杂。除了加密通道本身，似乎没有太多有效的保护通信安全的办法。

那么为什么还要引入掩码计算呢，除了增加计算机器的运算量外似乎并没有太多的收益（这也是不少同学疑惑的点）。

答案还是两个字：安全。但并不是为了防止数据泄密，而是为了防止早期版本的协议中存在的代理缓存污染攻击（proxy cache poisoning attacks）等问题。

掩码的作用将数据包重新编码，这样就避免伪造http包体了（即使伪造了，代理服务器不会识别，不去缓存）。

### 参考
1. [WebSocket协议深入探究](http://www.infoq.com/cn/articles/deep-in-websocket-protocol)
2. [学习WebSocket协议—从顶层到底层的实现原理](https://github.com/abbshr/abbshr.github.io/issues/22#issuecomment-261436452)