---
date: 2022-09-09T20:25:07+08:00  # 创建日期
author: "Rustle Karl"  # 作者

title: "Redis 数据结构及其实战场景"  # 文章标题
url:  "posts/redis/quickstart/scene"  # 设置网页永久链接
tags: [ "redis", "scene" ]  # 标签
categories: [ "Redis 学习笔记" ]  # 分类

toc: true  # 目录
draft: true  # 草稿
---

## 常见的数据结构

String、Hash、List、Set、SortedSet。

## String 字符串类型

redis 中最基本的数据类型，一个 key 对应一个 value。
    
String 类型是二进制安全的，意思是 redis 的 string 可以包含任何数据。如数字，字符串，jpg 图片或者序列化的对象。

**实战场景：**

1. 缓存： 经典使用场景，把常用信息，字符串，图片或者视频等信息放到 redis 中，redis 作为缓存层，mysql 做持久化层，降低 mysql 的读写压力。
2. 计数器：redis 是单线程模型，一个命令执行完才会执行下一个，同时数据可以一步落地到其他的数据源。
3. session：常见方案 spring session + redis 实现 session 共享

## Hash （哈希）

是一个 Mapmap，指值本身又是一种键值对结构，如 value={{field1,value1},......fieldN,valueN}}

![](../assets/images/quickstart/scene/33c69cef9f3a296f.png)

**实战场景：**

1. 缓存： 能直观，相比 string 更节省空间，的维护缓存信息，如用户信息，视频信息等。

##  链表

List 说白了就是链表（redis 使用双端链表实现的 List），是有序的，value 可以重复，可以通过下标取出对应的 value 值，左右两边都能进行插入和删除数据。

![](../assets/images/quickstart/scene/c5124b47150c884f.png)

使用列表的技巧

- lpush+lpop=Stack(栈)
- lpush+rpop=Queue（队列）
- lpush+ltrim=Capped Collection（有限集合）
- lpush+brpop=Message Queue（消息队列）

**实战场景：**

1. timeline：例如微博的时间轴，有人发布微博，用 lpush 加入时间轴，展示新的列表信息。

## Set 集合

集合类型也是用来保存多个字符串的元素，但和列表不同的是集合中：

1. 不允许有重复的元素。
2. 集合中的元素是无序的，不能通过索引下标获取元素。
3. 支持集合间的操作，可以取多个集合取交集、并集、差集。

![](../assets/images/quickstart/scene/40760ec93c9ed359.png)

**实战场景;**

1. 标签(tag)，给用户添加标签，或者用户给消息添加标签，这样有同一标签或者类似标签的可以给推荐关注的事或者关注的人。
2. 点赞，或点踩，收藏等，可以放到 set 中实现。

## zset  有序集合

有序集合和集合有着必然的联系，保留了集合不能有重复成员的特性，区别是，有序集合中的元素是可以排序的，它给每个元素设置一个分数，作为排序的依据。

有序集合中的元素不可以重复，但是 score 分数 可以重复。

![](../assets/images/quickstart/scene/4a329e01150906a0.png)

**实战场景：**

1. 排行榜：有序集合经典使用场景。例如小说视频等网站需要对用户上传的小说视频做排行榜，榜单可以按照用户关注数，更新时间，字数等打分，做排行。
