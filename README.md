# 1.什么是ark-rocketmq-client？
&emsp;&emsp;ark-rocketmq-client是ark系列框架中的rocketmq-client，基于阿里rocketmq-client组件开发，在rocketmq-client的基础上做了封装和功能的增强。
# 2.ark-rocketmq-client解决了什么问题？
&emsp;&emsp;公司需要使用rocketmq做消息队列进行业务解耦，但公版的rocketmq-client有很多限制，不能满足我们自己的需求。基于对rocketmq的管控，公司自己业务的需求及内部一些中间件串联的需要，因此我们基于公版做了一些定制化改造，如trace-id支持，便捷的消费者和生产者API，公共日志打印等。
# 3.功能列表
- 提供便捷的发送消息和消费消息api
- 支持trace-id，方便排查mq消息问题
- 支持灰度标记，压测标记传递
- 1套mq instance支持多套多环境，通过tag做路由

# 4.接入ark-rocketmq-client
```java
1、配置MQ参数
ark:
  rocketmq:
#   使用模式模式 online 阿里云ons ; open 开源RocketMq
    model: online
    accessKey: 
    secretKey: 
    nameSrvAddr: 
    groupId: GID_
    consumeThreadNums: 10
    #失败最大重试次数，默认：3
    maxReconsumeTimes: 4

2、引入依赖
<dependency>
    <groupId>com.ark.mq</groupId>
    <artifactId>ark-rocketmq-client</artifactId>
    <version>1.0</version>
</dependency>
    

    
3、producer 代码实现
    @Autowired
    private ProducerClient producerClient;

    /**
     * 发送同步普通消息
     * @return
     */
    @PostMapping("/sync")
    public MQSendResult normalSync() throws Exception {
        SendMessage sendMessage = new SendMessage();
        String id = UUID.randomUUID().toString();
        sendMessage.setMsgText("测试发送普通不延迟消息" + id);
        sendMessage.setKeys(id);
        sendMessage.setTopic(TOPIC_SPLIT_APPLY);
        MQSendResult mqSendResult = producerClient.syncSend(sendMessage, null);
        return mqSendResult;
    }

    /**
     * 发送异步的普通消息
     * @return
     */
    @PostMapping("/async")
    public void normalAsync() throws Exception {
        SendMessage sendMessage = new SendMessage();
        String id = UUID.randomUUID().toString();
        sendMessage.setMsgText("测试异步发送普通不延迟消息" + id);
        sendMessage.setKeys(id);
        sendMessage.setTopic(TOPIC_SPLIT_APPLY);
        producerClient.asyncSend(sendMessage, null, new MQSendCallBack() {
            @Override
            public void onSuccess(MQSendResult sendResult) {
              log.info("async receive send message success:{}", JSONObject.toJSONString(sendResult));
            }

            @Override
            public void onException(Throwable context) {
                log.error("async receive send message  error:{}", context);
            }
        });
    }

    
    /**
     * 测试发送同步延迟消息
     * @return
     */
    @PostMapping("/sync/delay")
    public MQSendResult normalSyncDelay() throws Exception {
        SendMessage sendMessage = new SendMessage();
        String id = UUID.randomUUID().toString();
        sendMessage.setMsgText("测试发送普通延迟消息" + id);
        sendMessage.setKeys(id);
        sendMessage.setTopic(TOPIC_SPLIT_APPLY);
        MQSendResult mqSendResult = producerClient.syncSend(sendMessage, 5000l);
        return mqSendResult;
    }


    
    
4、consumer 代码实现

    普通消息/事务消息/延时消息实现：
    @Slf4j
    @Component
    @RocketMQMessageListener(topic = "map_pay_split_apply",expression = "product||order||test")
    public class NormalMessageConsumerDefaultTag implements RocketMQListener<String> {
        
        @Override
        public void onMessage(String message, ConsumeMessage consumeMessage) {
            log.info("NormalMessageConsumerDefaultTag:{}",message);
        }
    }


    顺序消息实现
    expression：代表要监听的tag个数，用"||"分割，注：测试环境不支持*号(监听所有tag)，生产环境支持*号。测试环境：expression为空或写*则代表监听发送消息时不带tag的消息

    @Slf4j
    @Component
    @RocketMQMessageListener(topic = "topic_test_order_msg",expression = "product||order||test||list")
    public  class OrderConsumerMultTag implements OrderRocketMQListener<String> {
        @Override
        public void onMessage(String message, ConsumeMessage consumeMessage) {
          log.info("NormalMessageConsumerDefaultTag:{}",message);
        }
    }


```

