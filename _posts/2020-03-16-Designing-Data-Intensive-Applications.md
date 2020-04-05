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
好处:  
* 高可用性: 当node故障时, 可以从其他node获取数据.  
* 降低数据丢失的风险.  
* 多台机器可以同时处理read request, 增大系统read request的吞吐量.  
* 应用的使用者位于多个地方, 可以减少获取数据的延时.  
  
引入的问题:  
- 对于分布式Blob存储系统, 接口只有Put/Get/Delete，数据Put后不再改变, 保持多个副本的一致性比较简单.  
- 对于分布式sql数据库, 有各种数据Update操作, 保持多个副本的一致性比较难.  
  
保持多个副本的一致性的算法类型:  
* single-leader replication.  
* multi-leader replication.  
* leaderless replication.  
  
### Single-leader Replication  
**Replica**: Each node that stores a copy of the data.  
write/update request, 如何确保所有节点收到整个数据:  
1. 一个节点为leader, 所有写请求都通过这个leader.  
2. 其他节点为follower, leader把这个写请求(作为一条**replication log**)发给所有follower.  
  
leader和所有follower都能处理read request.  
一般分布式数据库/存储系统/消息队列都内置有这种复制的特性.  
  
#### Synchronous vs Asynchronous Replication  
sync: leader收到client的一个写请求, 转发给所有follower, 等到所有follower回复ok后, 再回复client.  
async: leader收到client的一个写请求, 转发给所有follower, 同时更新好数据后就回复client(不等follower的回复).  
semi-sync: leader收到写请求, 转发给所有follower, 等到一定数量(一般是一半)的follower回复ok后, 就回复client.  
  
#### Setting Up New Followers  
加新机器或者代替坏的机器时,需要设置新的follower, 一般通过**snapshot**让follower的**replication logs**跟上leader.  
  
#### Handling Node Outages  
当系统的node下线维护, 或因机器/网络故障不能提供服务时, 我们需要降低这个node对系统可用性的影响.  
+ Follower failure: Catch-up recovery  
  follower的本地disk保存有replication log, 只需要连接leader, 获取后面的log.  
+ Leader failure: Failover(故障切换)  
  从followers中选出新leader, 这个过程一般是自动的(比如Raft).  
  
#### Implementation of Replication Logs  
// Todo: 待补. 这一节讲node把replication logs的操作写入底层的存储引擎, 等看完ch3再来补.  
  
### Problems with Replication Lag  
Leader-base Replication要求所有的write都经过Leader, 再由Leader给followers.假设Leader采用sync的复制方式, 系统的可用性会很差, 所以系统一般采用async/semi-sync的复制方式.  
但是async/semi-sync不可避免的会出现一个问题: **Replication Lag**. 对于系统的用户来说, 会看到**数据的不一致性**: 在某一个时刻, client_1的read request发给Leader, client_2的同一个请求发给某一个(replication lag的)follower, 这2个request返回不同的结果.  
**最终一致性**: 如果一个系统保证所有followers的数据最终都会和leader保持一致, 称这个系统具有"最终一致性"的能力.但这个"最终"有可能是任意长的时间.  
下面是3个比最终一致性强一点的一致性.  
  
#### Reading Your Own Writes  
Read-your-own-write-consistency: 保证用户会看到自己提交的任何更新.  
比如一个场景: 一个用户在一个网站更新自己的个人信息, 然后刷新页面, read-your-own-write保证用户会看到自己刚才更新的信息.  
实现:  
+ 当用户读取的信息是他能修改的, 从leader读. 所以这要求系统判断哪些信息是他能修改的.  
  比如: 用户在网站的个人信息, 一般只有他自己能修改, 所以用户读取自己的个人信息时, 从leader读取.  
+ client记录它最后一次write request的时间戳, 读取信息时从时间戳满足的node中读取.  
  时间戳具体可以是replication log的序列号, 或者system clock(会有误差), 或其他方式.  
  
当然, 在具体实现还需要考虑很多问题, 比如跨设备的场景: 在电脑上write, 在手机上读取信息, 很难用时间戳的方式实现read-your-own-write.  
  
#### Monotonic Reads  
Monotonic Read: 用户发了2个读请求, 保证第2个读请求的结果不会比第1次读请求的结果"更旧".  
比如一个场景: 用户第一次刷新页面, 看到页面的文章有10条评论; Monotonic reads保证用户第2次刷新页面, 不会出现文章只有5条评论的情况(除非删除评论, 但是, 有5条评论->10条评论->删除评论只剩5条, 这时候的5条评论是最新的数据，不是旧数据).  
一种实现:  
+ 用户总是从一个固定的节点处理它的读请求, 比如通过user_id的哈希值确定用户的读请求发往哪个节点.  
  但这种实现的问题是, 当这个node挂了时, 对应的用户的读请求被阻塞/转发到其他node.  
  
#### Consistent Prefix Reads  
...  
  
#### Solutions for Replication Lag  
当使用一个提供"最终一致性"的系统时, client需要考虑: 系统数据在某一个时刻的不一致状态, 会不会对client造成问题? 如果client不能接受这种不一致性:  
* 在client层处理这种不一致状态, 但这样代码会很复杂和dirty, 不建议这样处理.  
* 改用能提供更强的一致性的系统.  
  
### Multi-leader Replication   
#### Use Cases for Multi-Leader Replication  
- Multi-datacenter operation  
  把数据副本放到多个datacenter的好处是, 可以容忍整个datacenter出现故障, 和减少用户访问系统的延时.  
  这时可以采用multi-leader模式, 每一个datacenter一个leader. 与single-leader模式的对比:  
  1.性能上, 一个write request发给它所在的datacenter的leader, leader再异步发给其他datacenter的leaders, 所以client不需要等待跨datacenter的网络延时.  
  2.single-leader模式的leader所在的datacenter故障, 需要重新选举, 但是multi-leader不需要.  
  3.不同datacenter通过公有网络, 而同一个datacenter里面一般是局域网络, 速度更快.  
- Clients with offline operation / Collaborative editing  
  如果一个应用在连接不上网络时照样需要工作, 或者像Google Docs这种允许多个人同时编辑一个doc的应用, 可以考虑multi-leader模式.  
  
#### Handling write conflicts  
Multi-leader replication的最大问题就是会出现写冲突.  
比如, 2个人同时编辑一个wiki页面, user_1把标题A改成B, user_2同时把它改成C, 2个改动都在他们访问的leader修改成功了, 在把改动异步复制给所有replica时, 就检测到写冲突了.  
* 最简单的策略: 避免冲突. 
* 给每一个write request分配一个unique id, 再出现写冲突时, 选取id最大的write request.  
* 系统把写冲突的解决方案留给使用者去写.  
  
#### Multi-Leader Replication Topologies  
在multi-leader replication系统中, 还需要考虑leader之间的复制顺序, 比如:  
- all-to-all topology, 每一个leader收到一个write request, 都会转发给所有leader.  
- circular topology, leader之间以固定的环状方式转发, 比如, leader1 -> leader2 -> leader3.  
- star topology, 类似菊花图, 以一个leader为中心.   
  
### Leaderless Replication  
Leaderless replication模式, 每一个replica都能直接接收client的write request.  
  
#### Writing to the Database When a Node Is Down  
#### Limitations of Quorum Consistency  
#### Sloppy Quorums and Hinted Handoff  
#### Detecting Concurrent Writes  
  
## Ch6.Partitioning  
## Ch7.Transactions  
## Ch8.The trouble with Distributed System  
## Ch9. 
  

















