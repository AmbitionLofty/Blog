# SpringBoot入门及四种事件监听源码解析

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


## 监听源码分析

首先来看下执行的main方法:

```java

package com.lkl.springboot;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

import com.lkl.springboot.listener.MyApplicationStartedEventListener;

@SpringBootApplication
public class Application {
    public static void main(String[] args) {
        SpringApplication app = new SpringApplication(Application.class);
        //app.setAdditionalProfiles("dev");
        app.addListeners(new MyApplicationStartedEventListener());
        app.run(args);
    }
}
```


`SpringApplication app = new SpringApplication(Application.class)` 创建一个**SpringApplication**实例；创建实例执行对象构造方法；其构造方法如下：

```java
public SpringApplication(Object... sources) {
        initialize(sources);
    }
```

调用`initialize()`,该方法执行若干初始化操作，在后续再继续深入该方法。

`app.addListeners(new MyApplicationStartedEventListener());` 调用**SpringApplication**添加监听的方法执行操作：

```java
public void addListeners(ApplicationListener<?>... listeners) {
    this.listeners.addAll(Arrays.asList(listeners));
}
```


`this.listeners`为`List<ApplicationListener<?>>`类型，是SpringApplication中所有监听器的持有容器，**在initialize()方法中也会往该监听集合中添加初始化的监听器**

执行完添加监听器方法后执行`app.run(args)`方法


```java
public ConfigurableApplicationContext run(String... args) {
        StopWatch stopWatch = new StopWatch();
        stopWatch.start();
        ConfigurableApplicationContext context = null;

        System.setProperty(
                SYSTEM_PROPERTY_JAVA_AWT_HEADLESS,
                System.getProperty(SYSTEM_PROPERTY_JAVA_AWT_HEADLESS,
                        Boolean.toString(this.headless)));

		// getRunListeners  获取执行时监听的集合
        Collection<SpringApplicationRunListener> runListeners = getRunListeners(args);
        for (SpringApplicationRunListener runListener : runListeners) {
            runListener.started();
        }

        try {
            // Create and configure the environment
            ConfigurableEnvironment environment = getOrCreateEnvironment();
            configureEnvironment(environment, args);
            for (SpringApplicationRunListener runListener : runListeners) {
                runListener.environmentPrepared(environment);
            }
            if (this.showBanner) {
                printBanner(environment);
            }

            // Create, load, refresh and run the ApplicationContext  创建、加载、刷新和运行应用
            context = createApplicationContext();
            if (this.registerShutdownHook) {
                try {
                    context.registerShutdownHook();
                }
                catch (AccessControlException ex) {
                    // Not allowed in some environments.
                }
            }
            context.setEnvironment(environment);
            postProcessApplicationContext(context);
            applyInitializers(context);
            for (SpringApplicationRunListener runListener : runListeners) {
                runListener.contextPrepared(context);
            }
            if (this.logStartupInfo) {
                logStartupInfo(context.getParent() == null);
            }

            // Load the sources  加载资源
            Set<Object> sources = getSources();
            Assert.notEmpty(sources, "Sources must not be empty");
            load(context, sources.toArray(new Object[sources.size()]));
            for (SpringApplicationRunListener runListener : runListeners) {
                runListener.contextLoaded(context);
            }

            // Refresh the context  刷新context
            refresh(context);
            afterRefresh(context, args);
            for (SpringApplicationRunListener runListener : runListeners) {
                runListener.finished(context, null);
            }

            stopWatch.stop();
            if (this.logStartupInfo) {
                new StartupInfoLogger(this.mainApplicationClass).logStarted(
                        getApplicationLog(), stopWatch);
            }
            return context;
        }
        catch (Throwable ex) {
            try {
                for (SpringApplicationRunListener runListener : runListeners) {
                    finishWithException(runListener, context, ex);
                }
                this.log.error("Application startup failed", ex);
            }
            finally {
                if (context != null) {
                    context.close();
                }
            }
            ReflectionUtils.rethrowRuntimeException(ex);
            return context;
        }
    }
```


在`run()`方法中完成了spring boot的启动，方法代码比较长，本篇重点放在事件监听上；

```
Collection<SpringApplicationRunListener> runListeners = getRunListeners(args)
```

通过`getRunListeners(args)`获取执行时监听的集合，其代码如下：

```java
private Collection<SpringApplicationRunListener> getRunListeners(String[] args) {
        List<SpringApplicationRunListener> listeners = new ArrayList<SpringApplicationRunListener>();
        listeners.addAll(getSpringFactoriesInstances(SpringApplicationRunListener.class,
                new Class<?>[] { SpringApplication.class, String[].class }, this, args));
        return listeners;
    }
```

**重点关注**

```
listeners.addAll(getSpringFactoriesInstances(SpringApplicationRunListener.class,
                new Class<?>[] { SpringApplication.class, String[].class }, this, args));
```

**该方法获取执行类型子类实例集合:**


```
private <T> Collection<? extends T> getSpringFactoriesInstances(Class<T> type,
            Class<?>[] parameterTypes, Object... args) {
        ClassLoader classLoader = Thread.currentThread().getContextClassLoader();

        // Use names and ensure unique to protect against duplicates 使用名称并确保独特地防止重复
        Set<String> names = new LinkedHashSet<String>(
                SpringFactoriesLoader.loadFactoryNames(type, classLoader));
        List<T> instances = new ArrayList<T>(names.size());

        // Create instances from the names 从名称创建实例
        for (String name : names) {
            try {
                Class<?> instanceClass = ClassUtils.forName(name, classLoader);
                Assert.isAssignable(type, instanceClass);
                Constructor<?> constructor = instanceClass.getConstructor(parameterTypes);
                T instance = (T) constructor.newInstance(args);
                instances.add(instance);
            }
            catch (Throwable ex) {
                throw new IllegalArgumentException("Cannot instantiate " + type + " : "
                        + name, ex);
            }
        }

        AnnotationAwareOrderComparator.sort(instances);
        return instances;
    }
```

看 `SpringFactoriesLoader.loadFactoryNames(type, classLoader)`

```java
public static List<String> loadFactoryNames(Class<?> factoryClass, ClassLoader classLoader) {
        String factoryClassName = factoryClass.getName();
        try {
            Enumeration<URL> urls = (classLoader != null ? classLoader.getResources(FACTORIES_RESOURCE_LOCATION) :
                    ClassLoader.getSystemResources(FACTORIES_RESOURCE_LOCATION));
            List<String> result = new ArrayList<String>();
            while (urls.hasMoreElements()) {
                URL url = urls.nextElement();
                Properties properties = PropertiesLoaderUtils.loadProperties(new UrlResource(url));
                String factoryClassNames = properties.getProperty(factoryClassName);
                result.addAll(Arrays.asList(StringUtils.commaDelimitedListToStringArray(factoryClassNames)));
            }
            return result;
        }
        catch (IOException ex) {
            throw new IllegalArgumentException("Unable to load [" + factoryClass.getName() +
                    "] factories from location [" + FACTORIES_RESOURCE_LOCATION + "]", ex);
        }
    }
```

**其中`classLoader.getResources....`通过类加载器获取`resources`  中 `FACTORIES_RESOURCE_LOCATION = "META-INF/spring.factories";` ；**

```
 Enumeration<URL> urls = (classLoader != null ? classLoader.getResources(FACTORIES_RESOURCE_LOCATION) :ClassLoader.getSystemResources(FACTORIES_RESOURCE_LOCATION));
```



代码回去扫描项目**工程中/META-INF下的spring.factories文件**,获取`org.springframework.boot.SpringApplicationRunListener`对应数据 。在`spring-boot-1.2.4.RELEASE`中可以找到如下信息

```
# Run Listeners
org.springframework.boot.SpringApplicationRunListener=\
org.springframework.boot.context.event.EventPublishingRunListener
```

即通过`Collection<SpringApplicationRunListener> runListeners = getRunListeners(args);`最终拿到的是`EventPublishingRunListener`。

在获取`EventPublishingRunListener`实例时，执行对应构造方法：

```java
public EventPublishingRunListener(SpringApplication application, String[] args) {
        this.application = application;
        this.args = args;
        this.multicaster = new SimpleApplicationEventMulticaster();
        for (ApplicationListener<?> listener : application.getListeners()) {
            this.multicaster.addApplicationListener(listener);
        }
    }
```
将`SpringApplication`中的监听器传递给`SimpleApplicationEventMulticaster`实例`multicaster`执行：

```
	for (SpringApplicationRunListener runListener : runListeners) {
            runListener.started();
        }
```

调用`EventPublishingRunListener`中的`started()`方法：


```java
@Override
public void started() {
    publishEvent(new ApplicationStartedEvent(this.application, this.args));
}
```


在该方法中首先创建一个`ApplicationStartedEvent`事件，将`this.application`传递过去，因此在执行`ApplicationStartedEvent`监听时可以获取`SpringApplication`实例。

执行`publishEvent()` 方法

```
   private void publishEvent(SpringApplicationEvent event) {
        this.multicaster.multicastEvent(event);
    }
```


调用`SimpleApplicationEventMulticaster.multicastEvent(event)`

```
    @Override
    public void multicastEvent(final ApplicationEvent event) {
        for (final ApplicationListener<?> listener : getApplicationListeners(event)) {
            Executor executor = getTaskExecutor();
            if (executor != null) {
                executor.execute(new Runnable() {
                    @Override
                    public void run() {
                        invokeListener(listener, event);
                    }
                });
            }
            else {
                invokeListener(listener, event);
            }
        }
    }
```


在该代码中需要注意的是for循环中获取监听器集合方`getApplicationListeners(event)`,由于传递的事件为`ApplicationStartedEvent`,因此该方法需要获取到`ApplicationStartedEvent`对应的监听器


```java

protected Collection<ApplicationListener<?>> getApplicationListeners(ApplicationEvent event) {
        Object source = event.getSource();
        Class<?> sourceType = (source != null ? source.getClass() : null);
        ListenerCacheKey cacheKey = new ListenerCacheKey(event.getClass(), sourceType);

        // Quick check for existing entry on ConcurrentHashMap...
        ListenerRetriever retriever = this.retrieverCache.get(cacheKey);
        if (retriever != null) {
            return retriever.getApplicationListeners();
        }

        if (this.beanClassLoader == null ||
                (ClassUtils.isCacheSafe(event.getClass(), this.beanClassLoader) &&
                        (sourceType == null || ClassUtils.isCacheSafe(sourceType, this.beanClassLoader)))) {
            // Fully synchronized building and caching of a ListenerRetriever
            synchronized (this.retrievalMutex) {
                retriever = this.retrieverCache.get(cacheKey);
                if (retriever != null) {
                    return retriever.getApplicationListeners();
                }
                retriever = new ListenerRetriever(true);
                Collection<ApplicationListener<?>> listeners =
                        retrieveApplicationListeners(event, sourceType, retriever);
                this.retrieverCache.put(cacheKey, retriever);
                return listeners;
            }
        }
        else {
            // No ListenerRetriever caching -> no synchronization necessary
            return retrieveApplicationListeners(event, sourceType, null);
        }
    }
```


看`retrieveApplicationListeners(event, sourceType, null)`方法；该方法代码比较长，截取一部分出来

```java
private Collection<ApplicationListener<?>> retrieveApplicationListeners(
            ApplicationEvent event, Class<?> sourceType, ListenerRetriever retriever) {

        LinkedList<ApplicationListener<?>> allListeners = new LinkedList<ApplicationListener<?>>();
        Set<ApplicationListener<?>> listeners;
        Set<String> listenerBeans;
        synchronized (this.retrievalMutex) {
            listeners = new LinkedHashSet<ApplicationListener<?>>(this.defaultRetriever.applicationListeners);
            listenerBeans = new LinkedHashSet<String>(this.defaultRetriever.applicationListenerBeans);
        }
        for (ApplicationListener<?> listener : listeners) {
            if (supportsEvent(listener, event.getClass(), sourceType)) {
                if (retriever != null) {
                    retriever.applicationListeners.add(listener);
                }
                allListeners.add(listener);
            }
        }


	...      
        OrderComparator.sort(allListeners);
        return allListeners;
    }
```

调用`supportsEvent`方法判断对应的监听器是否支持指定的事件

```java
protected boolean supportsEvent(ApplicationListener<?> listener,
            Class<? extends ApplicationEvent> eventType, Class<?> sourceType) {

        SmartApplicationListener smartListener = (listener instanceof SmartApplicationListener ?
                (SmartApplicationListener) listener : new GenericApplicationListenerAdapter(listener));
        return (smartListener.supportsEventType(eventType) && smartListener.supportsSourceType(sourceType));
    }
```

执行 `GenericApplicationListenerAdapter.supportsEventType(eventType)`

```java
public boolean supportsEventType(Class<? extends ApplicationEvent> eventType) {
        Class<?> declaredEventType = resolveDeclaredEventType(this.delegate.getClass());
        if (declaredEventType == null || declaredEventType.equals(ApplicationEvent.class)) {
            Class<?> targetClass = AopUtils.getTargetClass(this.delegate);
            if (targetClass != this.delegate.getClass()) {
                declaredEventType = resolveDeclaredEventType(targetClass);
            }
        }
        return (declaredEventType == null || declaredEventType.isAssignableFrom(eventType));
    }
```


调用`resolveDeclaredEventType()`方法获取指定类继承的父类或实现接口时传递的泛型对应的类型，这句话有点绕口，可以自行看一下

 
```java
  static Class<?> resolveDeclaredEventType(Class<?> listenerType) {
        return GenericTypeResolver.resolveTypeArgument(listenerType, ApplicationListener.class);
    }
```

`GenericTypeResolver`泛型解析工具类功能强大，我们在实际开发中同样可以利用。

至此`getApplicationListeners(event)`调用完成，大体思路为：**遍历所有的监听器，如果该监听器监听的事件为传递的事件或传递事件的父类则表示该监听器支持指定事件。**

**获取完指定事件对应监听器后，通过`Executor`执行一个子线程去完成监听器`listener.onApplicationEvent(event)`方法。**



*** 

OVER

*** 

[查看原文](http://blog.csdn.net/liaokailin/article/details/48186331)



