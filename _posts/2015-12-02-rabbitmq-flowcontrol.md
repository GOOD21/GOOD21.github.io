---
layout:     post
title:      "RabbitMQ-流量控制"
date:       2015-12-02
author:     "GOOD21"
header-img: "img/post-bg-unix-linux.jpg"
tags:
    - RabbitMQ
---

### 关于FlowControl

RabbitMQ 使用了一种基于 credit 的算法来 限制 message 被 publish 的速率 。

Publisher 只有在其从某个 queue 的 全部镜像处收到 credit 之后才被允许继续 publish 。如果存在没能成功发送出 credit 的 slaves ，则将导致 publisher **停止 publish 动作**。

Publisher 会**一直保持停止的状态**，直到所有 slave 都成功发送了 credit 或者直到剩余的 node 都认为某 slave 已经从 cluster 中断开了。

Erlang 会周期性地发送 tick 到所有的 node 上来检测是否出现连接断开。 tick 的间隔时间可以通过配置 net_ticktime 的值来控制。

FlowControl保证了主从的一致性，如果没有了它，会导致master->slave的过程**永远追不完**，也就是说当master挂了之后，slave会丢一部分（未追上的）信息。

### 如何利用FlowControl 找出系统的瓶颈

这是由Simon MacMullen 在2014年4月发布的一篇文章中提到的方法。

[Finding bottlenecks with RabbitMQ 3.3](http://www.rabbitmq.com/blog/2014/04/14/finding-bottlenecks-with-rabbitmq-3-3/)

![flow-q.png](/img/in-post/rabbitmq/flow-q.png)

这个FlowControl 贯穿了整个RabbitMQ的架构，包括connections、channels、queues，他们都有flow这个属性。而消息又是顺序的经过这些组件，所以flow的出现表明了 当前组件的下游限制了自身的发展。

消息经过的顺序：

```
Network
↓
Connection process - AMQP parsing, channel multiplexing
↓
Channel process - routing, security, coordination
↓
Queue process - in-memory messages, persistent queue indexing
↓
Message store - message persistence
```

* A connection is in flow control, but none of its channels are - This means that one or more of the channels is the bottleneck; the server is CPU-bound on something the channel does, probably routing logic. This is most likely to be seen when publishing small transient messages.
* A connection is in flow control, some of its channels are, but none of the queues it is publishing to are - This means that one or more of the queues is the bottleneck; the server is either CPU-bound on accepting messages into the queue or I/O-bound on writing queue indexes to disc. This is most likely to be seen when publishing small persistent messages.
* A connection is in flow control, some of its channels are, and so are some of the queues it is publishing to - This means that the message store is the bottleneck; the server is I/O-bound on writing messages to disc. This is most likely to be seen when publishing larger persistent messages.

