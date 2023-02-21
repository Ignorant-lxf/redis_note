# 发布订阅
redis在2.8版本以后实现了一个发布订阅机制,这使得redis在我们需要一个基本的发布订阅功能的时候可以充当一个消息队列.
Redis一共为我们提供了六个命令,两种匹配方法来实现发布订阅

- 客户端可以一次性订阅一个或者多个 channel，subscribe channel channel2 channel3
- PUBSUB 返回当前 publish/subscribe 系统的内部命令的活动状态,包含三个内部命令,分别为:channels(列出当前活跃的 channel),NUMSUB(返回指定 channel 的订阅数目),NUMPAT(返回订阅 pattern 的订阅数,不需要参数)
- 订阅多个 channel，即模式匹配. psubscribe chan*
- 消息发布，publish channel hello
- 取消某一个 channel 消息订阅，unsubscribe channel
- 取消某个 pattern 的消息订阅，punsubsribe chan*

上面包括两种匹配方式，**精准的频道匹配**和**正则表达式的模式匹配**  

- 频道匹配用 **字典** 作为数据结构，key 为频道，val 为用户链表
- 模式匹配中使用 **链表** 作为底层数据结构。节点结构为 pubsubPattern{模式串，用户信息}  

至于为什么模式匹配不用字典作为底层结构，可能是模式串的重复远远小于频道的重复吧，毕竟模式匹配支持正则表达式

