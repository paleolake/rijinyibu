---
layout: default
title: 第三章，Kafka生产者——向Kafka写入数据
---

# 第三章，Kafka生产者——向Kafka写入数据

### 3.1　生产者概览  
<img src="/images/kafka-definitive-guide.jpg" width="400">  
~ Kafka发送消息的主要步骤

### 3.2　创建Kafka生产者  
>发送消息主要有以下3种方式：  
发送并忘记（fire-and-forget）我们把消息发送给服务器，但并不关心它是否正常到达。大多数情况下，消息会正常到达，因为Kafka是高可用的，而且生产者会自动尝试重发。不过，使用这种方式有时候也会丢失一些消息。  
同步发送我们使用send()方法发送消息，它会返回一个Future对象，调用get()方法进行等待，就可以知道消息是否发送成功。  
异步发送我们调用send()方法，并指定一个回调函数，服务器在返回响应时调用该函数。

~应该了解。


### 3.4　生产者的配置
>acks参数指定了必须要有多少个分区副本收到消息，生产者才会认为消息写入是成功的。这个参数对消息丢失的可能性有重要影响。该参数有如下选项。     
acks=0，生产者在成功写入消息之前不会等待任何来自服务器的响应。  
acks=1，只要集群的首领节点收到消息，生产者就会收到一个来自服务器的成功响应。如果消息无法到达首领节点（比如首领节点崩溃，新的首领还没有被选举出来），生产者会收到一个错误响应，为了避免数据丢失，生产者会重发消息。  
如果acks=all，只有当所有参与复制的节点全部收到消息时，生产者才会收到一个来自服务器的成功响应。  

~需要知道。



>buffer.memor该参数用来设置生产者内存缓冲区的大小，生产者用它缓冲要发送到服务器的消息。如果应用程序发送消息的速度超过发送到服务器的速度，会导致生产者空间不足。这个时候，send()方法调用要么被阻塞，要么抛出异常，取决于如何设置block.on.buffer.full参数（在0.9.0.0版本里被替换成了max.block.ms，表示在抛出异常之前可以阻塞一段时间）。

~这个有别于大众的理解，不少人一般认为消息直接发送给了服务器，但不是这样。


>retries参数的值决定了生产者可以重发消息的次数，如果达到这个次数，生产者会放弃重试并返回错误。默认情况下，生产者会在每次重试之间等待100ms，不过可以通过retry.backoff.ms参数来改变这个时间间隔。建议在设置重试次数和重试时间间隔之前，先测试一下恢复一个崩溃节点需要多少时间（比如所有分区选举出首领需要多长时间），让总的重试时间比Kafka集群从崩溃中恢复的时间长，否则生产者会过早地放弃重试。

~知道有这个参数就行。



>batch.size当有多个消息需要被发送到同一个分区时，生产者会把它们放在同一个批次里。该参数指定了一个批次可以使用的内存大小，按照字节数计算（而不是消息个数）。  
linger.ms该参数指定了生产者在发送批次之前等待更多消息加入批次的时间。  

~两个组合使用的参数。


>max.in.flight.requests.per.connection该参数指定了生产者在收到服务器响应之前可以发送多少个消息。它的值越高，就会占用越多的内存，不过也会提升吞吐量。把它设为1可以保证消息是按照发送的顺序写入服务器的，即使发生了重试。

~这是一个非常重要的参数，值的设置将对消息发送的行为有很大影响。



>一般来说，如果某些场景要求消息是有序的，那么消息是否写入成功也是很关键的，所以不建议把retries设为0。可以把max.in.flight.requests.per.connection设为1，这样在生产者尝试发送第一批消息时，就不会有其他的消息发送给broker。不过这样会严重影响生产者的吞吐量，所以只有在对消息的顺序有严格要求的情况下才能这么做。

~对消息有顺序的要求如何做。