---
title: "Redis高可用方案"
date: 2018-11-13T22:42:40+08:00
draft: false
---
### 主从同步
为了避免单点故障，引入主从，提高可用性。当master挂掉，从库来接管。
* 增量同步：主节点将修改命令记录在本地的内存buffer中，然后异步将buffer中的指令同步到从节点。buffer是一个环形数组，如果内存满了，就会从头开始覆盖前面的内容。
* 快照同步：主库bgsave内存数据到磁盘，从库接收完毕，继续增量同步。
* 新的从节点加入，必须先执行一次快照同步，然后再继续进行增量同步。
* 无盘复制

### leader 选举
* redis leader选举有一个方案是sentinel，当主节点挂掉时，自动选择一个最优的从节点切换为主节点。
* 客户端连接集群时，首先连接sentinel, 通过sentinel来查询主节点地址，然后再去连接主节点进行数据交互。
* Sentinel无法保证消息完全不丢失，但可以尽量保证少丢失。它有两个选项可以限制主从延迟过大。
```
min-slaves-to-write 1
min-slaves-max-lag 10
```

### redis分片
* redis分片一个非官方的方案是Codis
![codis](https://images-cdn.shimo.im/wMDjzQOUq9Iy3ktM/codis.png!thumbnail)
* Codis分片是通过槽位来管理的，每个槽位记录hash对应到redis client的映射关系。
* 不同codis实例之间的槽位关系通过zookeeper或etcd来支持的。
* redis增加实例后，会引起codis槽位迁移, 当迁移过程中，有请求的key落在这个槽位上，先强制迁移这个槽位，再访问key。
* 由于槽位迁移会用到getall操作，如果存在大key，会引起服务卡顿，因此不建议大key（不超过1M）。

### redis cluster
* 待补充