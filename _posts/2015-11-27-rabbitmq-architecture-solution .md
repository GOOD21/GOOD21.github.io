---
layout:     post
title:      "RabbitMQ-架构方案"
date:       2015-11-27
author:     "GOOD21"
header-img: "img/post-bg-unix-linux.jpg"
tags:
    - RabbitMQ
---

### Vhost & 权限

根据业务区分创建用户、设置权限

```shell
rabbitmqctl add_user test testststst
rabbitmqctl add_vhost /test
rabbitmqctl set_permissions -p /test test ".*" ".*" ".*" 
```
设置Vhost的好处是，后期我们可以通过不同的用户&权限来控制HA策略或者一些紧急降级措施。

### 负载均衡
RabbitMQ不提供负载均衡，它只是一个MessageBroker。所以我们使用Haproxy做负载均衡。

### 高可用方案
经过测试，consumer 挂在 slave 上会造成队列堆积，性能较低。最终选择ha-mode为nodes模式，指定集群 半数+1 台机器。

因为我们现在只有5台机器，consumer连接到slave的概率还是比较大的，所以我们通过haproxy指定consumer只能链接非master、slave的机器。

其实最好的方案应该是exactly，这种方案可以自动补全镜像（当某台slave挂了，非master、slave节点会自动升级为slave），而且不需要用haproxy做分组，直接轮询就好。但是这种方式要求集群机器要多，这样consumer连接slave的概率才会小。

```shell
rabbitmqctl set_policy -p /test ha-test "^test\." \ 
'{"ha-mode":"nodes","ha-params":["rabbit@mb6", "rabbit@mb7", "rabbit@mb8"]}'
```

### 架构图

私有 keepalived + haproxy 高可用负载均衡

![bbt-RabbitMQv2.jpg](/img/in-post/rabbitmq/bbt-RabbitMQv2.jpg)

### 压测

**平均吞吐32000/s**

**10分钟数据：**

![bbt1.png](/img/in-post/rabbitmq/bbt1.png)

**8小时数据：**

![bbt2.png](/img/in-post/rabbitmq/bbt2.png)

**1天数据：**

![bbt3.png](/img/in-post/rabbitmq/bbt2.png)

**master负载（mb6）：**

![bbt5.png](/img/in-post/rabbitmq/bbt5.png)

**master流量（mb6）：**

![bbt6.png](/img/in-post/rabbitmq/bbt6.png)

**master IO（mb6）：**

![bbt7.png](/img/in-post/rabbitmq/bbt7.png)

### 容灾测试

1、非满载情况测试：

* 停掉一台slave：没反应
* 停掉master：consumer连接断开，重连之后恢复正常，吞吐无明显变化

![bbt8.png](/img/in-post/rabbitmq/bbt8.png)

2、满载情况测试：

* 停掉一台slave：所有连接不断，吞吐略有下降
* 停掉master：consumer连接断开，重连之后恢复正常，吞吐下降10%，启动之后开始恢复。

![bbt9.png](/img/in-post/rabbitmq/bbt9.png)

![bbt10.png](/img/in-post/rabbitmq/bbt10.png)

### 监控&报警

cpu使用量、load、内存使用量、硬盘使用量、io、swap使用量、TCP连接数、bond0流量

ping、tcp 5672/15672/25672、cpu 80%、io 70%

### 总结

1. 其实吞吐这个值在实际的生产环境中有很多种因素影响，并不能达到预期的值，例如包大小、consumer执行时间等，所以本次测试中的吞吐跟大的意义是指：RabbitMQ及时消费的时候，每秒最高的publish量。
2. 关于横向扩展，nodes模式下，随着机器的增加，内网通信随之增加，集群性能增量会逐渐降低，所以目前最好的提高吞吐方式是调整机柜位置，最好集群能走一台（或两台）交换机，提高内网调用性能，但是这样的话，如果这个交换机挂的话，整个服务也就不可用了，所以这就是一个trade off 的选择。
3. 当机器数量多的时候，可以选择exactly模式，只有部分特殊业务可以指定节点。


