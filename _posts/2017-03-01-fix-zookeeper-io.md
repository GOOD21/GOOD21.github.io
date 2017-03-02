---
layout:     post
title:      "解决Zookeeper占用IO过高的问题"
date:       2017-03-01
author:     "GOOD21"
header-img: "img/post-bg-unix-linux.jpg"
tags:
    - Zookeeper
---

最近因为Kafka集群要接入新的topic，所以简单看看各种压力情况，发现Zookeeper占用IO过高。

`iostat` 如下图：
![1.png](/img/in-post/fix-zookeeper-io/1.png)

`iotop` 如下图：
![2.png](/img/in-post/fix-zookeeper-io/2.png)

对比发现,磁盘IO的**频繁程度比较高**，**吞吐却不是很高**，Zookeeper基本上稳定在400k/s。

##### 为什么Zookeeper会导致磁盘IO高？
`Kafka`的`Consumer`消费时，需要保存`offset`在Zookeeper，而每一次的消费结束对需要对`offset`做修改，对于Zookeeper来说，这是一个事务，所以每一次都要**落在磁盘**上才返回ok。

##### 优化思路
Zookeeper有两种日志，一种是`snapshot`（快照），另一种是`log`（事务日志），出问题的点在于事务日志。

1. 写内存，`/dev/shm`是系统内存划分出的一块区域，默认大小是系统内存的一半，可以通过`df -h`看到。我们可以把事务log写到这里
2. 单独mount一块SSD，这就不说了，是钱的问题。

这里我最终选择的**写内存**的方式。

##### 实施步骤
1. 修改zoo.cfg，把事务日志指向内存文件系统dataLogDir=/dev/shm
2. 重启Zookeeper实例
3. 每个节点重复第一步和第二部

##### 关于自动清理log
网上的文章都建议自己写crontab脚本去定时清理，我倒是觉得Zookeeper提供的配置参数`autopurge.snapRetainCount` 和 `autopurge.purgeInterval` 已经很好用了。

**autopurge.snapRetainCount**：指的是保存的`快照日志`和`事务日志`的分别的文件个数，默认3

**autopurge.purgeInterval**：指的是清理的周期，0表示不开启，1表示每小时清理一次（一开始理解成保存多长时间的日志了，并不是）。

##### 效果
![3.png](/img/in-post/fix-zookeeper-io/3.png)

神清气爽~

