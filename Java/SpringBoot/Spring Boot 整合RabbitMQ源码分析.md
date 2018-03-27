# Spring Boot 整合RabbitMQ源码分析

***
[toc]
***

@(SpringBoot)

*** 

本篇说明Spring Boot如何整合RabbitMQ，不明白的朋友请看-->[SpringBoot怎么整合RabbitMQ的？](https://github.com/AmbitionLofty/Blog/blob/master/Java/SpringBoot/SpringBoot%E6%95%B4%E5%90%88RabbitMQ.md)



## RabbitAdmin

看下该部分代码:

```java
public AmqpAdmin amqpAdmin(CachingConnectionFactory connectionFactory) {  
        return new RabbitAdmin(connectionFactory);  
    }  
```
创建`RabbitAdmin`实例，调用构造方法:

```java
public RabbitAdmin(ConnectionFactory connectionFactory) {  
    this.connectionFactory = connectionFactory;  
    Assert.notNull(connectionFactory, "ConnectionFactory must not be null");  
    this.rabbitTemplate = new RabbitTemplate(connectionFactory);  
}  
```

创建连接工厂、rabbitTemplate，其中ConnectionFactory采用上一篇中自定义bean:

```java
public ConnectionFactory connectionFactory() {  
     CachingConnectionFactory connectionFactory = new CachingConnectionFactory();  
     connectionFactory.setAddresses("127.0.0.1:5672");  
     connectionFactory.setUsername("guest");  
     connectionFactory.setPassword("guest");  
     connectionFactory.setPublisherConfirms(true); //必须要设置  
     return connectionFactory;  
 }  
```

为`CachingConnectionFactory`实例，其缓存模式为通道缓存
```java
private volatile CacheMode cacheMode = CacheMode.CHANNEL;  
```
接下来看下`RabbitAdmin`类定义：
```java
public class RabbitAdmin implements AmqpAdmin, ApplicationContextAware, InitializingBean {  
...  
}  
```
实现接口`AmqpAdmin`(定义若干RabbitMQ操作父接口),这里需要强调的是`InitializingBean`，实现该接口则会调用`afterPropertiesSet`方法:

```java
public void afterPropertiesSet() {  
  
        synchronized (this.lifecycleMonitor) {  
  
            if (this.running || !this.autoStartup) {  
                return;  
            }  
  
            if (this.connectionFactory instanceof CachingConnectionFactory &&  
                    ((CachingConnectionFactory) this.connectionFactory).getCacheMode() == CacheMode.CONNECTION) {  
                logger.warn("RabbitAdmin auto declaration is not supported with CacheMode.CONNECTION");  
                return;  
            }  
  
            this.connectionFactory.addConnectionListener(new ConnectionListener() {  
  
                // Prevent stack overflow...  
                private final AtomicBoolean initializing = new AtomicBoolean(false);  
  
                @Override  
                public void onCreate(Connection connection) {  
                    if (!initializing.compareAndSet(false, true)) {  
                        // If we are already initializing, we don't need to do it again...  
                        return;  
                    }  
                    try {  
                           
                        initialize();  
                    }  
                    finally {  
                        initializing.compareAndSet(true, false);  
                    }  
                }  
  
                @Override  
                public void onClose(Connection connection) {  
                }  
  
            });  
  
            this.running = true;  
  
        }  
    }  
```
`synchronized (this.lifecycleMonitor)`加锁保证同一时间只有一个线程访问该代码，随后调用`this.connectionFactory.addConnectionListener`添加连接监听，各连接工厂关系:

实际调用为CachingConnectionFactory

```java
public void addConnectionListener(ConnectionListener listener) {  
        super.addConnectionListener(listener);  
        // If the connection is already alive we assume that the new listener wants to be notified  
        if (this.connection != null) {  
            listener.onCreate(this.connection);  
        }  
    } 
```


此时connection为null，无法执行到`listener.onCreate(this.connection);`
 往`CompositeConnectionListener connectionListener`中添加监听信息，最终保证在集合中:

```java
private List<ConnectionListener> delegates = new CopyOnWriteArrayList<ConnectionListener>();  
```

这里添加的监听代码执行，在后面调用时再来讲解。

至此--- `RabbitAdmin`创建完成。 

## Exchange

接下来继续来看`AmqpConfig.java`中的代码:

```java
  @Bean  
  public DirectExchange defaultExchange() {  
      return new DirectExchange(EXCHANGE);  
  }  
```
以上代码创建一个交换机，交换机类型为 **direct(直连)**
在申明交换机时需要指定交换机名称，默认创建可持久交换机；


## Queue
默认创建可持久队列：

```java
public Queue queue() {  
       return new Queue("spring-boot-queue", true); //队列持久  
   }  
```

## Binding


```java
   @Bean  
   public Binding binding() {  
       return BindingBuilder.bind(queue()).to(defaultExchange()).with(AmqpConfig.ROUTINGKEY);  
   }  
```

**`BindingBuilder.bind(queue())` 实现为：**
```java
public static DestinationConfigurer bind(Queue queue) {  
        return new DestinationConfigurer(queue.getName(), DestinationType.QUEUE);  
    } 
```

`DestinationConfigurer`通过`name、type`区分不同配置信息，其`to()`方法为重载方法，传递参数为四种交换机，分别返回`XxxExchangeRoutingKeyConfigurer`,其中`with`方法返回`Bingding`实例，因此在Binding信息中存储了
**队列、交换机、路由key等相关信息**

```java
public class Binding extends AbstractDeclarable {  
  
    public static enum DestinationType {  
        QUEUE, EXCHANGE;  
    }  
  
    private final String destination;  
  
    private final String exchange;  
  
    private final String routingKey;  
  
    private final Map<String, Object> arguments;  
  
    private final DestinationType destinationType;  
...  
}  
```


##  SimpleMessageListenerContainer

```java
    @Bean  
    public SimpleMessageListenerContainer messageContainer() {  
        SimpleMessageListenerContainer container = new SimpleMessageListenerContainer(connectionFactory());  
        container.setQueues(queue());  
        container.setExposeListenerChannel(true);  
        container.setMaxConcurrentConsumers(1);  
        container.setConcurrentConsumers(1);  
        container.setAcknowledgeMode(AcknowledgeMode.MANUAL); //设置确认模式手工确认  
        container.setMessageListener(new ChannelAwareMessageListener() {  
  
            @Override  
            public void onMessage(Message message, Channel channel) throws Exception {  
                byte[] body = message.getBody();  
                System.out.println("receive msg : " + new String(body));  
                channel.basicAck(message.getMessageProperties().getDeliveryTag(), false); //确认消息成功消费  
            }  
        });  
        return container;  
    }  
```

查看其实现的接口，注意**`SmartLifecycle`**：

接下来设置队列信息，在`AbstractMessageListenerContainer`

```java
private volatile List<String> queueNames = new CopyOnWriteArrayList<String>();  
```

 添加队列信息
`AbstractMessageListenerContainer.exposeListenerChannel`设置为true


```java
container.setMaxConcurrentConsumers(1);  
container.setConcurrentConsumers(1);  
```

 设置并发消费者数量，默认情况为`1`:
 
```java
container.setAcknowledgeMode(AcknowledgeMode.MANUAL); //设置确认模式手工确认
```
 设置消费者成功消费消息后确认模式，分为两种:
 
- 自动模式，默认模式，在RabbitMQ Broker消息发送到消费者后自动删除
- 手动模式，消费者客户端显示编码确认消息消费完成，Broker给生产者发送回调，消息删除


接下来设置消费者端消息监听，为`privatevolatile ObjectmessageListener` 赋值；

到这里消息监听容器也创建完成了，**但令人纳闷的时，消费者如何去消费消息呢？**从这里完全看不出来。那么接下来看下**`SmartLifecycle`接口**：



## SmartLifecycle

熟悉Spring都应该知道该接口，其定义为：

```java
public interface SmartLifecycle extends Lifecycle, Phased {  
  
    boolean isAutoStartup();  
    void stop(Runnable callback);  
  
}  
```

其中的`isAutoStartup`设置为`true`时，会自动调用Lifecycle接口中的start方法，既然我们为源码分析，也简单看下这个聪明的声明周期接口是如何实现它的聪明方法的：


在讲到执行Bean加载时，调用`AbstractApplicationContext.refresh()`，其中存在一个方法调用`finishRefresh()`

```java
protected void finishRefresh() {  
    // Initialize lifecycle processor for this context.  
    initLifecycleProcessor();  
  
    // Propagate refresh to lifecycle processor first.  
    getLifecycleProcessor().onRefresh();  
  
    // Publish the final event.  
    publishEvent(new ContextRefreshedEvent(this));  
  
    // Participate in LiveBeansView MBean, if active.  
    LiveBeansView.registerApplicationContext(this);  
}  
```

其中`initLifecycleProcessor`初始化生命周期处理器，
```java
protected void initLifecycleProcessor() {  
    ConfigurableListableBeanFactory beanFactory = getBeanFactory();  
    if (beanFactory.containsLocalBean(LIFECYCLE_PROCESSOR_BEAN_NAME)) {  
        this.lifecycleProcessor =  
                beanFactory.getBean(LIFECYCLE_PROCESSOR_BEAN_NAME, LifecycleProcessor.class);  
        if (logger.isDebugEnabled()) {  
            logger.debug("Using LifecycleProcessor [" + this.lifecycleProcessor + "]");  
        }  
    }  
    else {  
        DefaultLifecycleProcessor defaultProcessor = new DefaultLifecycleProcessor();  
        defaultProcessor.setBeanFactory(beanFactory);  
        this.lifecycleProcessor = defaultProcessor;  
        beanFactory.registerSingleton(LIFECYCLE_PROCESSOR_BEAN_NAME, this.lifecycleProcessor);  
        if (logger.isDebugEnabled()) {  
            logger.debug("Unable to locate LifecycleProcessor with name '" +  
                    LIFECYCLE_PROCESSOR_BEAN_NAME +  
                    "': using default [" + this.lifecycleProcessor + "]");  
        }  
    }  
}  
```

注册`DefaultLifecycleProcessor`对应bean
`getLifecycleProcessor().onRefresh()`调用`DefaultLifecycleProcessor`中方法`onRefresh`，调用`startBeans(true)`;


```java
Map<String, Lifecycle> lifecycleBeans = getLifecycleBeans();
```

获取所有实现**`Lifecycle`接口bean**，执行`bean instanceof SmartLifecycle && ((SmartLifecycle) bean).isAutoStartup()`判断，如果bean同时也为`Phased`实例，则加入到`LifecycleGroup`中，随后`phases.get(key).start()`调用start方法;



接下来要做的事情就很明显：要了解消费者具体如何实现，查看SimpleMessageListenerContainer中的start是如何实现的。


*** 

OVER


***
[原文](https://blog.csdn.net/liaokailin/article/details/49562651)