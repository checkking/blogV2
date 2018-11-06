---
title: "阅读长链接服务架构总结"
date: 2018-11-04T21:45:17+08:00
draft: false
---
### 背景
目前项目中用到了公司的长链接服务，抽空研究了一下公司长链接服务源码。
如果没有长链接服务，一些消息推送等功能只能用短链接轮询，或者长链接轮询的方式。长链接能够很好的解决短轮询和长轮询的一些不足。

### 整体架构
[!架构图](https://images-cdn.shimo.im/L4vecWHirMESFFXB/grontier_arch.png!thumbnail)

### 数据流交互
[!数据流交互](https://images-cdn.shimo.im/UaNx0YyBVl0smmpR/grontier2.png!thumbnail)

1. 客户端传送fpid, aid, device_id, access_key等参数与grontier建立websocket连接， websocket 的subprotocol为pbbp2。
2. grontier维护webscoket连接池, 维护在内存中。
3. grontier向backbone注册映射, backbone将注册关系存入redis中。
4. 客户端发送业务消息到grontier，grontier根据路由service, method参数进行路由调用backservice,
5. backservice处理业务逻辑，将下行消息通过调用backbone接口返回，backbone将下行消息返回给grontier, 最终由grontier 返回给客户端。

### 源码分析
grontier/main.go
主要是启动两个server, 一个是thrift server, 一个是websocket server， 端口不一样。
其中thrift server对是用于与backbone和backservice交互的, websocket server是与客户端交互的。
rontier/thrift.go
grontierThriftServer主要有3个函数，一个Start, 一个Close和handle。
Start是启动一个tcp监听端口，等待连接，如果有链接请求来了，则起一个协程调用handle处理这个链接。
handle 函数首先创建一个grontierServerhandler h，然后调用h.Serve，并传入BufferReader r, BufferWriter w。
在grontierServerHandler中，Serve函数循环从r中读取数据包，首先读取MessageHeader，包括messageType (int32)，name(string)，seq(int32)。其中name目前支持"PushByUUID" , messageType只支持CALL 。 读取正确的messageHeader后读取参数grontierPushByUUIDArgs。
读取完整的请求后，向request channel 塞入一条记录。 Serve函数还启动了一个协程，遍历channel 中的请求(request)， 调用grontierHandler的PushByUUID函数，也就是调用grontierClient的PushByUUID接口（是业务应用调用grontier对外提供的接口吗？还是backbone调用grontier?）
Close是关闭tcp链接， 是在服务被shutdown的时候调用的。

grontier/grontier.go
grontierWebsocketServer的结构体定义如下：

```go
type grontierWebsocketServer struct {
   mu        sync.Mutex
   listeners []net.Listener

   upgrader *websocket.Upgrader
}

```

其中 upgrader是websocket.Upgrader，用于将http提升为websocket。 grontierWebsocketServer的Start函数会用相同的端口起两个socket进行监听，一个是unixsocket，一个是tcp socket。
在grontierWebsocketServer的ListenAndServe函数中，创建一个ServeMux，也就是http的路由器。其中有一个最主要的：

```go
mux.HandleFunc("/ws/v2", s.serveV2)
```
是用于接收客户端消息的。
接下来看一下serveV2方法：
首先获得请求参数，包括:deviceid, appid, productid。
如果app对应的authPSM不为空，则获得对应的psm client，调用client.Auth方法。接着调用upgrader的Upgrade方法，提升http为websocket。其中Upgrade就是建立websocket的过程， 并返回一个connection。
拿到websocket的connection后，执行ConnHub.Run(connection)。在这个方法中，首先执行ConnHub.Register(connection)方法，将连接信息注册到backbone。然后执行Connection.ReadLoop():
ReadLoop执行for循环，并在for循环中执行：

```go
err := c.wsr.Serve(c.websocketCallBack)
if c.IsClosed() {
   return
}
ne, ok := err.(net.Error)
if ok && ne.Timeout() && c.wsr.Reentrant() && time.Since(c.GetAccessTime()) < c.PingInterval(2) { // if timeout, ping it
   atomic.AddUint64(&st.SndPing, 1)
   atomic.AddUint64(&c.st.SndPing, 1)
   if err := c.WriteMessage(websocket.PingMessage, nil); err != nil {
      log.XError(c.String(), "write ping err", err.Error())
      return
   }
   pongTimeout := c.PingInterval(2) - time.Since(c.GetAccessTime())
   if pongTimeout < 10*time.Second {
      pongTimeout = 10 * time.Second
   }
   c.conn.SetReadDeadline(time.Now().Add(pongTimeout))
   continue
}
atomic.AddUint64(&st.ReadErr, 1)
log.XError(c.String(), "read err:", err.Error())
c.ForceClose()
return

```
如果是超时，则发送ping消息给客户端。如果是其他错误，则强行关闭连接。
在websocket的Serve方法中，循环读取客户端发送过来的消息：

```go
for {
   r.reset()
   p, err := r.next(2)
   if err != nil {
      return err
   }
   final := p[0]&finalBit != 0
   opcode := MessageType(p[0] & 0xf)
   mask := p[1]&maskBit != 0
   payloadLen := int(p[1] & 0x7f)
   if rsv := p[0] & (rsv1Bit | rsv2Bit | rsv3Bit); rsv != 0 {
      return errRsvBit
   }

   switch opcode {
   case CloseMessage, PingMessage, PongMessage:
      if payloadLen > MaxControlFramePayloadSize {
         return errPayloadLen
      }
      if !final {
         return errFinFlag
      }
   case BinaryMessage, TextMessage:
   default:
      return errOpcode
   }
//.......
  if err := cb(opcode, payload); err != nil {
     return err
  }
}

```
在Connection的websocketCallBack中，对于PING消息，立马回复PONG消息，对于CLOSE消息，执行Close，对于BinaryMessage，则执行SendMessage2Backservice给Backservice 发rpc请求(SendMessage), 并将Backservice的返回消息发送给前端。
至此，grontier的主干流程梳理清楚了。接着梳理一下其他的流程。


### backbone模块
backbone是一个rpc服务，对外提供的接口包括：QueryOnline, PushByUUID，Push，RegisterDevice,  UnregisterDevice。

1. RegisterDevice:
主要是构造如下结构体：
```go
f := storage.grontierConnInfo{ProductID: r.ProductID, AppID: r.AppID,
   UserID: r.UserID, DeviceID: r.DeviceID, ClientVersion: r.ClientVersion}
```
然后执行 f.save将结果存储在redis中。

2. UnregisterDevice：
将connection信心从redis中删除.

3. QueryOnline:
QueryOnline的用途是根据grontier模块发送过来的参数请求，查询存在redis中的grontier connection。
4.  PushByUUID:
  首先根据uuid列表，拿到设备地址，然后遍历uuid和 grontier的addr的map，起协程调用PushByUUID2grontier。
```go
for sa, uuids := range addr2uuids {
   req.UUIDs = uuids
   wg.Add(1)
   go func(addr string, req backbone.PushByUUIDReq) {
      defer wg.Done()
      ret := PushByUUID2grontier(ctx, addr, req)
      mu.Lock()
      for k, v := range ret {
         resp.Results[k] = v
      }
      mu.Unlock()
   }(sa.String(), req)
}
```
其中PushByUUID2grontier，首先从内存的map中拿到​grontierClient​，如果内存map中没有，则调用​DialerWithRetry​拿到，并存入内存中 。


5. Push

做的事情跟PushByUUID一样。

### 为什么需要backbone
backbone存储了uuid->grontier的一个路由信息，加入没有backbone，backservice需要向客户端发消息，需要指定链接在哪个grontier中，实际上这个信息存储在backbone中，backservice只需要将消息发送给backbone，由backbone路由到正确的grontier。 grontier存储了uuid->websocketconnection信息。

### Backservice发现
grontier的conf写在etcd，grontier动态加载conf，可以发现有新的backservice加入，新的backservice的地址再通过consul获取

### 平滑升级
对Frontier分成两组：一组在线、一组热备。升级时，先升级热备，然后通过unixsocket的软链把热备切换到在线，对新连接生效。

### Nginx作用

1. 负载均衡，通过app_id将连接发送到不同的cluster

2. frontier和Nginx混布，使用unixsocket通信，规避了端口号不够的问题
s
