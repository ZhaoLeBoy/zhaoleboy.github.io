---
layout:     post
title:      "RabbitMQ-Exchange(交换机)"
subtitle:   "RabbitMQ-Exchange(交换机)"
date:        2019-05-01
author:     "ZhaoLe"
header-img: "img/mq/rabbit_mq/post-bg.png"
catalog: true
tags:
    - RabbitMQ
---

交换机的功能主要接受消息并且转发到绑定的队列，交换机不存储信息。交换机找不到队列会返回错误，交换机有四种类型：
Direct：默认模式，绑在routing key,只有匹配时，才会被交换器投送到绑定的队列中去。
Topic：按照规则转发消息
Headers：设置 header attribute 参数类型的交换机(暂时没用过)
Fanout：转发消息到所有绑定队列

任何发送到Direct Exchange的消息都会被转发到RouteKey中指定的Queue。

>版本号
* Spring Boot: 2.1.5.RELEASE
* RabbitMQ: 2.1.4.RELEASE

---
>application.properties
```properties
spring.rabbitmq.host=127.0.0.1
spring.rabbitmq.port=5672
spring.rabbitmq.username=guest
spring.rabbitmq.password=guest
```

#### Direct Exchange

`Direct Exchange`是`RabbitMQ`默认的交换器，根据`key`全匹配找队列
#表示匹配0个或若干个关键字，\*表示匹配一个关键字. 

##### 配置代码
```properties
#交换机名称
mq.key.direct-exchange=direct-exchange
#路由器key
mq.direct.route-key=test-direct-route-key
```

```java
@Configuration
public class RabbitConfig {
    @Autowired
    Environment env;

    @Bean
    public Queue directQueue() {
        return new Queue(env.getProperty("mq.direct.route-key"));
    }
    
    /***********Direct(默认 ，可以不配置)***********/
    @Bean
    public DirectExchange directExchange() {
        return new DirectExchange(env.getProperty("mq.key.direct-exchange"));
    }

    @Bean
    public Binding bindExchangeDirectQueue() {
        //根据key 绑定队列到交换机
        return BindingBuilder.bind(directQueue()).to(directExchange()).with(env.getProperty("mq.direct.route-key"));
    }
}
```

##### 消费者代码
```java
@Component
public class DirectConsumer {
    @RabbitListener(queues = "${mq.direct.route-key}")
    public void directListener(String message) {
        System.out.println(message);
    }
}
```

##### 测试代码
```java
@RunWith(SpringRunner.class)
@SpringBootTest(classes = DemoApplication.class)
public class SenderTemp {

    @Autowired
    private AmqpTemplate amqpTemplate;
    @Autowired
    private Environment env;

    static String context_temp = "交换机:%s\n路由：%s\n内容：%s";

    @Test
    public void sendDirectMsg() {
        String context = "Hello,Direct Msg";
        String routeKey = env.getProperty("mq.direct.route-key");
        String exchangeName = env.getProperty("mq.key.direct-exchange");
        String msg = String.format(context_temp, exchangeName, routeKey, context);
        amqpTemplate.convertAndSend(routeKey, msg);
    }
}
```

##### 输出结果
```shell
交换机:direct-exchange
路由：test-direct-route-key
内容：Hello,Direct Msg
```

#### Fanout Exchange

任何发送到`Fanout Exchange`的消息转发到与Exchange绑定的所有队列

##### 配置代码
```properties
mq.key.fanout-exchange=fanout-exchange
mq.fanout.route-key-1=test-fanout-route-key-1
mq.fanout.route-key-2=test-fanout-route-key-2
```

```java
@Configuration
public class RabbitConfig {

    @Autowired
    Environment env;

    @Bean
    public Queue fanoutQueue1() {
        return new Queue(env.getProperty("mq.fanout.route-key-1"));
    }

    @Bean
    public Queue fanoutQueue2() {
        return new Queue(env.getProperty("mq.fanout.route-key-2"));
    }

 /***********Fanout***********/
    @Bean
    public FanoutExchange fanoutExchange() {
        return new FanoutExchange(env.getProperty("mq.key.fanout-exchange"));
    }

    @Bean
    public Binding bindExchangeFanout1Queue() {
        return BindingBuilder.bind(fanoutQueue1()).to(fanoutExchange());
    }

    @Bean
    public Binding bindExchangeFanout2Queue() {
        return BindingBuilder.bind(fanoutQueue2()).to(fanoutExchange());
    }
}
```

##### 消费者代码

```java
@Component
public class Fanout1Consumer {

    @RabbitListener(queues = "${mq.fanout.route-key-1}")
    public void directListener(String message) {
        System.out.println("fanout1-->"+message);
    }
}
```
```java
@Component
public class Fanout2Consumer {
    @RabbitListener(queues = "${mq.fanout.route-key-2}")
    public void directListener(String message) {
        System.out.println("fanout2-->"+message);
    }
}
```

##### 测试代码
```java
@RunWith(SpringRunner.class)
@SpringBootTest(classes = DemoApplication.class)
public class SenderTemp {

    @Autowired
    private AmqpTemplate amqpTemplate;
    @Autowired
    private Environment env;

    static String context_temp = "交换机:%s\n路由：%s\n内容：%s";

    @Test
    public void doFanoutTask() {
        String context = "Hello,Fanout Msg";
        String routeKey = "";
        String exchangeName = env.getProperty("mq.key.fanout-exchange");
        String msg = String.format(context_temp, exchangeName, routeKey, context);
        amqpTemplate.convertAndSend(exchangeName, null, msg);
    }
}
```

##### 输出结果
```java
fanout1-->交换机:fanout-exchange
路由：
内容：Hello,Fanout Msg
fanout2-->交换机:fanout-exchange
路由：
内容：Hello,Fanout Msg
```

#### Topic Exchange
类似模糊匹配，`Exchange`将消息转发到所关注主题能`Route Key`模糊匹配的队列。

##### 配置代码
```properties
mq.key.topic-exchange=topic-exchange
mq.topic.route-key-1=topic.log.select
mq.topic.route-key-2=topic.log.select.new
mq.topic.route-key-3=topic.log
```

```java
@Configuration
public class RabbitConfig {

    @Autowired
    Environment env;
      /***********Topic***********/
    @Bean
    public TopicExchange topicExchange() {
        return new TopicExchange(env.getProperty("mq.key.topic-exchange"));
    }

    @Bean
    public Binding bindExchangeTopic1Queue() {
        return BindingBuilder.bind(topicQueue1()).to(topicExchange()).with("topic.log.select");
    }

    @Bean
    public Binding bindExchangeTopic2Queue() {
        return BindingBuilder.bind(topicQueue2()).to(topicExchange()).with("topic.log.select.#");
    }

    @Bean
    public Binding bindExchangeTopic3Queue() {
        return BindingBuilder.bind(topicQueue3()).to(topicExchange()).with("topic.log.#");
    }

}
```

##### 消费者代码
```java
@Component
public class Topic1Consumer {
    @RabbitListener(queues = "${mq.topic.route-key-1}")
    public void directListener(String message) {
        System.out.println("topic1-->" + message);
    }
}
@Component
public class Topic2Consumer {
    @RabbitListener(queues = "${mq.topic.route-key-2}")
    public void directListener(String message) {
        System.out.println("topic2-->"+message);
    }
}
@Component
public class Topic3Consumer {
    @RabbitListener(queues = "${mq.topic.route-key-3}")
    public void directListener(String message) {
        System.out.println("topic3-->" + message);
    }
}
```
##### 测试代码
```java
 @Test
    public void sendTopic1Msg() {
        String context = "Hello,Topic Msg";
        String routeKey1 = "topic.log.select";
        String exchangeName = env.getProperty("mq.key.topic-exchange");
        String msg = String.format(context_temp, exchangeName, routeKey1, context);
        amqpTemplate.convertAndSend(exchangeName, routeKey1, msg);
    }
    @Test
    public void sendTopic2Msg() {
        String context = "Hello,Topic Msg";
        String routeKey2 = "topic.log.select.new";

        String exchangeName = env.getProperty("mq.key.topic-exchange");
        String msg = String.format(context_temp, exchangeName, routeKey2, context);
        amqpTemplate.convertAndSend(exchangeName, routeKey2, msg);
    }

    @Test
    public void sendTopic3Msg() {
        String context = "Hello,Topic Msg";
        String routeKey3 = "topic.log";

        String exchangeName = env.getProperty("mq.key.topic-exchange");
        String msg = String.format(context_temp, exchangeName, routeKey3, context);
        amqpTemplate.convertAndSend(exchangeName, routeKey3, msg);
    }
```
##### 输出结果
```java

//1
topic3,key:topic.log-->交换机:topic-exchange
路由：topic.log.select
内容：Hello,Topic Msg
topic1,key:topic.log.select-->交换机:topic-exchange
路由：topic.log.select
内容：Hello,Topic Msg
topic2,key:topic.log.select.new-->交换机:topic-exchange
路由：topic.log.select
内容：Hello,Topic Msg

//2
topic2,key:topic.log.select.new-->交换机:topic-exchange
路由：topic.log.select.new
内容：Hello,Topic Msg
topic3,key:topic.log-->交换机:topic-exchange
路由：topic.log.select.new
内容：Hello,Topic Msg

//3
topic3,key:topic.log-->交换机:topic-exchange
路由：topic.log
内容：Hello,Topic Msg
```