# 概述

## 目标

实现 Raft协议, a replicated state machine protocol

## 如何通过复制实现容错？

- `replicated service`通过在多个副本服务器上存储其状态（即数据）的完整副本来实现容错
- `replicated service`允许服务继续运行，即使其中一些服务器遇到故障（崩溃或网络故障）
- 存在的问题是，故障可能导致`service`持有不同的数据副本

## RAFT简介

- Raft将客户端请求组织成一个序列，称为日志，并确保所有副本服务器看到相同的日志。每个副本都按日志顺序执行客户端请求，将其应用于其本地的副本数据。由于所有副本都看到相同的日志内容，都按相同的顺序执行相同的请求，因此最终会具有一致的数据。
- 如果有服务宕机然后恢复，Raft会负责将其log更新到最新状态
- 只要大多数服务器还活着，并且可以相互通信，Raft就会继续运行。
- 如果没有这样的大多数，Raft将通知，等到大多数恢复通信，它会从停止的地方继续

## 需要做什么

- 实现Raft作为一个Go对象类型，带有相关的方法，用作较大服务中的模块。
- 一组Raft实例使用RPC相互通信以维护`log`
- Your Raft interface will support an indefinite sequence of numbered commands, also called log entries.  The entries are numbered with index numbers.  The log entry with a given index will eventually be committed.  At that point, your Raft should send the log entry to the larger service for it to execute.

- 代码设计应该和这个一致[extended Raft paper](https://pdos.csail.mit.edu/6.824/papers/raft-extended.pdf)，尤其注意图2，你将会实现这篇论文的绝大多数设计，包括持久化状态，以及节点失败重启后读取这些状态，但是不需要实现第六节（集群成员改变）
- 额外读物，可以看看这些算法：Paxos, Chubby, Paxos Made Live, Spanner, Zookeeper, Harp, Viewstamped Replication, [Bolosky et al.](http://static.usenix.org/event/nsdi11/tech/full_papers/Bolosky.pdf)

