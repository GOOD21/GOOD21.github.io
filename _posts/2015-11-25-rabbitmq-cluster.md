---
layout:     post
title:      "RabbitMQ-集群"
date:       2015-11-25
author:     "GOOD21"
header-img: "img/post-bg-unix-linux.jpg"
tags:
    - RabbitMQ
---

### RAM nodes & DISC nodes

区别于元数据（metadata）是存在内存还是在硬盘

在集群中声明Exchange Queue Binding，这类操作要等到所有的节点都完成创建才会返回。如果是RAM节点就要修改内存数据，如果是DISC节点 就要等待写磁盘，节点过多这里的速度就会被大大的拖慢。

RabbitMQ的确也是这样要求的：集群中只要有一个DISC节点 就可以，其它的都可以是RAM节点。节点加入或退出集群一定至少要通知集群中的一个DISC 节点记录这种变动。

其实对于一般业务，并不会频繁的创建、删除、修改这些元数据，所以即使全都是DISC节点 也不会对性能造成太大影响。但是RPC这种服务，会声明匿名排他队列，如果全是DISC的话，影响就比较大。

这个元数据不包括我们publish的message，如果开启了持久化，就算是RAM节点，消息也会妥妥的写在硬盘上。

### 默认集群

集群中，元数据（可以理解为队列结构）是共享的，但是Queue是只存在于一个节点上的。

如果队列q1存在于NodeA，那么当客户端从NodeB获取q1的数据时，RabbitMQ会临时在NodeA、NodeB之间传输数据，把NodeA中的消息取出并经过NodeB发送给客户端。久而久之，无论客户端连接NodeA还是NodeB，出口总在NodeA，就会产生瓶颈。

默认集群的问题在于：如果此时NodeA的节点挂了，NodeB就取不到数据...如果NodeA有持久化的话，必须等待NodeA启动起来，q1才可以继续运作，如果没做持久化，然后就没有然后了。

### 镜像集群（HA方案）

RabbitMQ的HA方案，是根据Policy 来设置一些 队列（这里关注点在queue）匹配的规则，进而设置备份镜像。

###### 三种ha-mode：

1. all：集群里所有节点都对匹配上的queue做备份
2. exactly：需要参数count，集群里至少这些个节点对queue做备份
3. nodes：需要参数nodes，指定节点对queue做镜像

经过测试，all&exactly的master，会平均分配到集群的每一台机器，而nodes则只会在指定的nodes上。

进而会出现一个情况：
如果一个 queue 位于 [A B] mirrored-queue cluster 中（其中 A 为 master），然后你设置了 policy 为 nodes 以使该 queue 将镜像转移到 [C D] 上面。第一步将完成 [A C D] 的转变。一旦 queue 完成了其与 [C D] 的镜像同步动作，之前作为 master 的 A 将会令自身失效。

###### 当某个节点挂了之后：

这里需要注意的是，无论什么节点挂了，跟这个节点连接的客户端（也可能是经过haproxy）也会断，这时候需要在客户端代码里做一次重连机制。一次就行，不然就死循环了。

1、如果挂的是slave

* 系统除了做些许记录外几乎啥都不做：master 仍旧是 master ，客户端不需要采取任何行动，或者被通知 slave 的失效。值得注意的是，slave 的失效可能不会被立即检测出来，而RabbitMQ的flow control 会导致 message 被延迟发送。

2、如果挂的是master

* 某一个 slave 会被提升为新的 master 。被选中作为新的 master 的 slave 通常是看哪个 slave 最老，因为最老的 slave 与前任 master 之间的同步状态应该是最好的。需要注意的是，如果存在没有任何一个 slave 与 master 进行过 [ synchronised ]的情况，那么前任 master 的未被备份的 message 将会丢失。
* 新上任的master 会将所有的consumer都断开（客户端consumer需要重连，否者while-true的话就会耗死单核cpu），并且requeue所有已经投递给客户端但尚未收到 acknowledgement 的 message 。这类 message 中将包括那些客户端已发送过 acknowledgement 进行确认的消息：或者因为 acknowledgement 抵达 master 之前其丢失了，或者因为 acknowledgement 在 master 向 slave 进行广播的时候丢失了。无论是上述哪种情况，新 master 除了将其认为尚未 acknowledged 的消息进行 requeue 外没有更好的处理办法。 **这样就导致了一个消息可能会被执行两遍**。 如果设置了auto_ack（后台也叫noack），即consume到消息便确认，这样不会出现上述情况，所以需要根据具体业务需要合理设置auto_ack。

###### 当新加入了一个slave节点：

这个空节点，不会同步之前master的数据，而是从加入集群这一刻开始，同步新加入的数据。

![cluster1.png](/img/in-post/rabbitmq/cluster1.png)

这样只有当同步之前的消息执行完了之后，这个slave才会和master保持一致。

![cluster2.png](/img/in-post/rabbitmq/cluster2.png)

所以新增一个slave并没有提高队列中旧消息的可用性，但是提高了新消息的可用性。


