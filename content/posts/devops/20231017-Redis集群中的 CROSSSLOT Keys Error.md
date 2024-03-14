---
title: Redis集群中的 CROSSSLOT Keys Error
slug: redis-cluster-crossslot-err
categories:
  - 开发运维
tags:
  - Redis
  - CROSSSLOT
description: Redis集群环境中，使用HashSlot来决定分为key到那个节点，相同hash的key会存储到相同节点。在进行多键操作时，这些key必须位于同一个节点上，也就是hashslot必须相同，可以通过HASHTAG来设计这些需要多键操作的key。
date: 2023-10-17
updated: 2023-10-17
---

# 场景
Redis单节点没有问题，切换到Redis Cluster后业务上某一功能出错：
```
CROSSSLOT Keys in request don't hash to the same slot
```
出错的代码：
```go
var ctx = context.TODO()  
_, err := uq.queueImpl.rc.TxPipelined(ctx, func(pip redis.Pipeliner) error {  
   cmd := pip.LPush(ctx, uq.key, dest...)  
   if cmd.Err() != nil {  
      return cmd.Err()  
   }  
   return pip.SAdd(context.Background(), uq.setkey, dest...).Err()  
})
```
这段代码的逻辑是向 `list` 中push一条数据，再向一个 `set` 加入数据，两步操作通过 `pipline` 在同一个事务中完成。
# 问题分析
错误的大概意思就是: 请求中跨槽的key没有被hash到相同的hash槽中。通过代码分析，事务中的两次操作的key并不相同，他们没有被hash到同一个hash槽从而出现上述错误。

## 什么是hash槽

[Redis Cluster 规范](https://redis.io/docs/reference/cluster-spec/#overview-of-redis-cluster-main-components)中的 `Key distribution model`（key分布模型）说明如下：

Redis集群中将key的存储空间划分为16384个slot，整个集群可以支持最大16384个主节点，实际上建议不超过1000个节点。每一个主节点又可以处理16384个 hash slot。整个集群将hash slot分布于不同的节点组成高可用集群，单个节点有多个副本以便在节点故障时将重新平衡节点，达到高可用的目的。关于可用性保证，可以看[这里](https://redis.io/docs/reference/cluster-spec/#availability)。

如下的公式用于计算key的hash slot：
```
HASH_SLOT = CRC16(key) mod 16384
```


Redis将**相同hash值的slot分布到同一个node下**，如下图所示：

![hashslot分布模型](/images/devops/20231013164401.png "hashslot分布模型")
图片出自[这里](https://hackernoon.com/resolving-the-crossslot-keys-error-with-redis-cluster-mode-enabled)

可以看出，hash槽(slot)就是一个整数，通过key计算得来，它的作用就是**决定key存储于哪一个节点中**。

## 为什么会出现CROSSSLOT错误

主要原因是应用程序尝试在多个键上运行命令，但操作中涉及的键不在同一个哈希槽中，这会导致集群中不允许的“CROSSSLOT”操作。

比如，使用`Set`的[`SUION`](https://redis.io/commands/sunion/)命令时，如果多个`key`的 `hash slot` 不在集群中的同一个`node`上，则会出现`CROSSSLOT`错误。

前边场景中，在同一个事务中操作多个key，集群环境下必须要保证这些被操作的`key`必须被hash到同一个slot，否则同样会抛出`CROSSSLOT`错误。

Redis这么做的主要原因还是在于避免分布式数据被破坏的风险，而且在同一个事务下或者同一个命令中操作多个跨节点的`key`，会因网络等因素带来性能损耗，所以Redis禁止这么做。如果有这种场景，Redis也提供了解决方案：使用[Hash Tags](https://redis.io/docs/reference/cluster-spec/#hash-tags)。

# 问题解决

那么，什么是 `Hash Tag`?

`Hash Tag`(哈希标签)是一种计算`HASH_SLOT`的特殊处理方式，是一种确保多个键分配在同一个哈希槽中的方法。这样就可以实现在Redis集群中执行多`key`操作。

哈希标签的使用很简单，形式为`{...}`，也就是将 `key` 中的部分字符使用大括号包裹起来，比如 `{mykey}:abc`。Redis在计算`HASHSLOT`时只会计算`{}`中的字符串，而不是整个`key`。

举个例子，原本的两个`key`为`my_key:abc`、`my_key:def`具有不同的`hashslot`，因为默认Redis按照整个`key`计算。现在将其改为 `{my_key}:abc`、`{my_key}:def`，则Redis计算`hashslot`只会考虑`my_key`这个字符串而忽略key的其他部分，所以这两个key会被分配到同一个节点中。但是Redis只会处理第一对`{}`中的字符，例如：

* key `{user1000}.following` 和 `{user1000}.followers` 具有相同的`hashslot`因为计算时使用`user1000`；
* key `foo{}{bar}` 计算hash时使用整个key，因为第一对`{}`中时空字符；
* key `foo{{bar}}zap` 计算hash时使用 `{bar`，因为第一对`{}`中的字符就是`{bar`；
* key `foo{bar}{zap}` 计算hash时使用第一对`{}`中的`bar`；

# 总结

Redis集群环境中，使用 `HashSlot` 来决定分为 `key` 到那个节点，相同 `hash` 的 `key` 会存储到相同节点。在进行多键操作时，这些`key`必须位于同一个节点上，也就是`hashslot`必须相同，可以通过 `HASHTAG` 来设计这些需要多键操作的key。