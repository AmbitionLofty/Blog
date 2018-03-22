# SpringBoot入门及四种事件监听

@(SpringBoot)
***


## @SpringBootApplication 注解

`SpringBootApplication`注解源码如下：


```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@Configuration
@EnableAutoConfiguration
@ComponentScan
public @interface SpringBootApplication {

    /**
     * Exclude specific auto-configuration classes such that they will never be applied.
     * @return the classes to exclude
     */
    Class<?>[] exclude() default {};

}
```

**`@Configuration`** :  表示 Application 作为 spring 配置文件存在 
**`@EnableAutoConfiguration`:  ** 启动 spring boot 内置的自动配置 
**`@ComponentScan`** :  扫描bean，路径为 Application 类所在package以及package下的子路径，这里为 com.lkl.springboot，在spring boot中bean都放置在该路径已经子路径下。


## Controller 

创建一个package：`com.lkl.springboot.controller` 保存 `controller`

构建 `HelloWorldController.java`


```java
 package com.lkl.springboot.controller;

import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestMethod;
import org.springframework.web.bind.annotation.ResponseBody;
import org.springframework.web.bind.annotation.RestController;

@RestController
@RequestMapping("/springboot")
public class HelloWorldController {

    @RequestMapping(value = "/{name}", method = RequestMethod.GET)
    public String sayWorld(@PathVariable("name") String name) {
        return "Hello " + name;
    }
}
```


## 事件监听
spring boot在启动过程中增加事件监听机制，为用户功能拓展提供极大的便利。

支持的事件类型四种:

1. ```ApplicationStartedEvent```

2. ```ApplicationEnvironmentPreparedEvent```

3. ```ApplicationPreparedEvent```

4. ```ApplicationFailedEvent```

### 实现监听步骤：

1. 监听类实现`ApplicationListener`接口 
2. 将监听类添加到`SpringApplication`实例

### ApplicationStartedEvent


`ApplicationStartedEvent`：spring boot启动开始时执行的事件

创建对应的监听类 `MyApplicationStartedEventListener.java`


```java
package com.lkl.springboot.listener;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.context.event.ApplicationStartedEvent;
import org.springframework.context.ApplicationListener;

/**
 * spring boot 启动监听类
 * 
 * @author 
 * @version $Id: MyApplicationStartedEventListener.java, v 0.1  $
 */
public class MyApplicationStartedEventListener implements ApplicationListener<ApplicationStartedEvent> {

    private Logger logger = LoggerFactory.getLogger(MyApplicationStartedEventListener.class);

    @Override
    public void onApplicationEvent(ApplicationStartedEvent event) {
        SpringApplication app = event.getSpringApplication();
        app.setShowBanner(false);// 不显示banner信息
        logger.info("==MyApplicationStartedEventListener==");
    }
}
```


在该事件中可以获取到`SpringApplication`对象，可做一些执行前的设置.



`Application.java`类


```java
package com.lkl.springboot;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

import com.lkl.springboot.listener.MyApplicationStartedEventListener;

@SpringBootApplication
public class Application {
    public static void main(String[] args) {
        SpringApplication app = new SpringApplication(Application.class); 
        app.addListeners(new MyApplicationStartedEventListener());
        app.run(args);
    }
}
```


### ApplicationEnvironmentPreparedEvent


`ApplicationEnvironmentPreparedEvent`：spring boot 对应`Enviroment`已经准备完毕，但此时上下文context还没有创建。

**`MyApplicationEnvironmentPreparedEventListener.java`**


```java
package com.lkl.springboot.listener;

import java.util.Iterator;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.boot.context.event.ApplicationEnvironmentPreparedEvent;
import org.springframework.context.ApplicationListener;
import org.springframework.core.env.ConfigurableEnvironment;
import org.springframework.core.env.MutablePropertySources;
import org.springframework.core.env.PropertySource;

/**
 * spring boot 配置环境事件监听
 * @author 
 * @version $Id: MyApplicationEnvironmentPreparedEventListener.java, 
 */
public class MyApplicationEnvironmentPreparedEventListener implements                                                      ApplicationListener<ApplicationEnvironmentPreparedEvent> {
    private Logger logger = LoggerFactory.getLogger(MyApplicationEnvironmentPreparedEventListener.class);

    @Override
    public void onApplicationEvent(ApplicationEnvironmentPreparedEvent event) {

        ConfigurableEnvironment envi = event.getEnvironment();
        MutablePropertySources mps = envi.getPropertySources();
        if (mps != null) {
            Iterator<PropertySource<?>> iter = mps.iterator();
            while (iter.hasNext()) {
                PropertySource<?> ps = iter.next();
                logger
                    .info("ps.getName:{};ps.getSource:{};ps.getClass:{}", ps.getName(), ps.getSource(), ps.getClass());
            }
        }
    }

}

```


在该监听中获取到`ConfigurableEnvironment`后可以对配置信息做操作，**例如：**修改默认的配置信息，增加额外的配置信息等等~~~


### ApplicationPreparedEvent

`ApplicationPreparedEvent`:spring boot上下文 **context** 创建完成，但此时spring中的bean是没有完全加载完成的。

**`MyApplicationPreparedEventListener.java`**


```java

package com.lkl.springboot.listener;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.boot.context.event.ApplicationPreparedEvent;
import org.springframework.context.ApplicationContext;
import org.springframework.context.ApplicationListener;
import org.springframework.context.ConfigurableApplicationContext;

/**
 * 上下文创建完成后执行的事件监听器
 * 
 * @author liaokailin
 * @version $Id: MyApplicationPreparedEventListener.java, v 0.1 2015年9月2日 下午11:29:38 liaokailin Exp $
 */
public class MyApplicationPreparedEventListener implements ApplicationListener<ApplicationPreparedEvent> {
    private Logger logger = LoggerFactory.getLogger(MyApplicationPreparedEventListener.class);

    @Override
    public void onApplicationEvent(ApplicationPreparedEvent event) {
        ConfigurableApplicationContext cac = event.getApplicationContext();
        passContextInfo(cac);
    }

    /**
     * 传递上下文
     * @param cac
     */
    private void passContextInfo(ApplicationContext cac) {
        //dosomething()
    }

}
```


**在获取完上下文后，可以将上下文传递出去做一些额外的操作。**

**在该监听器中是无法获取自定义bean并进行操作的。**


### ApplicationFailedEvent


`ApplicationFailedEvent`:spring boot启动异常时执行事件 


**`MyApplicationFailedEventListener.java`**


```java
package com.lkl.springboot.listener;

import org.springframework.boot.context.event.ApplicationFailedEvent;
import org.springframework.context.ApplicationListener;

public class MyApplicationFailedEventListener implements ApplicationListener<ApplicationFailedEvent> {

    @Override
    public void onApplicationEvent(ApplicationFailedEvent event) {
        Throwable throwable = event.getException();
        handleThrowable(throwable);
    }

    /*处理异常*/
    private void handleThrowable(Throwable throwable) {
    }

}
```


在异常发生时，**最好是添加虚拟机对应的钩子进行资源的回收与释放，能友善的处理异常信息**。

在spring boot中已经为大家考虑了这一点，默认情况开启了对应的功能：


```java
public void registerShutdownHook() {
        if (this.shutdownHook == null) {
            // No shutdown hook registered yet.
            this.shutdownHook = new Thread() {
                @Override
                public void run() {
                    doClose();
                }
            };
            Runtime.getRuntime().addShutdownHook(this.shutdownHook);
        }
    }

```

**在`doClose()`方法中进行资源的回收与释放。**



**四种监听事件到这里就结束了，针对实际业务可添加自定义的监听器.**


[查看原文](http://blog.csdn.net/liaokailin/article/details/48186331)