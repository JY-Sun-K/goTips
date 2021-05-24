### 6个核心概念

**virtual hosts**：区分队列，隔离账号使用

 **connections**:  rabbitMQ 监听进程并展示

**exchange**（交换机）:  作用：他相当于一个中转，我们所有已声明的交换机都会在这边罗列出来。当一个生产者发送一个消息以后，它会首先进入我们的交换机，然后根据我们交换机指定的规则绑定对应的key，把这所有的消息通过交换机发送到对应的key里面。交换机就是这个作用。

**channel**:进行通讯

**queues**：队列作用就是绑定我们的交换器，或者是接受我们的消息，只要没有消费者，我们就一直把消息存储在队列里面，这里的队列就主要承担着临时存储消息的作用

**binding**



![rabbitmq原理](C:\Users\king\Desktop\rabbitmq原理.png)

​		

```
    type RabbitMQ struct {
       conn *amqp.Connection
       channel *amqp.Channel 
       //队列名称
       QueueName string
       //交换机
       Exchange string
       //key
       Key string
       //连接信息
       Mqurl string
    }
```



### 6个工作模式

##### 	 simple模式

##### work模式 

负载作用

##### 订阅模式

![rabbitmq订阅](C:\Users\king\Desktop\rabbitmq订阅.png)