---
layout:     post
title:      "RabbitMQ-安装"
date:       2015-11-23
author:     "GOOD21"
header-img: "img/post-bg-unix-linux.jpg"
tags:
    - RabbitMQ
---

### Server端

###### 下载安装包

```shell
# 官方下载地址
wget https://www.rabbitmq.com/releases/rabbitmq-server/v3.5.6/rabbitmq-server-3.5.6-1.noarch.rpm 

yum localinstall -y rabbitmq-server-3.5.6-1.noarch.rpm

# 如果yum直接安装提示需要erlang的话，先
# wget https://www.rabbitmq.com/releases/erlang/erlang-18.1-1.el6.x86_64.rpm
# 安装erlang的包，然后在yum localinstall RabbitMQ
```

###### 配置/htc/hosts 文件

将本机&集群的机器配置在里面，例如：

```text
127.0.0.1   mb4 localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6

10.1.2.35 mb5
10.1.2.36 mb6
10.1.2.37 mb7
10.1.2.38 mb8
```

###### 新建配置文件

vim /etc/rabbitmq/rabbitmq-env.conf 

```text
RABBITMQ_NODE_IP_ADDRESS=0.0.0.0
RABBITMQ_NODE_PORT=5672
RABBITMQ_NODENAME=rabbit@mb4
```

###### 启动RabbitMQ

```shell
/etc/init.d/rabbitmq-server start
```

###### 搭建集群

1、机器与机器之间需要共享一个erlang cookie，可以理解为公钥：               
vim /var/lib/rabbitmq/.erlang.cookie

```text
cookie：GQTQOWAHUPJDTMROXIOC
```

2、加入集群

```shell
# 先停止应用
rabbitmqctl stop_app

# 加入集群
rabbitmqctl join_cluster rabbit@mb4

# 启动应用
rabbitmqctl strat_app
```
3、查看集群状态

```text
mb4 ~ # rabbitmqctl cluster_status

Cluster status of node rabbit@mb4 ...
[{nodes,[{disc,[rabbit@mb4,rabbit@mb5,rabbit@mb6,rabbit@mb7,rabbit@mb8]}]},
 {running_nodes,[rabbit@mb8,rabbit@mb7,rabbit@mb6,rabbit@mb5,rabbit@mb4]},
 {cluster_name,<<"rabbit@mb4">>},
 {partitions,[]}]
```

###### 创建用户&权限

```shell
# 创建用户
rabbitmqctl add_user username passwd  

# 设置管理员权限
rabbitmqctl set_user_tags username administrator  

# 设置访问权限
rabbitmqctl set_permissions -p / username ".*" ".*" ".*"  
```

### PHP 客户端

###### 安装librabbitmq包

```shell
yum install librabbitmq-devel.x86_64
```

###### 下载&安装官方pecl包

```shell
wget https://pecl.php.net/get/amqp-1.6.1.tgz

tar -zxvf amqp-1.6.1.tgz 

cd amqp-1.6.1

phpize

./configure

make && make install
```

###### 增加配置文件

```text
extension=amqp.so
```

安装完成后 php -m 一下，无报错即为正常。

