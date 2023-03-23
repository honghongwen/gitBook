# RabbitMq的一些使用

### 介绍
> 消息队列各种各样，从基础的Redis、到RabbitMq,到阿里的RocketMq,Go原生的Nsq以及Kafka等。
> 消息队列的主要用途是跨线程/进程处理业务。


 ### 举例  
> 
> 比如，有两个服务，订单服务，积分服务。
> 由于积分的强一致性并不是特别高，可以在用户下完单后，发送一个消息到消息队列中。而在积分服务中，可以消费这
> 条下单消息，从而执行增加用户积分逻辑。整个过程是异步且解耦的。后续如果还有新扩展的业务如订单统计服务等，
> 也可以消费此条消息，从而统计订单数据。

### 说明
> 由于我们的系统已经选用了RabbitMq,所以做下常用的了解有助于日常工作。

---

## 1. RabbitMq的组件

* Exchange 交换机
* Queue 队列
* Binding 绑定键


> 消息生产者发送消息时需要指定交换机以及路由键，而交换机会根据交换机的类型确定下发到的队列上。消费者指定队列进行消费消息。

![image](../../images/mq.png)

---

### 1.1 常用Exchange类型

* Direct 
> 直连类型，根据指定的绑定键发送到指定的队列。在mq后台，需要配置绑定键以及队列，交换机。

* Fanout
> 广播模式下，不会使用到绑定键，而是使用匿名队
> 列，所有绑定到该交换机下的队列都会收到消息。

* Topic
> 主题模式下，会根据具体的路由键配置，发送到不同
> 的队列中，可以根据通配符进行配置。



--- 

### 1.2 配置队列

> 队列的配置需要在后台进行操作，默认用户guest/guest，
> 在部署的服务器15672接口下可以访问该后台。
> 假设已经配置好vHost和用户的前提下，需要继续进行以下的
> 配置。

1. Exchange
> 新建exchange，指定名称和类型。

2. Queue
> 新建队列，指定名称。

3. Binding
> 回到Exchange，点进刚创建的Exchange,添加Bind，To queue为刚新建的queue,routingkey自己指定。


--- 

### 1.3 伪编码

1. 生产者(订单服务)

   ```java
   public void addOrder(Order order) {
       orderDao.insert(order);
       
       rabbitMqTemplate.convertAndSend(
        // 交换机   Exchange
        "sinfor.directExchange", 
        // 路由键 Binding
        "sinfor.order.addOrder",
        // 封装订单消息数据 
        new MsgData(order))
   }
   ``` 

2. 消费者(积分服务)
```java
    @RabbitListener(queues = "sinfor.order.queue")
    public void handlerAddOrder(String message) {
        log.info("接受到下单数据{}", message);
        MsgData msg = JSONUtil.toBean(message, MsgData.class);
        String messageId = msg.getMessageId;
        // 根据msgId做幂等
        // 积分落库
        Order order = (Order) msg.getData();
        insertCredit(order);
    }
```


## 2. 交换机的选择

### 2.1 上述问题
> 假如我还有订单统计服务也想接收下单消息，那么上面的交换机配置就不能满足了。因为RabbitMq队列默认是轮询模式，假如订单统计服务也是消费sinfor.order.queue这个队列，那么积分&统计服务中只能有一个服务能正常的接受到数据了。

### 2.2 广播模式

> 假如我选择了fanout模式的交换机，是否可以解决这个问题。答案是可以，但是会带来另外的问题。
> 假设现在有两个队列，一个给积分服务使用，另一个给统计服务使用。可以解决掉上面使用同一个队列带来的问题。但是，后续假如用户取消了订单，订单服务中又新增了取消订单消息。
> 现在就有两种消息了，下单&取消订单消息，而这两种消息都走的同一个队列，那就只能在封装的MsgData中设置字段来区分开具体的消息。而且还会带来如果某一个消息数据非常大/出问题了，阻塞了这个队列上所有的消息。

![mq2](../../images/mq2.png)


### 2.3 Topic模式
> topic类型的exchange允许用户根据具体的路由键通配符进行匹配，会将消息发送到匹配上的队列上。比如，有消息sinfor.order.add（路由键）消息和sinfor.order.del消息，对积分服务来说，我想监听两个消息，而对统计服务来说，我暂时只统计每天的下单数量，不考虑用户取消掉的数量（模拟场景）。那么我只需要监听sinfor.order.add这个消息即可。此时，我可以创建两个队列，一个给积分服务，另一个给统计服务。而积分服务需要匹配两个消息，统计服务只需要匹配新增订单的消息。

#### 2.3.1 队列配置如下

* queue.sinfor.credit  &nbsp;&nbsp;&nbsp;队列名称，积分队列
  * sinfor.order.#     &nbsp;&nbsp;&nbsp;队列绑定的路由键 #代表匹配所有
* queue.sinfor.statistics
  * sinfor.order.add

![mq3](../../images/mq3.png)

> 同时，如果queue.sinfor.credit压力过大，消费起来速度不够及时的话，也可以将其做进一步拆分，如下
* queue.sinfor.credit.orderAdd
  * sinfor.order.add
* queue.sinfor.credit.orderDel
  * sinfor.order.del

> 这种情况下，对积分服务而言，新增订单走新增订单的队列，删除订单走删除订单的队列，消息积压的情况会进一步减小。

![mq3](../../images/mq4.png)


#### 2.3.2 伪代码

> 配置类  


```java
    // 使用topic类型交换机
    @Bean("sinforTopicExchange")
    public Exchange topicExchange() {
        return ExchangeBuilder.topicExchange(TOPIC_EXCHANGE_NAME).durable(true).build();
    }

    
    // 使用匿名队列
    @Bean("topicQueue")
    public Queue topicQueue() {
        return new AnonymousQueue();
    }

    // 所有匹配路由的消息都发送到此交换机上
    @Bean("topicBinding")
    public Binding topicBinding(@Qualifier("sinforTopicExchange") Exchange exchange, @Qualifier("topicQueue") Queue queue) {
        return BindingBuilder.bind(queue).to(exchange).with("sinfor.order.#").noargs();
    }
```


> 生产者 订单服务


```java
    @PostMapping("addOrder")
    public void addOrder() {
        // 订单入库
        Order order = new Order();
        order.setOrderNo(OrderNoGenUtil.gen());
        order.setData(...);
        orderDao.insert(order);

        // 订单消息
        MsgData msg = MsgData.getInstance.setData(order);
        rabbitTemplate.convertAndSend("exchange.sinfor.topic", "sinfor.order.add",  msg);
    }

    @PostMapping("delOrder/{orderId}")
    public Boolean delOrder(@PathVariable Long orderId) {
        // 删除订单
        orderDao.delByOrderId(orderId);

        MsgData msg = MsgData.getInstance.setData(orderId);
        // 删除订单消息
        rabbitTemplate.convertAndSend("exchange.sinfor.topic", "sinfor.order.del",  msg);
        return true;
    }
```


> 积分服务


```java
        // 此队列路由键为sinfor.order.add
        @RabbitListener(queues = "queue.sinfor.credit.orderAdd")
    public void addOrderListener(String message) {
        log.info("-------积分服务收到新增订单消息------:{}", message);
        MsgData msgData = JSONUtil.toBean(message, MsgData.class);

        // 过滤重复消息
        Boolean flag = redisTemplate.opsForValue().setIfAbsent(msgData.getMsgId(), "1", 10, TimeUnit.SECONDS);
        if (flag != null && flag) {
            Order order = (Order) msgData.getData();
            // 积分落库
            ...
        } else {
            log.warn("重复的消息，丢弃掉这条数据");
        }
    }

            // 此队列路由键为sinfor.order.del
        @RabbitListener(queues = "queue.sinfor.credit.orderDel")
    public void addOrderListener(String message) {
        log.info("-------积分服务收到删除订单消息------:{}", message);
        MsgData msgData = JSONUtil.toBean(message, MsgData.class);

        // 过滤重复消息（可以抽取为组件/切面）
        Boolean flag = redisTemplate.opsForValue().setIfAbsent(msgData.getMsgId(), "1", 10, TimeUnit.SECONDS);
        if (flag != null && flag) {
            Long orderId = (Long) msgData.getData();
            // 积分删减
            ...
        } else {
            log.warn("重复的消息，丢弃掉这条数据");
        }
    }
```

> 统计服务


```java
        // 此队列路由键为sinfor.order.add
        @RabbitListener(queues = "queue.sinfor.statistics")
    public void addOrderListener(String message) {
        log.info("-------统计服务收到新增订单消息------:{}", message);
        MsgData msgData = JSONUtil.toBean(message, MsgData.class);

        // 过滤重复消息
        Boolean flag = redisTemplate.opsForValue().setIfAbsent(msgData.getMsgId(), "1", 10, TimeUnit.SECONDS);
        if (flag != null && flag) {
            Order order = (Order) msgData.getData();
            // 统计落库
            ...
        } else {
            log.warn("重复的消息，丢弃掉这条数据");
        }
    }
```

> 此后比如业务扩展，新增的某个服务想监听我们的订单数据变化，只需要新增相对队列(queue.sinfor.xxx)，并做好路由键的映射（sinfor.order.add)后由对应服务编写代码即可，订单服务（生产者）无需再做变更。