---
layout:     post
title:      "RabbitMQ-PHP客户端"
date:       2015-12-01
author:     "GOOD21"
header-img: "img/post-bg-unix-linux.jpg"
tags:
    - RabbitMQ
---

### github

[https://github.com/GOOD21/php-rabbitmq](https://github.com/GOOD21/php-rabbitmq)

集成了异步队列、同步队列功能，提供监控API的封装。

### RPC（sync queue）

*异步队列比较简单，这里只说如何实现rpc。*

流程图参照下图：

![rpc.png](/img/in-post/rabbitmq/rpc.png)

1. 当客户端启动的时候，它创建一个排他的回调队列。
2. 在RPC请求中，客户端发送带有两个属性的消息：一个是设置回调队列的 reply_to 属性，另一个是设置唯一值的 correlation_id 属性。
3. 将请求发送到一个 rpc_queue 队列中。
4. Server等待请求发送到这个队列中来。当请求出现的时候，它执行他的工作并且将带有执行结果的消息发送给reply_to字段指定的队列。
5. 客户端等待回调队列里的数据。当有消息出现的时候，它会检查correlation_id属性。如果此属性的值与请求匹配，将它返回给应用。

**所以，在Client（这里我们叫Client，也就是Producer）发送消息的时候，需要做三个事儿：**

第一，声明一个匿名排他队列，并作为Consumer等待接收返回结果：

```php
//匿名排他队列
$this->exclusive = new RabbitMQConsumer(
    self::RPC_SERVER_TYPE, 
    self::RPC_SERVER_TYPE, 
    null, 
    array($this, 'onResponse'), 
    null, 
    null, 
    AMQP_EXCLUSIVE
);  

// ....do something produce

$this->exclusive->wait();
```

第二，发送需要入队的消息，需要带着correlation_id和reply_to参数：

```php
$this->correlation_id = md5(uniqid() . mt_rand(1, 99999)); //生成一个随机的uuid
$producer = new RabbitMQProducer(self::RPC_SERVER_TYPE, self::RPC_SERVER_TYPE, $this->queue);
$r = $producer->produce($message, false, $this->correlation_id, $this->callback_queue);
```

第三，当在第一步接收到了Consumer的返回值之后，断开连接。

```php
//断开连接
$this->exclusive->disconnect();
```

**同时，Server（这里我们叫Server，也就是Consumer）也需要做两件事：**

第一，消费Client端发送过来的消息：

```php
$this->worker = new RabbitMQConsumer(
    RabbitMQRPCClient::RPC_SERVER_TYPE,
    RabbitMQRPCClient::RPC_SERVER_TYPE,
    $queue,
    array($this, 'RabbitMQRPCCallBack')
);
```

第二，在Callback里，实现对Client端声明的匿名排他队列的入队消息，即返回值。这里需要从回调方法的第二个参数中，获得相同的channel，再去发送消息，因为exclusive队列只有相同的连接才可以produce和consume。

```php
public function RabbitMQRPCCallBack ($envelope, $queue) {
    $message = $this->receiver->handleMessage($envelope->getBody());

    $ex = new AMQPExchange($queue->getChannel());
    $ex->publish(
        $message,
        $envelope->getReplyTo(),
        AMQP_NOPARAM,
        array(
            'delivery_mode'=>1,
            'correlation_id'=>$envelope->getCorrelationId(),
            'reply_to'=>$envelope->getReplyTo()
        )
    );

    if(!$this->receiver->getAutoAck()) {
        //至少执行一次，处理完消息再确认
        $queue->ack($envelope->getDeliveryTag());
    }
}
```

***Client端如果不断开exclusive队列的连接会怎么样？***

*Exclusive (used by only one connection and the queue will be deleted when that connection closes)*

*当然如果正常的Producer，执行完进程会关闭，那么连接也就断了，但是如果是在Consumer中又调用了RPC的Producer（即在消费者中，又调用了rpc同步队列），这样这个Consumer是while(true)的循环，所以连接不会断，所以exclusive队列一直存在。直到连接被打满，机器挂。*


