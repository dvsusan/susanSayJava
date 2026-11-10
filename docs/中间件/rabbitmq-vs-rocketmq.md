

## 前言

最近有球友问我：苏三哥，现在一般的项目中的消息中间件，是用**RabbitMQ**，还是**RocketMQ**，更好？

这是一个非常常见的问题。

今天这篇文章就专门跟大家一起聊聊这个话题，希望对你会有所帮助。

## 一、为什么需要消息队列？

有些小伙伴在工作中可能已经用过消息队列，但未必清楚它的核心价值。

让我们先通过一个简单的场景来理解：

**没有消息队列的系统：**
```java
@RestController
public class OrderController {
    
    @Autowired
    private InventoryService inventoryService;
    
    @Autowired
    private PromotionService promotionService;
    
    @Autowired
    private LogService logService;
    
    @PostMapping("/order")
    public Result createOrder(@RequestBody OrderRequest request) {
        // 串行调用多个服务，用户需要等待所有操作完成
        inventoryService.deductStock(request.getProductId(), request.getQuantity());
        promotionService.updatePromotionUsage(request.getUserId(), request.getPromotionId());
        logService.recordOrderLog(request);
        
        return Result.success("下单成功");
    }
}
```

**引入消息队列后：**
```java
@RestController
public class OrderController {
    
    @Autowired
    private OrderService orderService;
    
    @Autowired
    private RocketMQTemplate rocketMQTemplate;
    
    @PostMapping("/order")
    public Result createOrder(@RequestBody OrderRequest request) {
        // 1. 保存订单主记录（核心业务）
        Order order = orderService.saveOrder(request);
        
        // 2. 发送消息，异步处理非核心业务
        OrderCreatedEvent event = new OrderCreatedEvent(order);
        rocketMQTemplate.send("order-topic", event);
        
        // 3. 立即返回成功
        return Result.success("订单创建成功，处理中");
    }
}

// 消费者异步处理
@Component
@RocketMQMessageListener(topic = "order-topic", consumerGroup = "order-group")
public class OrderConsumer implements RocketMQListener<OrderCreatedEvent> {
    
    @Override
    public void onMessage(OrderCreatedEvent event) {
        // 并行执行多个操作，但用户无需等待
        inventoryService.deductStock(event.getProductId(), event.getQuantity());
        promotionService.updatePromotionUsage(event.getUserId(), event.getPromotionId());
        logService.recordOrderLog(event);
    }
}
```

消息队列的核心价值在于：**解耦、异步、削峰**。

但不同的消息队列在实现这些价值时，侧重点各不相同。

其实咱们的星球中的[商城微服务系统、秒杀系统、SaaS点餐系统](https://mp.weixin.qq.com/mp/wappoc_appmsgcaptcha?poc_token=HBdrtmmju_hXmiFlibzBQNjE59MYqGTDaEXD3pCz&target_url=https%3A%2F%2Fmp.weixin.qq.com%2Fs%3F__biz%3DMzkwNjMwMTgzMQ%3D%3D%26mid%3D2247535401%26idx%3D1%26sn%3D3d9596cc07051e827bf7024679d3f462%26scene%3D21#wechat_redirect)就大量使用了MQ实现相关业务逻辑。

比如：异步海量数据导出功能、WebSocket消息实时通信功能、商品秒杀功能、订单超时自动取消功能等等。


## 二、核心特性对比

让我们先通过一个表格直观感受两者的核心差异：

| 对比维度 | RabbitMQ | RocketMQ |
|---------|----------|----------|
| **吞吐量(TPS)** | 5-10K  | 50K+  |
| **延迟(ms)** | 50-100  | 20-50  |
| **协议支持** | AMQP、MQTT、STOMP等7种协议 | 自定义TCP协议为主 |
| **路由机制** | 4种Exchange，灵活路由 | Topic模型，相对简单 |
| **事务消息** | 不支持 | 支持  |
| **定时/延时消息** | 插件支持 | 原生支持，秒级精度  |
| **顺序消息** | 单队列保证 | 全局/分区顺序支持  |
| **开发语言** | Erlang | Java |
| **社区活跃度** | 极高，生态丰富 | 较高，阿里生态为主  |

## 三、RabbitMQ解析

### 3.1 RabbitMQ的核心优势

有些小伙伴可能对RabbitMQ的灵活路由机制印象深刻。确实，这是RabbitMQ最强大的特性之一。

**RabbitMQ的四种交换机类型：**


![RabbitMQ的核心优势 示意图1](https://web-1259428203.cos.ap-chengdu.myqcloud.com/e0db618e-5c9a-4493-8e84-f5ab3973071d.png)

**代码示例：Topic交换机实现复杂路由**
```java
// RabbitMQ配置类
@Configuration
public class RabbitMQConfig {
    
    @Bean
    public TopicExchange orderExchange() {
        return new TopicExchange("order.exchange");
    }
    
    @Bean
    public Queue paymentQueue() {
        return QueueBuilder.durable("payment.queue").build();
    }
    
    @Bean
    public Queue inventoryQueue() {
        return QueueBuilder.durable("inventory.queue").build();
    }
    
    @Bean
    public Binding paymentBinding() {
        return BindingBuilder.bind(paymentQueue())
                .to(orderExchange())
                .with("order.payment.*");
    }
    
    @Bean
    public Binding inventoryBinding() {
        return BindingBuilder.bind(inventoryQueue())
                .to(orderExchange())
                .with("order.inventory.*");
    }
}

// 生产者
@Service
public class OrderProducer {
    
    @Autowired
    private RabbitTemplate rabbitTemplate;
    
    public void sendOrderEvent(String eventType, OrderEvent event) {
        // routingKey可以是 order.payment.success 或 order.inventory.update
        rabbitTemplate.convertAndSend("order.exchange", 
                                      "order." + eventType, 
                                      event);
    }
}

// 消费者 - 支付服务
@Component
@RabbitListener(queues = "payment.queue")
public class PaymentConsumer {
    
    @RabbitHandler
    public void handlePayment(OrderEvent event) {
        System.out.println("支付服务处理：" + event);
    }
}

// 消费者 - 库存服务
@Component
@RabbitListener(queues = "inventory.queue")
public class InventoryConsumer {
    
    @RabbitHandler
    public void handleInventory(OrderEvent event) {
        System.out.println("库存服务处理：" + event);
    }
}
```

RabbitMQ的另一个优势是**多协议支持**。它原生支持AMQP 0-9-1、AMQP 1.0、MQTT、STOMP等多种协议，这在物联网、移动端等异构系统集成时非常有用。

### 3.2 RabbitMQ的局限性

虽然RabbitMQ灵活易用，但它也存在一些明显的短板：

1. **性能瓶颈**：单节点吞吐量约5-10K TPS，在百万级消息/秒场景下，Erlang虚拟机可能成为性能瓶颈。
2. **集群扩展复杂**：依赖镜像队列实现高可用，增加存储开销，网络分区处理较为复杂。
3. **顺序消息困难**：默认不保证跨消费者的消息顺序。
4. **事务机制不完善**：RabbitMQ的事务机制会极大影响性能，官方也不推荐使用。

### 3.3 RabbitMQ适用场景

**推荐使用场景**：
- 中小规模微服务系统（消息量<10万级/秒）
- 需要复杂路由逻辑的业务（如订单状态变更通知）
- 多协议接入场景（IoT设备、移动端、WebSocket）
- 对可靠性要求高于性能的企业级应用

**不推荐场景**：
- 超大规模日志处理（首选Kafka）
- 严格顺序要求的金融交易（考虑RocketMQ）
- 极低延迟需求的高频交易场景

## 四、RocketMQ解析

### 4.1 RocketMQ的核心优势

RocketMQ是阿里巴巴开源的分布式消息中间件，在电商、金融等大规模分布式系统中表现优异。

**RocketMQ的高可用架构：**


![RocketMQ的核心优势 示意图2](https://web-1259428203.cos.ap-chengdu.myqcloud.com/40e56edc-c46f-4d51-a6cb-9a336fe890a9.png)


**RocketMQ的核心特性代码示例：**

#### 1. 事务消息（RocketMQ的王牌特性）

```java
@Component
public class TransactionOrderService {
    
    @Autowired
    private RocketMQTemplate rocketMQTemplate;
    
    @Autowired
    private OrderMapper orderMapper;
    
    // 发送事务消息
    public void createOrderWithTransaction(OrderRequest request) {
        OrderMessage orderMessage = new OrderMessage(request);
        
        TransactionSendResult result = rocketMQTemplate.sendMessageInTransaction(
            "order-tx-group",
            "order-topic",
            MessageBuilder.withPayload(orderMessage).build(),
            request // 传递给本地事务执行器的参数
        );
        
        if (result.getLocalTransactionState() == LocalTransactionState.COMMIT_MESSAGE) {
            System.out.println("事务提交成功");
        }
    }
    
    // 本地事务执行器
    @RocketMQTransactionListener(txProducerGroup = "order-tx-group")
    public class OrderTransactionListener implements RocketMQLocalTransactionListener {
        
        @Override
        public RocketMQLocalTransactionState executeLocalTransaction(Message msg, Object arg) {
            OrderRequest request = (OrderRequest) arg;
            
            try {
                // 1. 执行本地事务 - 保存订单
                orderMapper.insert(request);
                
                // 2. 返回提交状态
                return RocketMQLocalTransactionState.COMMIT;
            } catch (Exception e) {
                // 3. 异常时回滚
                return RocketMQLocalTransactionState.ROLLBACK;
            }
        }
        
        @Override
        public RocketMQLocalTransactionState checkLocalTransaction(Message msg) {
            // 事务回查 - 解决分布式事务的最终一致性
            OrderMessage orderMessage = (OrderMessage) msg.getPayload();
            
            // 查询本地事务是否成功
            Order order = orderMapper.selectById(orderMessage.getOrderId());
            
            if (order != null) {
                return RocketMQLocalTransactionState.COMMIT;
            } else {
                return RocketMQLocalTransactionState.ROLLBACK;
            }
        }
    }
}
```

#### 2. 精确到秒级的定时消息

RocketMQ 5.x版本引入了**秒级定时消息**，这是通过时间轮算法实现的：

```java
@Service
public class DelayMessageService {
    
    @Autowired
    private RocketMQTemplate rocketMQTemplate;
    
    // 发送定时消息（精确到秒）
    public void sendDelayMessage(MessageData data, long delaySeconds) {
        long delayTimestamp = System.currentTimeMillis() + delaySeconds * 1000;
        
        Message<MessageData> message = MessageBuilder.withPayload(data)
                .setHeader("DELAY_TIME", delayTimestamp)
                .build();
        
        rocketMQTemplate.syncSend("delay-topic", message);
    }
    
    // 消费定时消息
    @Service
    @RocketMQMessageListener(topic = "delay-topic", consumerGroup = "delay-group")
    public class DelayMessageConsumer implements RocketMQListener<MessageData> {
        
        @Override
        public void onMessage(MessageData message) {
            System.out.println("定时消息到达：" + message);
        }
    }
}
```

RocketMQ的定时消息底层通过**时间轮（TimerWheel）+ TimerLog**实现：


![精确到秒级的定时消息 示意图3](https://web-1259428203.cos.ap-chengdu.myqcloud.com/0410261e-13c6-4a97-b9af-bceb5233f60f.png)


这种设计保证了即使消息延迟时间很长（如半个月），也不会丢失。

#### 3. 顺序消息实现

```java
@Service
public class OrderlyMessageService {
    
    @Autowired
    private RocketMQTemplate rocketMQTemplate;
    
    // 发送顺序消息 - 按订单ID路由
    public void sendOrderlyMessage(String orderId, OrderEvent event) {
        // 通过MessageQueueSelector将同一订单的消息发送到同一个队列
        rocketMQTemplate.syncSendOrderly(
            "order-topic", 
            event, 
            orderId  // 选择key
        );
    }
    
    // 顺序消费
    @Service
    @RocketMQMessageListener(
        topic = "order-topic",
        consumerGroup = "orderly-group",
        consumeMode = ConsumeMode.ORDERLY  // 顺序消费模式
    )
    public class OrderlyConsumer implements RocketMQListener<OrderEvent> {
        
        @Override
        public void onMessage(OrderEvent event) {
            // 保证同一订单的消息串行处理
            System.out.println("顺序处理订单事件：" + event);
        }
    }
}
```

### 4.2 RocketMQ的局限性

1. **协议封闭**：主要支持自定义TCP协议，多语言客户端支持较弱
2. **社区生态**：相比RabbitMQ，生态规模较小，主要在阿里生态中活跃
3. **学习曲线**：概念较多（NameServer、Broker、Queue等），入门门槛略高
4. **运维复杂度**：需要部署NameServer和Broker集群，运维成本相对较高

### 4.3 RocketMQ适用场景

**推荐使用场景**：
- 电商交易系统（订单、支付、库存）
- 金融核心系统（需要事务一致性）
- 大规模分布式系统（消息量>10万级/秒）
- 需要严格顺序消息的业务（如交易流水）
- 需要定时/延时消息的场景

**不推荐场景**：
- 多协议接入需求（首选RabbitMQ）
- 简单的内部系统（Kafka或RabbitMQ更轻量）
- 非Java技术栈为主的项目

## 五、如何选型？

### 5.1 决策树


![决策树 示意图4](https://web-1259428203.cos.ap-chengdu.myqcloud.com/710b9e32-33bf-4732-abad-23302ff00df2.png)


### 5.2 关键决策因素

**选择RabbitMQ当**：
- 需要复杂消息路由（Topic、Direct、Fanout等）
- 需要支持多种协议（MQTT、AMQP、STOMP）
- 团队熟悉Erlang或对运维复杂度敏感
- 消息量适中（<10万TPS）

**选择RocketMQ当**：
- 需要分布式事务支持
- 需要严格顺序消息
- 需要精准定时/延时消息
- 系统规模较大（>10万TPS）
- Java技术栈为主

### 5.3 混合架构方案

在实际项目中，完全可以采用混合架构：

```yaml
架构方案：
  - 主干消息: RocketMQ
    用途: 核心交易、订单、支付
    理由: 事务保证、高吞吐
    
  - 分支消息: RabbitMQ
    用途: 通知推送、日志收集、IoT设备
    理由: 多协议支持、灵活路由
    
  - 监控体系:
    - Prometheus + Grafana
    - 队列积压监控
    - 消费延迟告警
```

## 总结

通过本文我们可以得出以下结论：

1. **RabbitMQ**：胜在灵活性和协议兼容性，适合企业级应用集成、复杂路由场景。
2. **RocketMQ**：胜在功能全面和性能优势，特别适合电商、金融等需要事务保证和顺序消息的场景。
3. **没有绝对的优劣，只有最适合的选择**。选择消息队列时，需要综合考虑：
   - 业务需求（事务、顺序、定时）
   - 性能要求（吞吐量、延迟）
   - 技术栈（Java/非Java）
   - 运维能力
4. **关注技术演进**：RocketMQ 5.x在定时消息、高可用等方面持续创新；RabbitMQ也在引入Quorum Queues等增强一致性。建议每18个月重新评估技术栈。

最后，送给大家一句话：**选型不是终点，理解才是关键**。

只有深入理解消息队列的原理，才能在遇到问题时游刃有余。

如果你有任何问题或经验分享，欢迎在评论区交流讨论！

<!-- article-related-reading:start -->
## 相关阅读

- [RocketMQ已正式接入AI！](./rocketmq-embraces-ai.html)
- [我工作中用MQ的10种场景](./10-mq-use-cases-in-my-work.html)
- [解决MQ消息丢失问题的5种方案](./5-solutions-for-mq-message-loss.html)

<!-- article-related-reading:end -->
