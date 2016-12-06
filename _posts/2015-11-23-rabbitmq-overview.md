---
layout:     post
title:      "RabbitMQ-概览"
date:       2015-11-23
author:     "GOOD21"
header-img: "img/post-bg-unix-linux.jpg"
tags:
    - RabbitMQ
---

### 简介
> RabbitMQ is a messaging broker - an intermediary for messaging. It gives your applications a common platform to send and receive messages, and your messages a safe place to live until received.

消息代理、消息中间件

###### Reliability

RabbitMQ offers a variety of features to let you trade off performance with reliability, including persistence, delivery acknowledgements, publisher confirms, and high availability.

###### Large Community

There's a large community around RabbitMQ, producing all sorts of clients, plugins, guides, etc. Join our mailing list to get involved!

github [https://github.com/rabbitmq/](https://github.com/rabbitmq/)

google group  [https://groups.google.com/forum/#!forum/rabbitmq-users](https://groups.google.com/forum/#!forum/rabbitmq-users)

### 系统架构

![overview1.jpg](/img/in-post/rabbitmq/overview1.jpg)

###### Producer
消息生产者，数据发送方。发送的消息有两部分：payload和label。payload顾名思义就是传输的数据。label是Exchange的名字或者说是一个tag，它描述了payload，而且RabbitMQ也是通过这个label来决定把这个Message发给哪个Consumer。

###### Exchange
交换机。从架构图可以看出，Procuder Publish的Message进入了Exchange。然后Exchange通过“routing keys”， RabbitMQ会找到应该把这个Message放到哪个queue里。queue也是通过这个routing keys来做的绑定。

###### Broker Server
一种传输服务。他的角色就是维护一条从Producer到Consumer的路线，保证数据能够按照指定的方式进行传输。其中包含了exchange 和 queue。

###### Consumer
消息消费者，数据接收方。在接收的Message中，只有payload，label已经被删掉了。对于Consumer来说，它是不知道谁发送的这个信息的。

### 消息生命周期

###### 创建队列

关于这个事儿纠结了好久，到底是由Consumer创建，还是由Producer创建？

如果queue不存在，当然Consumer不会得到任何的Message；但是如果queue不存在，那么Producer Publish的Message会被丢弃。

**所以，还是为了数据不丢失，我们的做法是Consumer和Producer都创建queue。**

###### Producer 发送消息

Producer投递消息到queue都是经由Exchange完成的。消息发送到RabbitMQ都会携带一个routing key（也可能是空key），通过routing key把queue绑定到exchange上。

RabbitMQ会根据绑定匹配routing key，如果匹配成功消息会转发到指定queue，如果没有匹配到queue消息就会丢弃。

###### Consumer 处理消息

Consumer同样需要通过指定的queue拿到对应的消息（当然也可以自己创建&绑定）。

这里需要注意的是有一个ack机制，只有发送了ack，Broker才认为这个消息被处理了，才会把消息删除。我们可以显示的在程序中去ack，也可以自动的ack。

### 消息持久化

按常理，应该是队列持久化就可以满足我们的需求了。

但是RabbitMQ本着“皮之不存毛将焉附”的原则，需要满足三点：

1. Exchange 设置为Durable
2. Queue 设置为Durable
3. publish时 参数 delivery_mode = 2

当满足前两点的时候，如果没设置delivery_mode，真正的消息依旧是存在内存里的。

### 消息确认机制（ack）
凡是消息队列（无论kafka、RabbitMQ或其他），都有这么几种可能的：

1. At most once 消息可能会丢，但绝不会重复传输。在RabbitMQ里对应的是auto_ack＝ture，默认接到消息即确认。
2. At least one 消息绝不会丢，但可能会重复传输。在RabbitMQ里，处理消息完成之后，手动调用ack方法。
3. Exactly once 每条消息肯定会被传输一次且仅传输一次，很多时候这是用户所想要的。这个需要第三方工具来做，思路无非就是用锁保证事务性。

### 权限管理

RabbitMQ的用户角色分类：none、management、policymaker、monitoring、administrator


###### none
不能访问 management plugin

###### management
用户可以通过AMQP做的任何事外加：

* 列出自己可以通过AMQP登入的virtual hosts 
* 查看自己的virtual hosts中的queues, exchanges 和 bindings
* 查看和关闭自己的channels 和 connections
* 查看有关自己的virtual hosts的“全局”的统计信息，包含其他用户在这些virtual hosts中的活动。

###### policymaker
management可以做的任何事外加：

* 查看、创建和删除自己的virtual hosts所属的policies和parameters

###### monitoring
policymaker和monitoring可以做的任何事外加:

* 创建和删除virtual hosts
* 查看、创建和删除users
* 查看创建和删除permissions
* 关闭其他用户的connections

### Vhost

Vhost是AMQP的一个基础概念，连接到RabbitMQ默认就有一个名为"/"的Vhost可用，本地调试的时候可以直接使用这个默认的Vhost。 

出于安全和可移植性的考虑，一个vhost内的exchange不能绑定到其他的vhost。

可以按照业务功能组来规划Vhost，在集群环境中只要在某个节点创建Vhost就会在整个集群内的节点都创建该Vhost。

VHost和权限都不能通过AMQP协议创建，在RabbitMQ中都是使用rabbitmqctl进行创建&管理的。


