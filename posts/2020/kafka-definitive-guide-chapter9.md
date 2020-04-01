---
layout: default
title: 第9章，管理Kafka
---

# 第9章，管理Kafka

#### 9.1.2　增加分区
>从消费者角度来看，为基于键的主题添加分区是很困难的。因为如果改变了分区的数量，键到分区之间的映射也会发生变化。所以，对于基于键的主题来说，建议在一开始就设置好分区数量，避免以后对其进行调整。

~基于键的分区都有这个困难，不管是Kafka，还是Mysql等。


#### 9.1.5　列出主题详细信息
>有两个参数可用于找出有问题的分区。使用--under-replicated-partitions参数可以列出所有包含不同步副本的分区。使用--unavailable-partitions参数可以列出所有没有首领的分区，这些分区已经处于离线状态，对于生产者和消费者来说是不可用的。  
示例：列出包含不同步副本的分区。# kafka-topics.sh --zookeeper zoo1.example.com:2181/kafka-cluster --describe --under-replicated-partitions  

~这两个参数对于问题的诊断非常重要。
