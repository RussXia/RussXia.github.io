---
layout: blog
title: "Jedis的主要设计与实现"
catalog: true
tag: [Java,2021,Redis]
---
# Jedis的主要设计与实现

> github地址: https://github.com/redis/jedis

Jedis是Redis的Java客户端的一种实现，支持单机版、Redis Cluster、Redis Sentinel等三种模式，以及支持分片的ShardedJedis。

 Jedis实例有3种请求模式，Pipeline，Transaction和Client。

Jedis的源码还是比较清晰、简洁的，下面是关于Jedis的一些分享。

## Jedis的类结构

![e72120b1f6daf4a951e75c05b9191a0f](https://raw.githubusercontent.com/RussXia/RussXia.github.io/master/_pic/e72120b1f6daf4a951e75c05b9191a0f.png)

Jedis继承自BinaryJedis，BinaryJedis持有一个Client，Client继承自Connection，Connection中持有一个Socket实例和RedisOutputStream、RedisInputStream两个流。

## Jedis如何通过JedisPool管理连接





## Jedis对Redis协议的实现



+ [redis官方协议说明](https://redis.io/topics/protocol)

+ [redis中文版协议说明](http://redisdoc.com/topic/protocol.html)



## JedisCluster和JedisSentinel





## ShardedJedis的实现







## Jedis、Lettuce和Redisson框架的各自特点

