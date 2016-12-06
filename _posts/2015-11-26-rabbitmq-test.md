---
layout:     post
title:      "RabbitMQ-测试"
date:       2015-11-26
author:     "GOOD21"
header-img: "img/post-bg-unix-linux.jpg"
tags:
    - RabbitMQ
---

### 单机测试

机器：mem12 单机直连 4核4线程     
是否持久化：否     
结论：无论启1个或者两个或者4个producer的进程，总吞吐一直在16000/s

![single_test1.png](/img/in-post/rabbitmq/single_test1.png)

机器：mem12 单机直连 4核4线程     
是否持久化：否     
过程：
定义一组pc（producer/consumer）为一个p对应6个c

1. 启动一组pc，吞吐稳定在16000/s 跟上面的测试效果一样
2. 增加一组pc，吞吐波动在16000-18000/s 
3. 增加第三组pc，出现如下图情况，producer量上增，而consumer量下降，阶段性crash、重启；
4. 通过flow control 发现是卡在mem12的cpu上

![single_test4.png](/img/in-post/rabbitmq/single_test4.png)

![single_test5.png](/img/in-post/rabbitmq/single_test5.png)

![single_test6.png](/img/in-post/rabbitmq/single_test6.png)

机器：mb4 单机直连 12核24线程     
是否持久化：否     
过程：如上面方法，从第7/8组开始，基本上的基本上都稳定在了100000/s左右（这个瓶颈应该就是在单核的处理能力上了），最终是加到了10组，此时cpu大概在80%左右，其实还可以往上再压，但是已经不重要了，我们的需求是8000/s。       
（从此，mem12没了朋友）

![single_test7.png](/img/in-post/rabbitmq/single_test7.png)

![single_test8.png](/img/in-post/rabbitmq/single_test8.png)

机器：mb4 单机直连 12核24线程     
是否持久化：是     
过程：如上述过程，直接压到了第14组，此时的cpu利用率是80%左右，io在20%，吞吐在85000/s左右。
结论：增加持久化功能（在总cpu80%、消息字节不多的情况下）吞吐率会下降25%左右。这个可以忍。

*为何图表的线条相对于上面的要平缓？       
个人觉得应该是此时的单核cpu是因为部分的io导致等待，并没有完全利用。而上图是单核cpu已经到极限了。*

![single_test9.png](/img/in-post/rabbitmq/single_test9.png)

![single_test10.png](/img/in-post/rabbitmq/single_test10.png)

![single_test11.png](/img/in-post/rabbitmq/single_test11.png)


### 集群测试

最先选择的exactly模式：

* haproxy：10.1.2.41:5672 mb11 轮询
* 机器：mb4、mb5、mb6、mb7、mb8 12核24线程
* ha-mode：exactly 3

发现如下图不稳定的情况：

* 队列阶段性堆积
* publish的量和deliver的量互相制约

![cluster_test1.png](/img/in-post/rabbitmq/cluster_test1.png)

为了查明上述情况，做了如下测试，producer/consumer分别指定连接 同一队列 的master、mirror、none，测试是否稳定 √

| publish\consumer | master | mirror | none |
| --- | --- | --- | --- |
| master | √（8000/s） | X（9000/s 略有堆积） | √（8000/s） |
| mirror | √（12000/s）  | X（6000-9000/s 大量堆积） | √（12000/s） |
| none | √（10000/s） | X（10000/s 大量堆积） | √（9000/s） |

producer master 、 consumer mirror：

![cluster_test2.png](/img/in-post/rabbitmq/cluster_test2.png)

producer master 、 consumer none：

![cluster_test3.png](/img/in-post/rabbitmq/cluster_test3.png)

producer mirror 、 consumer master：

![cluster_test4.png](/img/in-post/rabbitmq/cluster_test4.png)

producer mirror 、 consumer mirror：

![cluster_test5.png](/img/in-post/rabbitmq/cluster_test5.png)

producer mirror 、 consumer none：

![cluster_test6.png](/img/in-post/rabbitmq/cluster_test6.png)

producer none 、 consumer master：

![cluster_test7.png](/img/in-post/rabbitmq/cluster_test7.png)

producer none 、 consumer mirror：

![cluster_test8.png](/img/in-post/rabbitmq/cluster_test8.png)

producer none 、 consumer none（case1：same node 8500/s）：

![cluster_test9.png](/img/in-post/rabbitmq/cluster_test9.png)

producer none 、 consumer none（case2：different node 9000/s）：

![cluster_test10.png](/img/in-post/rabbitmq/cluster_test10.png)


**测试结论**：

* consumer 挂在 mirror 上会造成队列堆积，性能较低
* producer/consumer都在master上的话由于master本身的性能损耗（同步mirror），导致吞吐下降

***为什么consumer 挂在 mirror 上会造成队列堆积，性能较低？***

*官方文档中说所有的非publish操作都是先由master处理，再广播到各个slave上的（All actions other than publishes go only to the master, and the master then broadcasts the effect of the actions to the slaves.）*

*分析一下consumer工作完成后连接slave 之后的事情：*

![cluster_test11.png](/img/in-post/rabbitmq/cluster_test11.png)

*1、 slave将任务发给master*     
*2、 master处理完成后，异步的干了两件事：*

* *告诉连接master的slave完成了*
* *广播给所有slave更新状态*

*3、 slave接到master处理完的请求*      
*4、 slave要等待本身状态的更新*

*通过第4点，这就能看出来为什么连接slave会造成队列堆积，而连接master或none的时候不会。*

*然后RabbitMQ通过Flow Control机制，限制了publish的量，所以堆积量又会降下来，像这样一直反复。*

