---
title: UUID那些事儿（下）
description:
toc: true
authors: []
tags: [uuid]
categories: [杂谈]
series: []
date: 2017-12-10T20:24:04+08:00
lastmod: 2017-12-10T20:24:04+08:00
featuredVideo:
featuredImage:
draft: false
---

唯一 ID 在业界的更多实现方法。

<!--more-->

## 业务属性

- 滴滴：时间+起点编号+车牌号
- 淘宝订单：时间戳+用户ID
- 其他电商：时间戳+下单渠道+用户ID，有的会加上订单第一个商品的ID。

举例roll业务场景--用户参与roll房间记录
时间+roll房间id+用户id

## 数据库方案-Flicker的解决方案

| 自增id | 调用方ip     |
| :----- | :----------- |
| 5      | 192.168.2.13 |
| 3      | 192.168.2.15 |
| 6      | 192.168.3.15 |

## Redis分布式方案

## 推荐

[美团leaf](https://tech.meituan.com/MT_Leaf.html)
是现有大部分方案的优化方案

## 其他

[shardingjdbc 本身就是中间件:-D](http://shardingjdbc.io/docs/02-guide/key-generator/)

## 参考

[分布式架构系统生成全局唯一序列号的一个思路](https://mp.weixin.qq.com/s?__biz=MjM5MDI3MjA5MQ==&mid=2697266651&idx=2&sn=77a5b0d4cabcbb00fafeb6a409b93cd7&scene=21#wechat_redirect)
[flickr方案原版(科学上网)](http://code.flickr.net/2010/02/08/ticket-servers-distributed-unique-primary-keys-on-the-cheap/)
[flickr方案网络版](http://chuzhiyan.com/2016/08/22/票务服务：廉价的分布式唯一主键/)