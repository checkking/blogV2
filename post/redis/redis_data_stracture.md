---
title: "Redis数据结构相关"
date: 2018-11-13T14:18:15+08:00
draft: false
---
### 字符串
* redis字符串是一个二进制安全的，并自带有长度信息。字符串叫做「SDS」，也就是Simple Dynamic String.

```
struct SDS<T> {
  T capacity; // 数组容量
  T len; // 数组长度
  byte flags; // 特殊标识位，不理睬它
  byte[] content; // 数组内容
}
```

* 当字符串长度很短时，使用embstr的格式，当长度超过44字节时，使用raw形式存储。raw形式字符串内容与RedisObject对象的不是在一个连续的内存空间中。

![redis string](https://images-cdn.shimo.im/hf1KV9LmDY4no4Ha/redis_obj.png!thumbnail)

### 字典
* dict有两个hasttable,通常情况下只有一个hashtable有值，但在dict扩容缩容时，要分配新的hashtable。
![redis_dict_ht](https://images-cdn.shimo.im/lU9TCgUPbYstRBnf/redis_dict.png!thumbnail)
* 渐进式rehash: 当redis处于rehash进程中，redis的增删修改等操作都会触发rehash小步搬迁,并且redis还会在定时任务中对词典进行主动搬迁。

### 压缩列表
* zset和hash容器对象在元素个数较少的时候，采用压缩列表(ziplist)进行存储。
* 当set集合容纳的元素都是整数并且元素个数较小时，Redis会使用intset来存储结合元素。
![ziplist](https://images-cdn.shimo.im/tpdBS5DNyRIQ3vQ4/ziplist.png!thumbnail)

### 快速列表
* redis为了节省普通列表因pre,next指针所带来的内存开销和碎片化，采用快速列表的来实现列表。
* quicklist时ziplist和linkedlist的混合体，它将linkedlist按段切分，每一段使用ziplist来紧凑存储，多个ziplist之间使用双向指针串接起来。

```
struct ziplist {
    ...
}
struct ziplist_compressed {
    int32 size;
    byte[] compressed_data;
}
struct quicklistNode {
    quicklistNode* prev;
    quicklistNode* next;
    ziplist* zl; // 指向压缩列表
    int32 size; // ziplist 的字节总数
    int16 count; // ziplist 中的元素数量
    int2 encoding; // 存储形式 2bit，原生字节数组还是 LZF 压缩存储
    ...
}
struct quicklist {
    quicklistNode* head;
    quicklistNode* tail;
    long count; // 元素总数
    int nodes; // ziplist 节点的个数
    int compressDepth; // LZF 算法压缩深度
    ...
}
```

### 跳跃列表
* zset内部实现时基于dict和skiplist
* zset每一个层元素的遍历都是从 kv header 出发。

