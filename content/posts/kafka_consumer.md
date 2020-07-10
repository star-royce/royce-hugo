---
title: "Kafka消费详解"
date: 2020-06-02
slug: "Kafka consumer detail"
draft: false
tags:
- Tech
- Kafka
categories:
- Kafka

---

> 参考文档: https://www.cnblogs.com/rainwang/p/7493742.html

## 一、Consumer的线程安全性

&emsp;&emsp;KafkaProducer是线程安全的，但Consumer却没有设计成线程安全的。当用户想要在在多线程环境下使用kafkaConsumer时，需要自己来保证synchronized。

&emsp;&emsp;当你想要关闭Consumer或者为也其它的目的想要中断Consumer的处理时，可以调用consumer的wakeup方法。这个方法会抛出WakeupException。

## 二、Consumer 消费方式

&emsp;&emsp;Consumer读取partition中的数据是通过client端主动调用发起一个fetch请求来执行的。

&emsp;&emsp;而从KafkaConsumer来看，它有一个**poll**方法。但是这个poll方法只是可能会发起fetch请求。

&emsp;&emsp;原因是：Consumer每次发起fetch请求时，读取到的数据是有限制的，通过配置项max.partition.fetch.bytes来限制的。而在执行poll方法时，会根据配置项个max.poll.records来限制一次最多pool多少个record。

&emsp;&emsp;那么就可能出现这样的情况： 在满足max.partition.fetch.bytes限制的情况下，假如fetch到了100个record，放到本地缓存后，由于max.poll.records限制每次只能poll出15个record。那么KafkaConsumer就需要执行7次才能将这一次通过网络发起的fetch请求所fetch到的这100个record消费完毕。其中前6次是每次poll15个record，最后一次是poll出10个record。



## 三、Consumer Offset Commit

&emsp;&emsp;由于Kafka消费方式是由client端主动拉取，所以当使用完poll从本地缓存拉取到数据之后，client端要调用commitSync方法（或者commitAsync方法）去commit 下一次该去读取 哪一个offset的message。而这个commit方法会通过走网络的commit请求将offset在coordinator中保留，这样就能够保证下一次读取，既不会重复消费消息，也不会遗漏消息。

Kafka Consumer Java Client支持两种模式：

- 由KafkaConsumer自动提交

  ```java
   Properties props = new Properties();
   ...
   props.put("enable.auto.commit", "true");
   // 提交偏移量的时间间隔，默认5000ms
   props.put("auto.commit.interval.ms", "1000");
  ```

- 用户通过调用commitSync、commitAsync方法的方式完成offset的提交

  ````java
  Properties props = new Properties();
   ...
   props.put("enable.auto.commit", "true");
   props.put("auto.commit.interval.ms", "1000");
  
  /**
  	手动提交 - 同步方式
  	隐患：一定程度上限制了应用程序的吞吐量，因为提交请求后，broker响应之前应用程序会一直阻塞。
  */
  consumer.commitSync();
  /**
  	手动提交 - 异步方式
  	优势：可以通过降低提交频率来提升吞吐量
  	隐患：一旦发生再均衡，会增加重复消息的数量
  */
  consumer.commitAsync();
  
  /**
  	更多使用 - 也可以提交特定偏移量，同步/异步两种方式均可
  */
  consumer.commitAsync(currentOffsets, null);
  
  /**
  	更多使用 -  同步/异步两种方式可以组合提交，用来确保提交能够成功
  	ps: 具体使用场景不是很明白。
  */
  try {
      for (; ; ) {
         ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(100));
         consumer.commitAsync();
      }
  } finally {
     try{
        consumer.commitSync();
     } finally{
        consumer.close();
  }
  ````

  

## 四、Consumer Configuration

<table><tr><td bgcolor=gray><big>bootstrap.servers</big></td></tr></table>

&emsp;&emsp;在启动consumer时配置的broker地址的。不需要将cluster中所有的broker都配置上，因为启动后会自动的发现cluster所有的broker。配置的格式是：host1:port1;host2:port2…

<table><tr><td bgcolor=gray><big>key.descrializer、value.descrializer</big></td></tr></table>

Message record 的key, value的反序列化类。有内置的 **StringDeserializer**，也可自定义。

<table><tr><td bgcolor=gray><big>group.id</big></td></tr></table>

用于表示该consumer想要加入到哪个group中。

<table><tr><td bgcolor=gray><big>heartbeat.interval.ms</big></td></tr></table>

&emsp;&emsp;心跳间隔。心跳是在consumer与coordinator之间进行的。心跳是确定consumer存活，加入或者退出group的有效手段。

&emsp;&emsp;这个值必须设置的小于session.timeout.ms，因为当Consumer由于某种原因不能发Heartbeat到coordinator时，并且时间超过session.timeout.ms时，就会认为该consumer已退出，它所订阅的partition会分配到同一group 内的其它的consumer上。

&emsp;&emsp;通常设置的值要低于session.timeout.ms的1/3。默认值是：3000 （3s）

<table><tr><td bgcolor=gray><big>session.timeout.ms</big></td></tr></table>

&emsp;&emsp;Consumer session 过期时间。这个值必须设置在broker configuration中的group.min.session.timeout.ms 与 group.max.session.timeout.ms之间。其默认值是：10000 （10 s） 

<table><tr><td bgcolor=gray><big>enable.auto.commit</big></td></tr></table>

&emsp;&emsp;Consumer 在commit offset时有两种模式：自动提交(true)，手动提交(false)。更多详情可参考 **三、Consumer Offset Commit** , 默认值是true。

<table><tr><td bgcolor=gray><big>auto.commit.interval.ms</big></td></tr></table>

自动提交间隔。范围：[0,Integer.MAX]，默认值是 5000 （5 s）

<table><tr><td bgcolor=gray><big>auto.offset.reset</big></td></tr></table>

&emsp;&emsp;这个配置项，是告诉Kafka Broker在发现kafka在没有初始offset，或者当前的offset是一个不存在的值（如果一个record被删除，就肯定不存在了）时，该如何处理。它有4种处理方式(**默认值是latest**)：

​	1） earliest：自动重置到最早的offset。

​	2） latest：看上去重置到最晚的offset。

​	3） none：如果边更早的offset也没有的话，就抛出异常给consumer，告诉consumer在整个consumer group中都没有发现有这样的offset。

​	4） 如果不是上述3种，只抛出异常给consumer。 

<table><tr><td bgcolor=gray><big>connections.max.idle.ms</big></td></tr></table>

&emsp;&emsp;连接空闲超时时间。因为consumer只与broker有连接（coordinator也是一个broker），所以这个配置的是consumer到broker之间的。默认值是：540000 (9 min)

<table><tr><td bgcolor=gray><big>fetch.max.wait.ms</big></td></tr></table>

&emsp;&emsp;Fetch请求发给broker后，在broker中可能会被阻塞的（当topic中records的总size小于fetch.min.bytes时），此时这个fetch请求耗时就会比较长。这个配置就是来配置consumer最多等待response多久。 

<table><tr><td bgcolor=gray><big>fetch.min.bytes</big></td></tr></table>

&emsp;&emsp;当consumer向一个broker发起fetch请求时，broker返回的records的大小最小值。如果broker中数据量不够的话会wait，直到数据大小满足这个条件。取值范围是：[0, Integer.Max]，默认值是1。

默认值设置为1的目的是：使得consumer的请求能够尽快的返回。

<table><tr><td bgcolor=gray><big>fetch.max.bytes</big></td></tr></table>

&emsp;&emsp;一次fetch请求，从一个broker中取得的records最大大小。如果在从topic中第一个非空的partition取消息时，如果取到的第一个record的大小就超过这个配置时，仍然会读取这个record，也就是说在这片情况下，只会返回这一条record。

&emsp;&emsp;broker、topic都会对producer发给它的message size做限制。所以在配置这值时，可以参考broker的message.max.bytes 和 topic的max.message.bytes的配置。取值范围是：[0, Integer.Max]，默认值是：52428800 （5 MB）

<table><tr><td bgcolor=gray><big>max.partition.fetch.bytes</big></td></tr></table>

&emsp;&emsp;一次fetch请求，从一个partition中取得的records最大大小。如果在从topic中第一个非空的partition取消息时，如果取到的第一个record的大小就超过这个配置时，仍然会读取这个record，也就是说在这片情况下，只会返回这一条record。

&emsp;&emsp;broker、topic都会对producer发给它的message size做限制。所以在配置这值时，可以参考broker的message.max.bytes 和 topic的max.message.bytes的配置。

<table><tr><td bgcolor=gray><big>max.poll.interval.ms</big></td></tr></table>

前面说过要求程序中不间断的调用poll()。如果长时间没有调用poll，且间隔超过这个值时，就会认为这个consumer失败了。

<table><tr><td bgcolor=gray><big>max.poll.records</big></td></tr></table>

Consumer每次调用poll()时取到的records的最大数。

<table><tr><td bgcolor=gray><big>receive.buffer.byte</big></td></tr></table>

Consumer receiver buffer （SO_RCVBUF）的大小。这个值在创建Socket连接时会用到。

取值范围是：[-1, Integer.MAX]。默认值是：65536 （64 KB）

如果值设置为-1，则会使用操作系统默认的值。

<table><tr><td bgcolor=gray><big>request.timeout.ms</big></td></tr></table>

请求发起后，并不一定会很快接收到响应信息。这个配置就是来配置请求超时时间的。默认值是：305000 （305 s）

<table><tr><td bgcolor=gray><big>client.id</big></td></tr></table>

Consumer进程的标识。如果设置一个人为可读的值，跟踪问题会比较方便。

<table><tr><td bgcolor=gray><big>interceptor.classes</big></td></tr></table>

用户自定义interceptor。

<table><tr><td bgcolor=gray><big>metadata.max.age.ms</big></td></tr></table>

Metadata数据的刷新间隔。即便没有任何的partition订阅关系变更也行执行。

范围是：[0, Integer.MAX]，默认值是：300000 （5 min）