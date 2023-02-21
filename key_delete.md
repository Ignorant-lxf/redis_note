# 键删除策略

- 主要通过 pexpareat 这个命令背后的函数实现的
- 所有过期的键都存在一个字典中，就是 redisdb.expires ，key 为原来的键，val 为超时时间

**问题**
- 如何使得删除不在一个时间点上进行，避免在服务器很忙的时候火上浇油？
- 主从复制中如何处理定时时间，单一的方法可行吗？

**解决办法**
**惰性删除** 当使用到某个设置了超时时间的 key 时，就去检查它是否超时，超时则删除。某种程度上可以规避掉上面的第一个问题。但是会存在新的问题：  
- 如果某个设置了超时时间的 key 长时间不被使用，就会一直存在于内存中，而内存对于 redis 来说很宝贵
- 如果使用主从复制实现读写分离，那么可能主服务器永远无法被读到，导致多个服务器内 大量数据冗余

**定期删除** 在惰性删除的基础上，可以在每个时间周期里面执行一个定期删除    
- 在每次执行定期删除时都会记录执行时间，根据处理时间来判断当前过期的键与未过期的键的疏密程度，这样可以最大程度节省资源

从删除策略上来讲其实我们最容易想到的不应该是上面的两种方法,而是使用定时器,在超时时执行回调,在使用时间轮的情况下,我们可以做到O(1)删除超时的元素,最大程度的节省内存.
但是可能我们的服务器在当前并不缺少内存,没有必要去节省,所以此时CPU时间应该优先处理请求.而且对于内存数据库这样要求高性能的场合来讲,使用定时器导致的多次系统调用可能也是问题之一(并没有测试),这也许就是redis没有采用定时器来实现超时,而是采用惰性删除和定期删除相结合的方式来实现超时的原因吧.

**refer to https://blog.csdn.net/weixin_43705457/article/details/105030025**