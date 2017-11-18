# 关于 NSQ 的使用需要改进的几个方面

### 减少 Topic 的数量。

> A topic is a distinct stream of data.  
> A channel is a logical grouping of consumers subscribed to a given topic

NSQ 的机制是一个 Topic，至少需要一个消费者处理。
不应该用 Topic 取代路由的功能。

> Basic message routing - with NSQ, topics and channels are all you get. There’s no concept of routing or affinity based upon on key. It’s something that we’d love to support for various use cases, whether it’s to filter individual messages, or route certain ones conditionally. *Instead we end up building routing workers, which sit in-between queues and act as a smarter pass-through filter.*(参考：[Scaling NSQ to 750 Billion Messages](https://segment.com/blog/scaling-nsq/))

nsq 作者(mreiferson)回答的关于 Topic 数量的问题：

[topic capacity #307](https://github.com/nsqio/nsq/issues/307)

[拓扑图](https://camo.githubusercontent.com/ad53b80b1e65de2d6ced74232d8dd5c6cbb170d3/68747470733a2f2f662e636c6f75642e6769746875622e636f6d2f6173736574732f3138373434312f323133313636302f38626436326464322d393261332d313165332d383836332d3863643964633263376366632e706e67)

另外关于在 IM 场景下的使用建议[Design question on topics](https://groups.google.com/forum/#!topic/nsq-users/AeoiVMt37eE)

### 只发送需要的消息

NSQ 尽最大程度的保证消息不会丢失，一个原因是它没有 replication 的机制，另外一个原因应该是它认为发送给它的消息应该都是一定要使用的消息。应该利用它的这个特性，而不是抵消这个特性。

比如，自动外呼作为一个Topic，只发送跟自动外呼相关的消息。如果以后需要其他消息，可以根据需要的消息类型，再定义一个新的Topic，在其中发送需要的相关消息。这样就可以保证每个消息都是需要被消费的。

或者，生产者发送所有的消息，消费者根据 channel 来区分使用方式，由消费者进行过滤。类似官网实例的[图](http://nsq.io/overview/design.html)


## 其他
（参考：[http://nsq.io/overview/design.html]((http://nsq.io/overview/design.html))）
### 关于 topic 和 channel 创建的时机

> Topics and channels are not configured a priori. *Topics are created on first use by publishing to the named topic or by subscribing to a channel on the named topic.* Channels are created on first use by subscribing to the named channel.

### 连接机制

> At a lower level each nsqd has a long-lived TCP connection to nsqlookupd over which it periodically pushes its state. This data is used to inform which nsqd addresses nsqlookupd will give to consumers. For consumers, an HTTP /lookup endpoint is exposed for polling

每个 nsqd 会与 nsqlookupd 建立 TCP 长连接，定时推送自己的状态。nsqlookd 保存这些状态，consumer 通过定时调用 HTTP /lookup 向 nsqlookupd 获取 nsqd 的位置。


### 消息发送机制

> NSQ guarantees that a message will be delivered at least once, though duplicate messages are possible. Consumers should expect this and de-dupe or perform idempotent operations.

nsq 如何保证消息至少被发送一次：
假设 consumer client 已经连接上的 nsqd

1. client 通知 nsqd 它已经准备好接受消息
2. nsqd 把消息发送出去并且*临时在本地缓存一份*
3. client 回复 FIN(finish) 或者 REQ(requeue) 分别表明消息接受成功或者失败。

> This ensures that the only edge case that would result in message loss is an unclean shutdown of an nsqd process. In that case, any messages that were in memory (or any buffered writes not flushed to disk) would be lost.

> Client libraries are designed to send a command to update RDY count when it reaches ~25% of the configurable max-in-flight setting (and properly account for connections to multiple nsqd instances, dividing appropriately).

### 当单点 nsqd 崩溃，如何保证消息不丢失

> In the case where we require reliability and guaranteed delivery we publish to multiple nodes and standard channels that persist messages in the case of subscribers disconnecting or overflow in what can't be buffered in memory

> If preventing message loss is of the utmost importance, even this edge case can be mitigated. One solution is to stand up redundant nsqd pairs (on separate hosts) that receive copies of the same portion of messages. Because you’ve written your consumers to be idempotent, doing double-time on these messages has no downstream impact and allows the system to endure any single node failure without losing messages.

一个服务将同样的消息同时分发给多个 nsqd，如果有两个 nsqd 就在服务内启动两个生产者的 client。消息内增加 uid，消费者去重。
