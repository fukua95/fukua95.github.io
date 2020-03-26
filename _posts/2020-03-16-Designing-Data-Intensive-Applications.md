---
layout: post
title: "Designing Data Intensive Applications"
author: fukua95
tags: ["distributed system"]
category: system
key: 100006
---

# Part1 Foundations of Data System  
todo.  
  
# Part2 Distributed System  
为什么需要分布式系统:  
**Scalability**: 数据容量,读写负载等超过一台机器的能力.  
**Fault tolerance/high availability**: 应用要求当若干台机器/网络/整个数据中心故障时,仍然可以提供服务.  
**Latency**: 假设应用的使用者遍布全球,在不同地区部署系统,能减少网络延时.  
  
在分布式系统中,**Node**特指: In a distributed system, each machine or virtual machine running the database software is called a node. Each node uses its CPUs, RAM and Disks independently.  
  
## Ch5.Replication  
Replication: 复制,把数据的多个副本放到多台机器上,这些机器通过网络连接.  
- 对于分布式Blob存储系统, 接口只有Put/Get/Delete，数据Put后不再改变, 保持多个副本的一致性比较简单.  
- 对于分布式数据库系统, 会有各种数据Update操作, 保持多个副本的一致性比较难.  
  
保持多个副本的一致性的算法可以分类:  
* single-leader replication.  
* multi-leader replication.  
* leaderless replication.  
  
### Single-leader replication  
**Replica**: Each node that stores a copy of the data.  
write/update request, 如何确保所有节点收到整个数据:  
1. 一个节点为leader, 所有写请求都通过这个leader.  
2. 其他节点为follower, leader把这个写请求(作为一条**replication log**)发给所有follower.  
leader和所有follower都能处理read request.  
一般分布式数据库/存储系统/消息队列都内置有这种复制的特性.  
  
#### Synchronous vs Asynchronous Replication  

  
### Multi-leader replication  
  
### Leaderless replication  
  

## Ch6.Partitioning  
## Ch7.Transactions  
## Ch8.The trouble with Distributed System  
## Ch9. 
  

