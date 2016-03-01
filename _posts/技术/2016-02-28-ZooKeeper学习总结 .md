---
layout: post
title: ZooKeeper学习总结
category: 技术
tags: 
keywords: ZooKeeper学习总结
description: ZooKeeper学习总结
---

#ZooKeeper 学习总结
####*参阅书籍：<<实战Hadoop--开启通向云计算的捷径>>*

##ZooKeeper 定义
- ZooKeeper 是一个高效、可靠的协同工作系统，它是Google的Chubby的开源实现。
    - ZooKeeper 用来协调**分布式应用**上的各种服务。
    - ZooKeeper 提供了一套完整的解决方案来简单高效的去处理**部分失败**的问题。
    - 利用 ZooKeeper 可以构建一个有效防止**单点失效**以及处理**负载均衡**的分布式应用系统。
    - ZooKeeper 是由一组 ZooKeeper 服务器构成的系统。

***部分失败:***在分布式应用中可能遇到这样的问题，当消息在网络中的两个节点间传输时，若传输过程中发生了某种故障（可能是网络错误的问题），发送方便无法得知接收方是否得到了完整的传输信息，为了确认消息是否准确到达，发送方必须再次向接收方进行询问，否则发送发无法准确地知晓操作是否成功。这就是部分失败。

##ZooKeeper 的工作原理
- **ZooKeeper 服务的启动与工作过程**
    - ZooKeeper 服务启动后会从配置文件中所设置的服务器中选择一台作为**“领导者”(Leader)**,其余的机器便成为**“跟随者”(Follower)**,当且仅当一半或一半以上的“跟随者”的状态和“领导者”的状态同步之后，才代表“领导者”的选举过程完成了。此过程正确无误地结束之后，ZooKeeper的服务也就开启了。
    - 在整个ZooKeeper系统运行的过程中如果“领导者”出现问题失去了响应，那么原有的“跟随者”将重新选出一个新的“领导者”来完成整个系统的协调工作。
![ZooKeeper工作原理图](https://zookeeper.apache.org/doc/trunk/images/zkservice.jpg)

- **ZooKeeper 的数据结构与组成**
    - ZooKepper 的结构类似于树，树中的节点称为**Znode**,每个节点都可以用来存储数据。
    - 可以给每个节点都添加一个**"监控"(Watcher)**,当节点的状态发生改变时，用“监控”触发某个事件。
    - 在整个ZooKeeper系统中一定存在**根节点("/"节点)**，并且所有的其他节点都必须创建在"/"节点之下。
    - ZooKeeper 中每个节点都有一个与其相关联的**ACL(Access Control List, 访问控制列表)**，每个节点能存储的数据大小限定在**1MB**以内。
    - ZooKeeper 中节点分为**短暂性**节点和**持久性**节点。
![Zookeeper目录结构](https://zookeeper.apache.org/doc/trunk/images/zknamespace.jpg)

- **原子广播**
    客户端所有的写请求都被转发给“领导者”，“领导者”将收到的请求通过广播的形式发送给所有的“跟随者”，当超过半数的“跟随者”修改数据并持久化后，“领导者”才会提交这个更新，这个过程的执行要么全部成功，要么全部失败。
