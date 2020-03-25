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
  
**Node**: In a distributed system, each machine or virtual machine running the database software is called a node. Each node uses its CPUs, RAM and Disks independently.  
**Replication**: Keeping a copy of the same data on several different nodes, potentially in different locations.  
**Partitioning**: Splitting a big database into smaller subsets called partitions so that different partitions can be assigned to different nodes.  
  
## Ch5.Replication  
## Ch6.Partitioning  
## Ch7.Transactions  
## Ch8.The trouble with Distributed System  
## Ch9. 
  

