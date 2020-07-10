---
title: "Kafka基础知识"
date: 2020-06-02
slug: "Kafka basic"
draft: false
tags:
- Tech
- Kafka
categories:
- Kafka

---


> 参考文档: https://juejin.im/post/5aa881c26fb9a028c979bfa5

## 一、简介

Apache Kafka是一个分布式消息发布订阅系统。支持分区（partition）、多副本（replica）,基于zookeeper协调。可以实时的处理大量数据以满足各种需求场景，比如基于hadoop的批处理系统、低延迟的实时系统、storm/Spark流式处理引擎、web/nginx日志、访问日志，消息服务等等。

## 二、优势

> Apache Kafka与传统消息系统相比，有以下不同：
> 1. 它被设计为一个分布式系统，易于向外扩展；
> 2. 它同时为发布和订阅提供高吞吐量；
> 3. 它支持多订阅者，当失败时能自动平衡消费者；
> 4. 它将消息持久化到磁盘，因此可用于批量消费，例如ETL，以及实时应用程序。

## 三、基础概念

> 1. Kafka是运行在一个集群上，所以它可以拥有一个或多个服务节点；
> 2. Kafka集群将消息存储在特定的文件中，对外表现为Topics；
> 3. 每条消息记录都包含一个key,消息内容以及时间戳；

## 四、核心接口

> 1. Producer API允许了应用可以向Kafka中的topics发布消息；
> 2. Consumer API允许了应用可以订阅Kafka中的topics,并消费消息；
> 3. Streams API允许应用可以作为消息流的处理者，比如可以从topicA中消费消息，处理的结果发布到topicB中；
> 4. Connector API提供Kafka与现有的应用或系统适配功能，比如与数据库连接器可以捕获表结构的变化

## 五、关键术语

#### 5.1 Broker

一台kafka服务器就是一个broker。一个集群由多个broker组成。一个broker可以容纳多个Topic。

#### 5.2 Topics

Topics是一些主题的集合，也可以认为是一个消息队列，生产者可以向其写入消息，消费者可以从中读取消息。一个Topic由一个或多个partition(分区)组成，支持多个生产者或消费者同时订阅它，所以其扩展性很好。

#### 5.3 Partition

- Topic是逻辑的概念，Partition是物理的概念，对用户来说是透明的。Kafka可以将主题(Topic)划分为多个分区(Partition)。
- Kafka会根据分区规则选择把消息存储到哪个分区中，只要如果分区规则设置的合理，那么所有的消息将会被均匀的分布到不同的分区中，这样就实现了负载均衡和水平扩展。
- 而多个订阅者可以从一个或者多个分区中同时消费数据，以支撑海量数据处理能力。Partition中的每条消息都会被分配一个有序的id(offset), kafka可以保证同一个Partition中的消息是顺序发给consumer，但是无法保证(多个partition间)的顺序。
- 此外，由于消息是以追加到分区中的，多个分区顺序写磁盘的总效率更高，是Kafka高吞吐率的重要保证之一。

#### 5.4 Partition Replicas

Kafka通过多副本机制实现故障自动转移，当Kafka集群中一个Broker失效情况下仍然保证服务可用。

kakfa会从多副本中选举出一个首领副本 (Leader replica)，所有的事件都直接发送给首领副本；其他副本作为跟随者副本 (Follower replica)，只通过复制来保持与首领副本数据一致，不做其他操作。当首领副本不可用时，会从Follower replica中选举新首领

>**Kafka中采用分区的设计有几个目的**
>
>1. 是可以处理更多的消息，不受单台服务器的限制。Topic拥有多个分区意味着它可以不受限的处理更多的数据
>2. 分区可以作为并行处理的单元
>3. 可以用来保障消息顺序。因为Topic一个分区中消息只能由消费者组中的订阅这个Topic的多个消费中唯一一个消费者处理，所以消息肯定是按照先后顺序进行处理的。但是它也仅仅是保证Topic的一个分区顺序处理，不能保证跨分区的消息先后处理顺序。
#### 5.5 Offset

分区中的消息都被分配了一个序列号，称之为偏移量(offset),在每个分区中此偏移量都是唯一的。

Kafka集群保持所有的消息，直到它们过期， 无论消息是否被消费了。

消费者所持有的仅有的元数据就是这个偏移量，也就是消费者在这个log中的位置。 这个偏移量由消费者控制，消费者可以将偏移量重置为更老的一个偏移量，重新读取消息。这种设计对消费者来说操作自如，一个消费者的操作不会影响其它消费者对此log的处理。

>kafka的存储文件都是按照offset.kafka来命名，用offset做名字的好处是方便查找。例如你想找位于2049的位置，只要找到2048.kafka的文件即可。值得注意的是, the first offset是00000000000.kafka

#### 5.6 Producer

消息生产者，就是向kafka broker发消息的客户端。生产者也负责选择消息发布到哪个Topic上的哪一个分区。最简单的方式从分区列表中轮流选择。也可以根据某种算法依照权重选择分区。在对消息读取顺序有严格要求的需求下，可以采取一些算法来保证消息会落到哪个分区，进而利用kafka单个分区顺序读取的机制。

#### 5.7 Consumer

消息消费者，向kafka broker取消息的客户端。

Kafka提供了单一的消费者抽象模型: 消费者组(consumer group)。

消费者用一个消费者组名标记自己。 一个发布在Topic上消息被分发给此消费者组中的一个消费者。

>消费者与分区数
>
>1. 当消费者数 > 分区数，会存在消费者浪费的情况，增加消费的数据无法提高消费效率
>2. 当消费者数 = 分区数，一个消费者只会处理一个分区的数据
>3. 当消费者数 < 分区数，一个消费者会处理多个分区的数据
#### 5.8 Consumer Group(CG)

消息系统有两类，一是广播，二是订阅发布。

广播是把消息发送给所有的消费者；发布订阅是把消息只发送给订阅者。

Kafka通过Consumer Group组合实现了这两种机制。

Topic的消息会复制（不是真的复制，是概念上的）到所有的CG，但每个CG只会把消息发给该CG中的一个 consumer，基于这个原理，
- 要实现广播，只要每个consumer有一个独立的CG就可以了。
- 要实现单播，只要所有的consumer在同一个CG。
- 用CG还可以将consumer进行自由的分组而不需要多次发送消息到不同的topic。

一个消费组内，订阅同一个Topic的多个消费者，一个消费只会处理它所订阅的Topic的一个分区下的数据。


Kafka会根据消费者组的情况均衡分配消息，比如有消息着实例宕机，亦或者有新的消费者加入等情况。

>**Consumer Group Rebalance 时机**
>
>1. 组成员个数发生变化。例如有新的 consumer 实例加入该消费组或者离开组。
>2. 订阅的 Topic 个数发生变化。
>3. 订阅 Topic 的分区数发生变化。
#### 5.9 Coordinator 

​		Coordinator 协调者，协调consumer、broker。早期版本中Coordinator，使用zookeeper实现，但是这样做，rebalance的负担太重。为了解决scalable的问题，不再使用zookeeper，而是让每个broker来负责一些group的管理，这样consumer就完全不再依赖zookeeper了。 

> **5.9.1 Consumer连接到coordinator**
>
> 从Consumer的实现来看，在执行poll或者是join group之前，都要保证已连接到Coordinator。**连接到coordinator的过程**是：
>
> - 连接到最后一次连接的broker（如果是刚启动的consumer，则要根据配置中的borker）。它会响应一个包含coordinator信息(host, port等)的response
> - 连接到coordinator。

> **5.9.2 Consumer Group Management**
>
> Consumer Group 管理中，也是需要coordinator的参与。一个Consumer要join到一个group中，或者一个consumer退出时，都要进行rebalance。**进行rebalance的流程是**：
>
> - 会给一个coordinator发起Join请求（请求中要包括自己的一些元数据，例如自己感兴趣的topics）
> - Coordinator 根据这些consumer的join请求，选择出一个leader，并通知给各个consumer。这里的leader是consumer group 内的leader，是由某个consumer担任，不要与partition的leader混淆。
> - Consumer leader 根据这些consumer的metadata，重新为每个consumer member重新分配partition。分配完毕通过coordinator把最新分配情况同步给每个consumer。
> - Consumer拿到最新的分配后，继续工作。