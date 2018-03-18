#SpringBoot整合RabbitMQ
[toc]


>
如果你注定要成为厉害的人, 那问题的答案就深藏在你的血脉里。



本篇文章主要讲述Spring Boot与RabbitMQ的整合。因为我们公司的云服务用到了RabbitMQ 技术，之前都是自己封装，正好我们也正在往SpringBoot转变，这个技术正好用到，看来代码又要重构咯。

有想了解重构的朋友，我之前也有对《重构》一书的解读，出门左转就能看到。


----------
导包：

```
<dependency>
<groupId>org.springframework.boot</groupId>
<artifactId>spring-boot-starter-amqp</artifactId>
</dependency>
```


###消息生产者

ConnectionFactory配置
创建AmqpConfig文件AmqpConfig.java(后期的配置都在该文件中)

```
package cn.usr.springbootrabbitmq;

import com.rabbitmq.client.Channel;
import org.springframework.amqp.core.*;
import org.springframework.amqp.rabbit.connection.CachingConnectionFactory;
import org.springframework.amqp.rabbit.core.ChannelAwareMessageListener;
import org.springframework.amqp.rabbit.core.RabbitTemplate;
import org.springframework.amqp.rabbit.listener.SimpleMessageListenerContainer;
import org.springframework.beans.factory.config.ConfigurableBeanFactory;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Scope;

/**
* @program: Learn-SpringBootRabbitmq
* @author: Rock 【shizhiyuan@usr.cn】
* @Date: 2018/2/23 0023
*/
@(SpringBoot)Configuration
public class AmqpConfig {
public static final String EXCHANGE = "spring-boot-exchange2";
public static final String ROUTINGKEY = "spring-boot-routingKey2";


@Bean
public CachingConnectionFactory connectionFactory() {
CachingConnectionFactory connectionFactory = new CachingConnectionFactory();
connectionFactory.setAddresses("127.0.0.1");
connectionFactory.setUsername("guest");
connectionFactory.setPassword("guest");
connectionFactory.setVirtualHost("/");
// 这里需要显示调用才能进行消息的回调 必须要设置
connectionFactory.setPublisherConfirms(true);
return connectionFactory;
}

```

###RabbitTemplate

```
@Bean
@Scope(ConfigurableBeanFactory.SCOPE_PROTOTYPE)
public RabbitTemplate rabbitTemplate() {
RabbitTemplate template = new RabbitTemplate(connectionFactory());
return template;
}
```

这里设置为原型，具体的原因在后面会讲到，在发送消息时通过调用RabbitTemplate中的如下方法：
一会调用的时候用：
```
public void convertAndSend(String exchange, String routingKey, Object object, CorrelationData correlationData)
```

###Producer

调用啦：

```
package cn.usr.springbootrabbitmq;

import org.springframework.amqp.rabbit.core.RabbitTemplate;
import org.springframework.amqp.rabbit.support.CorrelationData;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;

import java.util.UUID;

/**
* @program: Learn-SpringBootRabbitmq
* @author: Rock 【shizhiyuan@usr.cn】
* @Date: 2018/2/23 0023
*/
@Component
public class Producer implements RabbitTemplate.ConfirmCallback {
private RabbitTemplate rabbitTemplate;


/**
* 构造方法注入
*/
@Autowired
public Producer(RabbitTemplate rabbitTemplate) {
this.rabbitTemplate = rabbitTemplate;
//这是是设置回调能收到发送到响应，confirm()在下面解释
rabbitTemplate.setConfirmCallback(this);
}

public void sendMsg(String content) {
CorrelationData correlationId = new CorrelationData(UUID.randomUUID().toString());
//convertAndSend(exchange:交换机名称,routingKey:路由关键字,object:发送的消息内容,correlationData:消息ID)
rabbitTemplate.convertAndSend(AmqpConfig.EXCHANGE, AmqpConfig.ROUTINGKEY, content, correlationId);
}

@Override
public void confirm(CorrelationData correlationData, boolean ack, String cause) {
System.out.println(" 回调id:" + correlationData);
if (ack) {
System.out.println("消息成功消费");
} else {
System.out.println("消息消费失败:" + cause);
}
}
}
```

如果需要在生产者需要消息发送后的回调，需要对rabbitTemplate设置ConfirmCallback对象，由于不同的生产者需要对应不同的ConfirmCallback，如果rabbitTemplate设置为单例bean，则所有的rabbitTemplate实际的ConfirmCallback为最后一次申明的ConfirmCallback。


###消息消费者

还是在`AmqpConfig.class`里面

步骤就是

1. 声明交换机
2. 声明队列
3. 绑定RoutingKey

```
/**
* 针对消费者配置
* 1. 设置交换机类型
* 2. 将队列绑定到交换机
* <p>
* <p>
* FanoutExchange: 将消息分发到所有的绑定队列，无routingkey的概念
* HeadersExchange ：通过添加属性key-value匹配
* DirectExchange:按照routingkey分发到指定队列
* TopicExchange:多关键字匹配
*/
@Bean
public DirectExchange defaultExchange() {
return new DirectExchange(EXCHANGE);
}

@Bean
public Queue queue() {
return new Queue("spring-boot-queue", true);//队列持久
}

@Bean
public Binding binding() {
return BindingBuilder.bind(queue()).to(defaultExchange()).with(AmqpConfig.ROUTINGKEY);
}


@Bean
public SimpleMessageListenerContainer messageContainer() {
SimpleMessageListenerContainer container = new SimpleMessageListenerContainer(connectionFactory());
container.setQueues(queue());
container.setExposeListenerChannel(true);
container.setMaxConcurrentConsumers(1);
container.setConcurrentConsumers(1);
// 设置确认模式手工确认
container.setAcknowledgeMode(AcknowledgeMode.MANUAL);
container.setMessageListener(new ChannelAwareMessageListener() {
@Override
public void onMessage(Message message, Channel channel) throws Exception {
byte[] body = message.getBody();
System.out.println("receive msg : " + new String(body));
//确认消息成功消费
channel.basicAck(message.getMessageProperties().getDeliveryTag(), false);
}
});
return container;
}
```


下面是完整的配置：

```
package cn.usr.springbootrabbitmq;

import com.rabbitmq.client.Channel;
import org.springframework.amqp.core.*;
import org.springframework.amqp.rabbit.connection.CachingConnectionFactory;
import org.springframework.amqp.rabbit.core.ChannelAwareMessageListener;
import org.springframework.amqp.rabbit.core.RabbitTemplate;
import org.springframework.amqp.rabbit.listener.SimpleMessageListenerContainer;
import org.springframework.beans.factory.config.ConfigurableBeanFactory;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Scope;

/**
* @program: Learn-SpringBootRabbitmq
* @author: Rock 【shizhiyuan@usr.cn】
* @Date: 2018/2/23 0023
*/
@Configuration
public class AmqpConfig {
public static final String EXCHANGE = "spring-boot-exchange2";
public static final String ROUTINGKEY = "spring-boot-routingKey2";


@Bean
public CachingConnectionFactory connectionFactory() {
CachingConnectionFactory connectionFactory = new CachingConnectionFactory();
connectionFactory.setAddresses("127.0.0.1");
connectionFactory.setUsername("guest");
connectionFactory.setPassword("guest");
connectionFactory.setVirtualHost("/");
// 这里需要显示调用才能进行消息的回调 必须要设置
connectionFactory.setPublisherConfirms(true);
return connectionFactory;
}

@Bean
@Scope(ConfigurableBeanFactory.SCOPE_PROTOTYPE)
public RabbitTemplate rabbitTemplate() {
RabbitTemplate template = new RabbitTemplate(connectionFactory());
return template;
}


/**
* 针对消费者配置
* 1. 设置交换机类型
* 2. 将队列绑定到交换机
* <p>
* <p>
* FanoutExchange: 将消息分发到所有的绑定队列，无routingkey的概念
* HeadersExchange ：通过添加属性key-value匹配
* DirectExchange:按照routingkey分发到指定队列
* TopicExchange:多关键字匹配
*/
@Bean
public DirectExchange defaultExchange() {
return new DirectExchange(EXCHANGE);
}

@Bean
public Queue queue() {
return new Queue("spring-boot-queue", true);
}

@Bean
public Binding binding() {
return BindingBuilder.bind(queue()).to(defaultExchange()).with(AmqpConfig.ROUTINGKEY);
}


@Bean
public SimpleMessageListenerContainer messageContainer() {
SimpleMessageListenerContainer container = new SimpleMessageListenerContainer(connectionFactory());
container.setQueues(queue());
container.setExposeListenerChannel(true);
container.setMaxConcurrentConsumers(1);
container.setConcurrentConsumers(1);
// 设置确认模式手工确认
container.setAcknowledgeMode(AcknowledgeMode.MANUAL);
container.setMessageListener(new ChannelAwareMessageListener() {
@Override
public void onMessage(Message message, Channel channel) throws Exception {
byte[] body = message.getBody();
System.out.println("receive msg : " + new String(body));
//确认消息成功消费
channel.basicAck(message.getMessageProperties().getDeliveryTag(), false);
}
});
return container;
}
}
```

到这里我就能完成SpringBoot整合RabbitMQ的数据收发了。

结果：
```
receive msg : ceshi-----?
回调id:CorrelationData [id=dfe3b3d1-f5a3-42d9-a514-a73729e009d5]
消息成功消费
```



点赞收藏关注不迷路。么么哒




参考：http://blog.csdn.net/liaokailin/article/details/49559571
