---
author:
  name: ruifeng
  link: https://blog.luckypeak.xyz
  email: me@luckypeak.xyz
  avatar: /safari-pinned-tab.svg
title: MongoDB 为什么不允许对哈希片键施加唯一性约束？
subtitle: 
date: 2023-10-28
lastmod: 2023-11-05
categories:
  - 数据库
tags:
  - MongoDB
  - 分片
draft: false
---
[片键（shard key）](https://www.mongodb.com/docs/manual/core/sharding-shard-key/)的背后一定有索引的支持。对于哈希片键（hashed shard key），就会在片键对应的字段上建立[哈希索引（hashed index）](https://www.mongodb.com/docs/manual/core/indexes/index-types/index-hashed/#std-label-index-type-hashed)。与通常所说的索引存储字段值不同，哈希索引存储的是对字段进行哈希后的哈希值，而该哈希值就是[哈希分片（hashed sharding）](https://www.mongodb.com/docs/manual/core/hashed-sharding/#footnote-hashvalue)时的片键。

为什么 MongoDB [不允许对哈希片键施加唯一性约束](https://www.mongodb.com/docs/manual/reference/method/sh.shardCollection/#mongodb-method-sh.shardCollection)呢？

这是因为对哈希片键施加唯一性约束没有意义。如上所述，哈希片键基于哈希索引，而对哈希索引，或者说对哈希值施加唯一性约束是无意义的。因为，由于哈希冲突的存在，同一字段上的不同值亦有可能哈希值相同。
