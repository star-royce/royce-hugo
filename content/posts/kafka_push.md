---
title: "Kafka消息推送 - GO"
date: 2020-06-02
slug: "Kafka product by go"
draft: false
tags:
- Tech
- Kafka
categories:
- Kafka


---

## Kafka 消息推送 - Golang

#### 依赖包

  github.com/Shopify/sarama

#### 消息推送模式
1. oneway
   - 这种模式只发消息，不会去管实际有没有发送成功，消息可靠性最低，但是低延迟、高吞吐，这种对于某些完全对可靠性没有要求的场景还是适用的。
   - 这种模式下，ACK模式只能设置为0。
2. 同步模式。
   - 此模式因为每次发消息都要等待broker的确认，收到确认后才能进行下一条消息的发送，所以吞吐量最低
   - 这种模式下，ACK模式一般是设置为1。即当你通过同步方式发送消息后，会等待broker的确认，确认结果等同于相应的Leader Replica备份结果。但是依旧存在着丢失数据的可能性，比如leader宕机并且还没有选出新的leader。
3. 异步模式
   - 这种模式下，生产者调用推送方法后，会把消息放到内部缓存区，同时结合batch.size，当缓存数据达到一定量时，就会批量发送给kafka，从而减少网络带来的系能开销
   - 异步模式下的消息可靠性同样取决于ACK模式的设置，为了确保消息不丢失，我们可以选择将ACK模式设置为all(-1)
   - 异步模式一般都需要设置一个回调函数，用于接收broker消息确认结果

#### ACK模式

- **acks=0，生产者不会等待broker的任何确认，消息会被立即添加到缓冲区并被认为已经发送**。在这种情况下，**不能保证服务器已经收到消息**，并且重试配置不会生效（因为客户端通常不会知道任何异常），每条消息返回的偏移量始终设置为-1。由于生产者不等待broker的任何确认，因此它可以以网络支持的最快速度发送消息，所以这个配置适用于实现非常高的吞吐量。
- **acks=1，在leader服务器的副本收到消息的同一时间，生产者会接收到broker的确认。如果消息不能写入leader的副本（例如，如果leader宕机并且还没有选出新的leader），生产者将接收到异常响应，然后可以重新发送消息，避免丢失数据。** 如果leader宕机并且消息没有被写入到新的leader（通过不确定的leader选举），**该消息仍然会丢失。** 在这种情况下，吞吐量取决于消息是同步还是异步发送。如果我们的客户端等待服务器的回复（通过上述的调用发送消息时返回的Future对象的get()方法），它明显会显著地增加延迟（至少通过网络往返）。如果客户端使用callback，则延迟不会那么明显，但吞吐量将受到正在发送消息数量的限制（例如，在接收到响应之前生产者将会发送多少消息）。
- **acks=all（或-1），一旦所有的同步副本接收到消息，生产者才会接收到broker的确认。这是最安全的模式，** 因为可以确保多于一个的broker接收到该消息，即使在宕机的情况下，该消息也能被保存。然而，延迟性会比acks=1的时候更高，因为需要等待所有broker接收到消息。
  - PS: kafka有一个配置参数，min.insync.replicas，默认是1（也就是只有leader），该属性规定了最小的ISR数。这意味着当acks为-1（即all）的时候，这个参数规定了必须写入的ISR集中的副本数，如果没达到，那么producer会产生异常。

#### 简易版代码实现

```go

func SaramaProducer()  {

    config := sarama.NewConfig()
    //等待服务器所有副本都保存成功后的响应
    config.Producer.RequiredAcks = sarama.WaitForAll
    //随机向partition发送消息
    config.Producer.Partitioner = sarama.NewRandomPartitioner
    //是否等待成功和失败后的响应,只有上面的RequireAcks设置不是NoReponse这里才有用.
    config.Producer.Return.Successes = true
    config.Producer.Return.Errors = true
    //设置使用的kafka版本,如果低于V0_10_0_0版本,消息中的timestrap没有作用.需要消费和生产同时配置
    //注意，版本设置不对的话，kafka会返回很奇怪的错误，并且无法成功发送消息
    config.Version = sarama.V0_10_0_1

    fmt.Println("start make producer")
    //使用配置,新建一个异步生产者, []string内是示例的kafka节点地址
    producer, e := sarama.NewAsyncProducer([]string{"182.61.9.153:6667","182.61.9.154:6667","182.61.9.155:6667"}, config)
    if e != nil {
        fmt.Println(e)
        return
    }
    defer producer.AsyncClose()

    //处理broker确认回调
    fmt.Println("start goroutine")
    go func(p sarama.AsyncProducer) {
        for{
            select {
            case  <-p.Successes():
                //fmt.Println("offset: ", suc.Offset, "timestamp: ", suc.Timestamp.String(), "partitions: ", suc.Partition)
            case fail := <-p.Errors():
                fmt.Println("err: ", fail.Err)
            }
        }
    }(producer)

    var value string
    for i:=0;;i++ {
        time.Sleep(500*time.Millisecond)
        time11:=time.Now()
        value = "this is a message 0606 "+time11.Format("15:04:05")

        // 发送的消息,主题。
        // 注意：这里的msg必须得是新构建的变量，不然你会发现发送过去的消息内容都是一样的，因为批次发送消息的关系。
        msg := &sarama.ProducerMessage{
            Topic: "0606_test",
        }

        //将字符串转化为字节数组
        msg.Value = sarama.ByteEncoder(value)
        //fmt.Println(value)

        //使用通道发送
        producer.Input() <- msg
    }
}
```

#### 项目实际实现-GO

1. 不可能在每次发送消息的时候去new一个producer
   - 实际实现: 在服务启动的时候就初始化一个异步的Producer,再提供一个push方法，实际有消息需要发送的时候，通过此方法发送即可。

2. 回调处理机制不够完善

   - 实现原理: 简易代码中，是组装好消息后，开启协程来监听success/error channle，然后再往input channel里边发消息，利用go的管道来实现回调监听的效果。
   - 问题:
     - 回调是通过channel来区分消息发送成功/失败，不解析value的话(即发送出去的msg)，并没有更多的可用于区分的信息
     - kafka只保证一个分区内的数据是顺序的，多分区间的顺序不保证
     - 项目本身存在并发场景，先收到回调的消息，不一定就是先触发了发送操作的那条
     - go 协程调度，也不是谁先申明了这个协程，就一定会被优先调度到
     - 基于以上结论分析，如果按简易代码的实现，在发消息的时候开协程，再根据协程监听到的channel做处理，则无法做一些消息的针对性处理，比如消息发送状态回写到表

   - 实际实现
     - 服务启动时，在成功初始化Producer之后，直接起一个协程池来监听回调。当没有任何消息产生时，select语法会让协程内的逻辑在此堵塞，直到有消息确认发生；当收到消息确认时, 结合value来做特定数据处理。
     - 为了避免重复的数据库查询操作，会在发送消息前，把这条消息的数据库ID记录到redis中，收到回调后，可以直接从redis拿到关键ID来做更新曹走

3. 资源关闭顺序性保障
   - 因为是在服务启动的时候初始化了永久的的Producer跟回调监听协程，所以在服务关闭时，也要有相应的资源关闭操作
   - 首先必须先关回调监听协程池，这样可以确保不会panic，也不会有回调消息丢失。值得注意的是，关闭协程池的时候，还要考虑协程内消息未处理完毕的情况。
   - 接着才是去关闭kafka Producer


#### 关键配置

1. Producer.RequiredAcks

   Kafka ACK 模式，这里为了保证消息不丢失，设置为all(-1)
2. Producer.Timeout

   推送响应超时时间，目前使用默认配置，设置为10秒。实际配置需根据实际情况修正。
3. Producer.Return.Successes

   配置ACK all使用，则必须设置为true
4. Producer.Return.Errors

   配置ACK all使用，则必须设置为true。默认配置即为true

5. Producer.Retry.Max
   - 重试次数，默认配置为3.
   - 为了解决重试机制引起的消息乱序，Kafka引入了Producer ID（即PID）和Sequence Number。对于接收的每条消息，如果其序号比Broker维护的序号）大一，则Broker会接受它，否则将其丢弃。
     - 如果消息序号比Broker维护的序号差值比一大，说明中间有数据尚未写入，即乱序，此时Broker拒绝该消息，Producer抛出InvalidSequenceNumber
     - 如果消息序号小于等于Broker维护的序号，说明该消息已被保存，即为重复消息，Broker直接丢弃该消息，Producer抛出DuplicateSequenceNumber

6. Producer.Retry.Backoff

   重试间隔，默认100ms。