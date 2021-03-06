---
title: "接口流量控制"
date: 2017-05-01T21:07:16+08:00
draft: false
---
### 背景
公有云的服务通常是将私有云的服务进行包装，并对外提供服务的，由于业务应用系统的负载能力有限，为了防止非预期的请求对系统压力过大而拖垮业务应用系统，需要对请求流量进行限速。

### 漏斗算法
漏桶(Leaky Bucket)算法思路很简单,水(请求)先进入到漏桶里,漏桶以一定的速度出水(接口有响应速率),当水流入速度过大会直接溢出(访问频率超过接口响应速率),然后就拒绝请求,可以看出漏桶算法能强行限制数据的传输速率。

- 优点

  可以让流量匀速通过，实现简单

- 缺点

  流量始终匀速输出，对于突发特性的流量支持地不好

- 实现

  用一个队列即可搞定，消费者线程匀速取出

### 令牌桶算法

令牌桶算法(Token Bucket)和 Leaky Bucket 效果一样但方向相反的算法,更加容易理解.随着时间流逝,系统会按恒定1/QPS时间间隔(如果QPS=100,则间隔是10ms)往桶里加入Token(想象和漏洞漏水相反,有个水龙头在不断的加水),如果桶已经满了就不再加了.新请求来临时,会各自拿走一个Token,如果没有Token可拿了就阻塞或者拒绝服务.

- 优点

  可以应对突发性的流量

- 缺点

  实现起来不是很容易

- 实现

  下面详细

### 通过redis实现令牌桶算法进行流量控制

#### 流量控制项目

- 单个Ip访问速度限制

  规则: reqs / seconds， 例如: 300 / 60, 表示每分钟最多允许300个请求，也就是平均每秒钟5个请求，但是我们可以允许流量的抖动，允许每5秒内有100个请求，这时，我们可以这样设定： 100 / 5, 两个规则加在一起就能满足两个要求了。

  具体流程：

  一个请求过来，对于每个规则构造key:
  ```java
  key = GLOBAL_PREFIX_FOR_REDIS + PREFIX_RATE_LIMIT + KEY_SPLIT_FLAG + request.getClientIp() + i;
  ```

     判断redis中key对应的列表长度, 如果列表长度小于限制，则通过; 如果大于等于限制，首先判断最早加入列表的元素（time）时间和当前时间差是否大于perSecond， 如果是，则将最早的时间元素从redis中移除，并将当前时间元素加入redis，允许请求通过，而且标记上后续清理过期的时间项目。如果不允许通过，则抛出异常

```java
        // 统计是否封禁
        for (int i = 0; maxRateLimitPerIps != null && i < maxRateLimitPerIps.size(); i++) {
            String maxRateLimitPerIp = maxRateLimitPerIps.get(i);
            String[] rateLimitInArray = maxRateLimitPerIp.split("/");
            int maxLimit = Integer.parseInt(rateLimitInArray[RATE_MAX_LIMITED_INDEX]);
            if (DEFAULT_LIMIT_VALUE.equals(rateLimitInArray[RATE_MAX_LIMITED_INDEX])) {
                break;
            }
            int perSecond = Integer.parseInt(rateLimitInArray[RATE_LIMITED_STAT_SECOND_INDEX]);
            ForbiddenConfigVO forbiddenConfigVO = RateLimitUtil.getForbiddenConfig(rateLimitInArray, nowTime);
            checkLimit(RateLimitUtil.getIpKey(request) + i, maxLimit, perSecond, nowTime, forbiddenConfigVO);
        }


    public static String getIpKey(RequestEntity request) {
        return new StringBuffer(GLOBAL_PREFIX_FOR_REDIS).append(PREFIX_RATE_LIMIT).append(KEY_SPLIT_FLAG).append(request.getClientIp()).append(KEY_SPLIT_FLAG)
                .toString();
    }

```

```java
    public static RateLimitStatResult statRateLimitList(Jedis jedis,
                                                        String listKey, Long nowTime, int perSecond, int maxLimit) {
        StopWatch sw = new StopWatch("statRateLimitList_begin_cost:"); // 增加时间监控日志
        sw.start("statRateLimitList_jedis.lrange_cost:");
        RateLimitStatResult rateLimitStatResult = new RateLimitStatResult();
        List<String> rateList = jedis.lrange(listKey, 0, -1);
        sw.stop();
        sw.start("statRateLimitList_deal_list_cost:");
        if (rateList == null || rateList.isEmpty()) {
            jedis.rpush(listKey, nowTime.toString());
            setListExpireTime(listKey, perSecond, jedis);
            LOGGER.debug("new list, key is " + listKey);
            sw.stop();
            LOGGER.debug(sw.prettyPrint());
            return rateLimitStatResult;
        }
        // 统计的窗口开启时间
        long beginTime = nowTime - perSecond * 1000;
        setListExpireTime(listKey, perSecond, jedis);
        if (rateList.size() < maxLimit) {
            jedis.rpush(listKey, nowTime.toString());
            LOGGER.debug("rateList.size(): " + rateList.size() + ",maxLimit" + maxLimit + ",now is:" + nowTime);
            if (rateList.size() > maxLimit / 2) {
                // 存入queue，清理过期数据
                rateLimitStatResult.setClearList(true);
            }
            sw.stop();
            LOGGER.debug(sw.prettyPrint());
            return rateLimitStatResult;
        }
        if (beginTime > RateLimitUtil.getTime(rateList.get(0))) {
            LOGGER.debug("not new list and compare first node result is delete, key is " + listKey);
            jedis.lrem(listKey, 0, rateList.get(0));
            jedis.rpush(listKey, nowTime.toString());
            rateLimitStatResult.setClearList(true);
            sw.stop();
            LOGGER.debug(sw.prettyPrint());
            // 存入queue，清理过期数据
            return rateLimitStatResult;
        }
        LOGGER.debug("not new list and OverLimited, key is " + listKey);
        rateLimitStatResult.setOverLimited(true);
        sw.stop();
        LOGGER.debug(sw.prettyPrint());
        return rateLimitStatResult;
    }
```

- 每个接口(/uri级别)限速

- 达到限速阈值后，是否封禁，这个可以在配置里配置
　封禁的实现是可以通过map来实现的，value为封禁截止时间

- 黑白名单

- 按账户级别进行限速

### 实现细节探讨
- redis分布式锁怎么防止由于客户端奔溃而死锁

  设置过期时间,对于高并发下，过期时间可能设置过短，对于目前业务来说，这种误差是可以接受的，如果不允许这种误差，要想更好的方案（暂时没想出来）

- redis锁的获取

  构造一个key作为lock key, 然后调用setnx, 如果返回不是1，则认为是已被锁住，需要等待，sleep(1)


