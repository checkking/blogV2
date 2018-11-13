---
title: "Redis持久化"
date: 2018-11-13T16:56:14+08:00
draft: false
---

### 持久化
* redis持久化包括rdb, aof, rdb是快照的全量备份的方式，aof是增量备份的方式, aof 可以采用aof重写使其进行紧凑。
* redis rdb 采用cow技术，保证父进程同时也允许写操作。
