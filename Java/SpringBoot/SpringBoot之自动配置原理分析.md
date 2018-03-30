# SpringBoot之自动配置原理分析

***
[toc]


@(SpringBoot)

*** 

Spring Boot中引入了自动配置，让开发者利用起来更加的简便、快捷，本篇讲利用RabbitMQ的自动配置为例讲分析下Spring Boot中的自动配置原理。

在上一篇末尾讲述了Spring Boot 默认情况下会为ConnectionFactory、RabbitTemplate等bean，在前面的文章中也讲到嵌入的Tomcat默认配置为8080端口

这些都属于Spring Boot自动配置的范畴，当然其自动配置相当多;


## EnableAutoConfiguration注解


**在创建Application时我们使用了SpringBootApplication注解**，再来看下其定义：

```java

@Target(ElementType.TYPE)  
@Retention(RetentionPolicy.RUNTIME)  
@Documented  
@Inherited  
@Configuration  
@EnableAutoConfiguration  
@ComponentScan  
public @interface SpringBootApplication {　　  
    Class<?>[] exclude() default {};  
  
} 
```

该注解上存在元注解`@EnableAutoConfiguration`，这就是Spring Boot自动配置实现的核心入口；其定义为：


```java
@Target(ElementType.TYPE)  
@Retention(RetentionPolicy.RUNTIME)  
@Documented  
@Inherited  
@Import({ EnableAutoConfigurationImportSelector.class,  
        AutoConfigurationPackages.Registrar.class })  
public @interface EnableAutoConfiguration {  
  
    /**  
     * Exclude specific auto-configuration classes such that they will never be applied.  
     * @return the classes to exclude  
     */  
    Class<?>[] exclude() default {};  
  
}  

```

很显然能看出有一特殊的注解`@Import`，加载bean时会解析Import注解，因此需要将目光聚集在这段代码:

```
@Import({ EnableAutoConfigurationImportSelector.class,  
        AutoConfigurationPackages.Registrar.class })  
```


## EnableAutoConfigurationImportSelector


来看`EnableAutoConfigurationImportSelector`类: 


```java

public String[] selectImports(AnnotationMetadata metadata) {  
        try {  
            AnnotationAttributes attributes = AnnotationAttributes.fromMap(metadata  
                    .getAnnotationAttributes(EnableAutoConfiguration.class.getName(),  
                            true));  
  
            Assert.notNull(attributes, "No auto-configuration attributes found. Is "  
                    + metadata.getClassName()  
                    + " annotated with @EnableAutoConfiguration?");  
  
            // Find all possible auto configuration classes, filtering duplicates  
            // 找到所有可能的自动配置类，过滤重复项
            List<String> factories = new ArrayList<String>(new LinkedHashSet<String>(  
                    SpringFactoriesLoader.loadFactoryNames(EnableAutoConfiguration.class,  
                            this.beanClassLoader)));  
  
            // Remove those specifically disabled  
            // 删除那些明确禁用的
            factories.removeAll(Arrays.asList(attributes.getStringArray("exclude")));  
  
            // Sort  
            // 分类
            factories = new AutoConfigurationSorter(this.resourceLoader)  
                    .getInPriorityOrder(factories);  
  
            return factories.toArray(new String[factories.size()]);  
        }  
        catch (IOException ex) {  
            throw new IllegalStateException(ex);  
        }  
    }  
```

看如下代码，获取类路径下`spring.factories`下key为`EnableAutoConfiguration`全限定名对应值:

```java
SpringFactoriesLoader.loadFactoryNames(EnableAutoConfiguration.class,  
                            this.beanClassLoader))  
```

其结果为：
```
# Auto Configure  
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\  
org.springframework.boot.autoconfigure.aop.AopAutoConfiguration,\  
org.springframework.boot.autoconfigure.amqp.RabbitAutoConfiguration,\  
org.springframework.boot.autoconfigure.MessageSourceAutoConfiguration,\  
org.springframework.boot.autoconfigure.PropertyPlaceholderAutoConfiguration,\  
org.springframework.boot.autoconfigure.batch.BatchAutoConfiguration,\  
org.springframework.boot.autoconfigure.cloud.CloudAutoConfiguration,\  
org.springframework.boot.autoconfigure.dao.PersistenceExceptionTranslationAutoConfiguration,\  
org.springframework.boot.autoconfigure.data.elasticsearch.ElasticsearchRepositoriesAutoConfiguration,\  
org.springframework.boot.autoconfigure.data.jpa.JpaRepositoriesAutoConfiguration,\  
org.springframework.boot.autoconfigure.data.mongo.MongoRepositoriesAutoConfiguration,\  
org.springframework.boot.autoconfigure.data.solr.SolrRepositoriesAutoConfiguration,\  
org.springframework.boot.autoconfigure.data.rest.RepositoryRestMvcAutoConfiguration,\  
org.springframework.boot.autoconfigure.data.web.SpringDataWebAutoConfiguration,\  
org.springframework.boot.autoconfigure.freemarker.FreeMarkerAutoConfiguration,\  
org.springframework.boot.autoconfigure.gson.GsonAutoConfiguration,\  
org.springframework.boot.autoconfigure.hateoas.HypermediaAutoConfiguration,\  
org.springframework.boot.autoconfigure.integration.IntegrationAutoConfiguration,\  
org.springframework.boot.autoconfigure.jackson.JacksonAutoConfiguration,\  
org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration,\  
org.springframework.boot.autoconfigure.jdbc.JndiDataSourceAutoConfiguration,\  
org.springframework.boot.autoconfigure.jdbc.XADataSourceAutoConfiguration,\  
org.springframework.boot.autoconfigure.jdbc.DataSourceTransactionManagerAutoConfiguration,\  
org.springframework.boot.autoconfigure.jms.JmsAutoConfiguration,\  
org.springframework.boot.autoconfigure.jmx.JmxAutoConfiguration,\  
org.springframework.boot.autoconfigure.jms.JndiConnectionFactoryAutoConfiguration,\  
org.springframework.boot.autoconfigure.jms.activemq.ActiveMQAutoConfiguration,\  
org.springframework.boot.autoconfigure.jms.hornetq.HornetQAutoConfiguration,\  
org.springframework.boot.autoconfigure.jta.JtaAutoConfiguration,\  
org.springframework.boot.autoconfigure.elasticsearch.ElasticsearchAutoConfiguration,\  
org.springframework.boot.autoconfigure.elasticsearch.ElasticsearchDataAutoConfiguration,\  
org.springframework.boot.autoconfigure.flyway.FlywayAutoConfiguration,\  
org.springframework.boot.autoconfigure.groovy.template.GroovyTemplateAutoConfiguration,\  
org.springframework.boot.autoconfigure.jersey.JerseyAutoConfiguration,\  
org.springframework.boot.autoconfigure.liquibase.LiquibaseAutoConfiguration,\  
org.springframework.boot.autoconfigure.mail.MailSenderAutoConfiguration,\  
org.springframework.boot.autoconfigure.mobile.DeviceResolverAutoConfiguration,\  
org.springframework.boot.autoconfigure.mobile.DeviceDelegatingViewResolverAutoConfiguration,\  
org.springframework.boot.autoconfigure.mobile.SitePreferenceAutoConfiguration,\  
org.springframework.boot.autoconfigure.mongo.MongoAutoConfiguration,\  
org.springframework.boot.autoconfigure.mongo.MongoDataAutoConfiguration,\  
org.springframework.boot.autoconfigure.mustache.MustacheAutoConfiguration,\  
org.springframework.boot.autoconfigure.orm.jpa.HibernateJpaAutoConfiguration,\  
org.springframework.boot.autoconfigure.reactor.ReactorAutoConfiguration,\  
org.springframework.boot.autoconfigure.redis.RedisAutoConfiguration,\  
org.springframework.boot.autoconfigure.security.SecurityAutoConfiguration,\  
org.springframework.boot.autoconfigure.security.FallbackWebSecurityAutoConfiguration,\  
org.springframework.boot.autoconfigure.social.SocialWebAutoConfiguration,\  
org.springframework.boot.autoconfigure.social.FacebookAutoConfiguration,\  
org.springframework.boot.autoconfigure.social.LinkedInAutoConfiguration,\  
org.springframework.boot.autoconfigure.social.TwitterAutoConfiguration,\  
org.springframework.boot.autoconfigure.solr.SolrAutoConfiguration,\  
org.springframework.boot.autoconfigure.velocity.VelocityAutoConfiguration,\  
org.springframework.boot.autoconfigure.thymeleaf.ThymeleafAutoConfiguration,\  
org.springframework.boot.autoconfigure.web.DispatcherServletAutoConfiguration,\  
org.springframework.boot.autoconfigure.web.EmbeddedServletContainerAutoConfiguration,\  
org.springframework.boot.autoconfigure.web.ErrorMvcAutoConfiguration,\  
org.springframework.boot.autoconfigure.web.GzipFilterAutoConfiguration,\  
org.springframework.boot.autoconfigure.web.HttpEncodingAutoConfiguration,\  
org.springframework.boot.autoconfigure.web.HttpMessageConvertersAutoConfiguration,\  
org.springframework.boot.autoconfigure.web.MultipartAutoConfiguration,\  
org.springframework.boot.autoconfigure.web.ServerPropertiesAutoConfiguration,\  
org.springframework.boot.autoconfigure.web.WebMvcAutoConfiguration,\  
org.springframework.boot.autoconfigure.websocket.WebSocketAutoConfiguration  

```

以上为Spring Boot中所有的自动配置相关类；在启动过程中会解析对应类配置信息，以RabbitMQ为例，则会去解析`RabbitAutoConfiguration`
```java

```



## RabbitAutoConfiguration

首先来看`RabbitAutoConfiguration`类上的注解：

　

```java
@Configuration  
@ConditionalOnClass({ RabbitTemplate.class, Channel.class })  
@EnableConfigurationProperties(RabbitProperties.class)  
@Import(RabbitAnnotationDrivenConfiguration.class)  
public class RabbitAutoConfiguration { 
```
- `@Configuration`: 应该不需要解释
- `@ConditionalOnClass`：表示存在对应的Class文件时才会去解析`RabbitAutoConfiguration`，否则直接跳过不解析，**这也是为什么在不导入RabbitMQ依赖Jar时工程能正常启动的原因**

- `@EnableConfigurationProperties`：表示对`@ConfigurationProperties`的内嵌支持，默认会将对应Class这是为bean，例如这里值为`RabbitProperties`.class，其定义为：


```java
@ConfigurationProperties(prefix = "spring.rabbitmq")  
public class RabbitProperties {  
  
    /**  
     * RabbitMQ host.  
     */  
    private String host = "localhost";  
  
    /**  
     * RabbitMQ port.  
     */  
    private int port = 5672;   .... //省略部分代码}  
```

 `RabbitProperties`提供对RabbitMQ的配置信息，其前缀为`spring.rabbitmq`，因此在上篇中配置的host、port等信息会配置到该类上，随`后@EnableConfigurationProperties`会将`RabbitProperties`注册为一个bean。


- `@Import`为导入配置，`RabbitAnnotationDrivenConfiguration`具体实现如下：



```java
@Configuration  
@ConditionalOnClass(EnableRabbit.class)  
class RabbitAnnotationDrivenConfiguration {  
  
    @Autowired(required = false)  
    private PlatformTransactionManager transactionManager;  
  
    @Bean  
    @ConditionalOnMissingBean(name = "rabbitListenerContainerFactory")  
    public SimpleRabbitListenerContainerFactory rabbitListenerContainerFactory(  
            ConnectionFactory connectionFactory) {  
        SimpleRabbitListenerContainerFactory factory = new SimpleRabbitListenerContainerFactory();  
        factory.setConnectionFactory(connectionFactory);  
        if (this.transactionManager != null) {  
            factory.setTransactionManager(this.transactionManager);  
        }  
        return factory;  
    }  
  
    @EnableRabbit  
    @ConditionalOnMissingBean(name = RabbitListenerConfigUtils.RABBIT_LISTENER_ANNOTATION_PROCESSOR_BEAN_NAME)  
    protected static class EnableRabbitConfiguration {  
  
    }  
  
}  
```


这里又涉及到一个重要的注解：`@ConditionalOnMissingBean`，其功能为如果存在指定name的bean，则该注解标注的bean不创建，`@ConditionalOnMissingBean(name = "rabbitListenerContainerFactory")  `

 表示的意思为：如果存在名称为`rabbitListenerContainerFactory`的bean，则该部分代码直接忽略，这是Spring Boot人性化体现之一，**开发者申明的bean会放在第一位**，实在是太6666


再回到`RabbitAutoConfiguration`类的具体实现
首先来看：

```java
@Configuration  
    @ConditionalOnMissingBean(ConnectionFactory.class)  
    protected static class RabbitConnectionFactoryCreator {  
  
        @Bean  
        public ConnectionFactory rabbitConnectionFactory(RabbitProperties config) {  
            CachingConnectionFactory factory = new CachingConnectionFactory();  
            String addresses = config.getAddresses();  
            factory.setAddresses(addresses);  
            if (config.getHost() != null) {  
                factory.setHost(config.getHost());  
                factory.setPort(config.getPort());  
            }  
            if (config.getUsername() != null) {  
                factory.setUsername(config.getUsername());  
            }  
            if (config.getPassword() != null) {  
                factory.setPassword(config.getPassword());  
            }  
            if (config.getVirtualHost() != null) {  
                factory.setVirtualHost(config.getVirtualHost());  
            }  
            return factory;  
        }  
  
    }  
```

创建了默认的ConnectionFactory，需要注意的时，这里的ConnectionFactory无回调的设置;


```java
    @Bean  
    @ConditionalOnMissingBean(RabbitTemplate.class)  
    public RabbitTemplate rabbitTemplate() {  
        return new RabbitTemplate(this.connectionFactory);  
    }  
```

创建了默认的`RabbitTemplate`；下面创建的`RabbitMessagingTemplate`实现对RabbitTemplate的包装。

在`RabbitAutoConfiguration`类中还剩AmqpAdmin的创建没有解;



----------------------

OVER


[原文](http://blog.csdn.net/liaokailin/article/details/49559951)


