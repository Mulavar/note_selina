### 基本概念

- **Broker**: 接收和分发消息的应用，RabbitMQ Server就是Message Broker。
- **Virtual host**: rabbitmq不能直接为用户分配一个可以访问哪一些exchange，或者queue的权限，因为rabbitmq的权限细粒度没有细化到交换器和队列，**他的最小细粒度是vhost(vhost中包含许多的exchanges，queues，bingdings)。**如果exchangeA 和queueA 只能让用户A访问，exchangeB 和queueB 只能让用户B访问，要达到这种需求，只能为exchangeA 和queueA创建一个vhostA，为exchangeB 和queueB 创建vhostB，这样就隔离开来了。
- **Connection**: publisher／consumer和broker之间的TCP连接。断开连接的操作只会在client端进行，Broker不会断开连接，除非出现网络故障或broker服务出现问题。
- **Channel**: 如果每一次访问RabbitMQ都建立一个Connection，在消息量大的时候建立TCP Connection的开销将是巨大的，效率也较低。Channel是在connection内部建立的逻辑连接，如果应用程序支持多线程，通常每个thread创建单独的channel进行通讯，AMQP method包含了channel id帮助客户端和message broker识别channel，所以channel之间是完全隔离的。Channel作为轻量级的Connection极大减少了操作系统建立TCP connection的开销。
- **Exchange**: 接受生产者发送的消息，并根据Binding规则将消息路由给服务器中的队列。ExchangeType决定了Exchange路由消息的行为，例如，在RabbitMQ中，ExchangeType有direct (point-to-point), topic (publish-subscribe) and fanout (multicast)三种，不同类型的Exchange路由的行为是不一样的。
- **Queue**: 消息最终被送到这里等待consumer取走。一个message可以被同时拷贝到多个queue中。
- **Binding**: exchange和queue之间的虚拟连接，binding中可以包含routing key。Binding信息被保存到exchange中的查询表中，用于message的分发依据。



### 订阅模式

一个生产者，多个消费者，每一个消费者都有自己的一个队列，生产者没有将消息直接发送到队列，而是发送到了交换机，每个队列绑定交换机，生产者发送的消息经过交换机，到达队列，实现一个消息被多个消费者获取的目的。需要注意的是，如果将消息发送到一个没有队列绑定的exchange上面，那么该消息将会丢失，这是因为在rabbitMQ中exchange不具备存储消息的能力，只有队列具备存储消息的能力。
