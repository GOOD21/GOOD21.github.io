---
layout:     post
title:      "RabbitMQ-功能"
date:       2015-11-24
author:     "GOOD21"
header-img: "img/post-bg-unix-linux.jpg"
tags:
    - RabbitMQ
---

### Exchange Type

要想知道RabbitMQ能满足哪些需求功能，首先需要了解Exchange的相关类型，最常用的是Direct、Fanout、Topics

性能排序：fanout > direct > topic。比例大约为11：10：6

###### Direct Exchange

![direct.jpg](/img/in-post/rabbitmq/direct.jpg)

Direct类型处理路由key，需要将一个队列通过key绑定到交换机上，要求该消息与一个特定的路由键完全匹配。

###### Fanout Exchange

![fanout.jpg](/img/in-post/rabbitmq/fanout.jpg)

Fanout 类型不处理路由键。你只需要简单的将队列绑定到交换机上。一个发送到交换机的消息都会被转发到与该交换机绑定的所有队列上。很像子网广播，每台子网内的主机都获得了一份复制的消息。Fanout交换机转发消息是最快的。

###### Topic Exchange

![topic.jpg](/img/in-post/rabbitmq/topic.jpg)

Topic 类型将路由键和某模式进行匹配。此时队列需要绑定要一个模式上。符号`#`匹配一个或多个词，符号`*`匹配不多不少一个词。因此`abc.#`能够匹配到“abc.a.xx”，但是`abc.*` 只会匹配到“abc.a”。

###### Headers exchange

这个类型的exchange使用的比较少，它也是忽略routingKey的一种路由方式。是使用Headers来匹配的。Headers是一个键值对，可以定义成Hashtable。

发送者在发送的时候定义一些键值对，接收者也可以再绑定时候传入一些键值对，两者匹配的话，则对应的队列就可以收到消息。

匹配有两种方式all和any。这两种方式是在接收端必须要用键值"x-mactch"来定义。all代表定义的多个键值对都要满足，而any则代码只要满足一个就可以了。

fanout，direct，topic exchange的routingKey都需要要字符串形式的，而headers exchange则没有这个要求，因为键值对的值可以是任何类型。


###### MORE
网上关于Exchange的plugin很多，不局限于这几种。

### 功能、应用场景

###### Working Queue

![workingqueue.png](/img/in-post/rabbitmq/workingqueue.png)

这个模式就是最简单的异步队列模式.

值得一提的是，RabbitMQ提供了一种公平分发的设置。

设想一下这个场景：有两个工作者，第奇数个消息对应的任务都很耗时，第偶数个消息对应的任务都很快就能执行完。这样的话其中有个工作者就会一直都很繁忙，另外一个工作者几乎不做任务。RabbitMQ不会去对这种现象做任何处理，依然均匀的去推送消息。

通过AMQPChannel::setPrefetchCount 来设置prefetch count的数量，下图设置为1，表示RabbitMQ同一时间发给消费者的消息不超过一条。这样就能保证消费者在处理完某个任务，并发送确认信息后，RabbitMQ才会向它推送新的消息，在此之间若是有新的消息话，将会被推送到其它消费者，若所有的消费者都在处理任务，那么就会等待。

![prefetch-count.png](/img/in-post/rabbitmq/prefetch-count.png)

从php.net中还会发现AMQPChannel::setPrefetchSize 方法，是用来设置size的。然而文档中说，count 和 size 只能同时生效一种。

另外，如果设置了auto_ack，那么AMQPChannel::setPrefetchSize 和 AMQPChannel::setPrefetchCount 是不生效的。

###### Publish/Subscribe

![pubsub2.png](/img/in-post/rabbitmq/pubsub2.png)

看到这个模型，果断使用上面说的 Fanout Exchange。

pub/sub 模型的实现方式有很多种，Kafka的方式是让Broker更轻，在Consumer层做group；RabbitMQ的方式是在Broker中的Exchange 分发给不同的queue，这也就让Broker更重，影响系统吞吐量。

###### Routing & Topics

![routing1.png](/img/in-post/rabbitmq/routing1.png)

通过上图可以看出来，利用的就是direct类型。

![routing2.png](/img/in-post/rabbitmq/routing2.png)

如果是同一个key的话，等同于fanout

![topics.png](/img/in-post/rabbitmq/topics.png)

topics类型，就是多了一个匹配功能

###### RPC(sync queue)

![rpc.png](/img/in-post/rabbitmq/rpc.png)

1. 当客户端启动的时候，它创建一个排他的回调队列。
2. 在RPC请求中，客户端发送带有两个属性的消息：一个是设置回调队列的 reply_to 属性，另一个是设置唯一值的 correlation_id 属性。
3. 将请求发送到一个 rpc_queue 队列中。
4. Server等待请求发送到这个队列中来。当请求出现的时候，它执行他的工作并且将带有执行结果的消息发送给reply_to字段指定的队列。
5. 客户端等待回调队列里的数据。当有消息出现的时候，它会检查correlation_id属性。如果此属性的值与请求匹配，将它返回给应用。

