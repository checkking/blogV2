---
title: "为什么nginx默认使用ET模式的epoll"
date: 2017-04-16T21:07:16+08:00
draft: false
---
#### ET&LT
EPOLL事件有两种模型：
1. Level Triggered (LT) 水平触发
- socket接收缓冲区不为空 有数据可读 读事件一直触发
- socket发送缓冲区不满 可以继续写入数据 写事件一直触发

符合思维习惯，epoll_wait返回的事件就是socket的状态

2. Edge Triggered (ET) 边沿触发
- socket的接收缓冲区状态变化时触发读事件，即空的接收缓冲区刚接收到数据时触发读事件
- socket的发送缓冲区状态变化时触发写事件，即满的缓冲区刚空出空间时触发读事件

仅在状态变化时触发事件

#### nginx with ET
使用ET模式，可以便捷的处理EPOLLOUT事件，省去打开与关闭EPOLLOUT的epoll_ctl（EPOLL_CTL_MOD）调用。从而有可能让你的性能得到一定的提升。  例如你需要写出1M的数据，写出到socket 256k时，返回了EAGAIN，ET模式下，当再次返回EPOLLOUT时，继续写出待写出的数据，当没有数据需要写出时，不处理直接略过即可。而LT模式则需要先打开EPOLLOUT，当没有数据需要写出时，再关闭EPOLLOUT（否则会一直会返回EPOLLOUT事件）

当nginx处理大并发大流量的请求时，LT模式会出现较多的epoll_ctl调用用于开关EPOLLOUT，因此ET模式就更合适了
