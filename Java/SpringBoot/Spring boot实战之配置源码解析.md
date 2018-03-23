# Spring boot实战之配置源码解析

***

[toc]
***


@(SpringBoot)



从源码的角度来看看配置文件


## 环境(Environment)

在bean中注入`Enviroment`实例即可调用配置资源信息，如以下代码：


```java
package com.lkl.springboot.config;  
  
import org.springframework.beans.factory.annotation.Autowired;  
import org.springframework.core.env.Environment;  
import org.springframework.stereotype.Component;  
  
/** 
 * 注入enviroment 
 *  
 * @author liaokailin 
 * @version $Id: DIEnviroment.java, v 0.1 2015年10月2日 下午9:17:19 liaokailin Exp $ 
 */  
@Component  
public class DIEnviroment {  
    @Autowired  
    Environment environment;  
  
    public String getProValueFromEnviroment(String key) {  
        return environment.getProperty(key);  
    }  
}  
```

## Spring boot是如何来构建环境?

`SpringApplication.run(String... args)`中的代码:

```java
	// Create and configure the environment  
    ConfigurableEnvironment environment = getOrCreateEnvironment();  
```

```java
private ConfigurableEnvironment getOrCreateEnvironment() {
    if (this.environment != null) {
        return this.environment;
    }
    if (this.webEnvironment) {
        return new StandardServletEnvironment();
    }
    return new StandardEnvironment();
}
```


初始 `environment` 为空，`this.webEnvironment` 判断构建的是否为web环境，通过`deduceWebEnvironment`方法推演出为`true`


```java

private boolean deduceWebEnvironment() {  
        for (String className : WEB_ENVIRONMENT_CLASSES) {  
            if (!ClassUtils.isPresent(className, null)) {  
                return false;  
            }  
        }  
        return true;  
    }  

```


由于可以得到得出构建的`enviroment`为`StandardServletEnvironment`;

其类继承关系如下:
 
创建对象调用其父类已经自身构造方法，`StandardServletEnvironment、StandardEnvironment`无构造方法，调用`AbstractEnvironment`构造方法


```java
public AbstractEnvironment() {  
        customizePropertySources(this.propertySources);  
        if (this.logger.isDebugEnabled()) {  
            this.logger.debug(format(  
                    "Initialized %s with PropertySources %s", getClass().getSimpleName(), this.propertySources));  
        }  
    }  
```

 首先看`this.propertySources`定义:

```
private final MutablePropertySources propertySources = new MutablePropertySources(this.logger);  
```


从字面的意义可以看出`MutablePropertySources`为多`PropertySource`的集合,其定义如下：


```java
public class MutablePropertySources implements PropertySources {  
  
    private final Log logger;  
  
    private final List<PropertySource<?>> propertySourceList = new CopyOnWriteArrayList<PropertySource<?>>();  ...}  

```
其中`PropertySource`保存配置资源信息


```java
public abstract class PropertySource<T> {  
  
    protected final Log logger = LogFactory.getLog(getClass());  
  
    protected final String name;  
  
    protected final T source; ...}  
```

**资源信息元数据`PropertySource`** 包含name和泛型，**一份资源信息存在唯一的name以及对应泛型数据**，在这里设计为泛型表明可拓展自定义类型。

如需自定义或增加资源信息，即只需构建`PropertySource`或其子类，然后添加到`MutablePropertySources`中属性`List<PropertySource<?>>`集合中，`MutablePropertySources`又作为`AbstractEnvironment`中的属性，因此将`AbstractEnvironment`保存在**springbean容器**中即可访问到所有的`PropertySource`。




来继续查看源码:


继续来看`AbstractEnvironment`对应构造方法中的`customizePropertySources`


```java
protected void customizePropertySources(MutablePropertySources propertySources) {  
    }  
```


为`protected`且无实现的方法，将具体的实现放在子类来实现，调用`StandardServletEnvironment`中的具体实现：\


```java
protected void customizePropertySources(MutablePropertySources propertySources) {  
        propertySources.addLast(new StubPropertySource(SERVLET_CONFIG_PROPERTY_SOURCE_NAME));  
        propertySources.addLast(new StubPropertySource(SERVLET_CONTEXT_PROPERTY_SOURCE_NAME));  
        if (JndiLocatorDelegate.isDefaultJndiEnvironmentAvailable()) {  
            propertySources.addLast(new JndiPropertySource(JNDI_PROPERTY_SOURCE_NAME));  
        }  
        super.customizePropertySources(propertySources);  
    }  
```

这里调用的`propertiesSources`即为`AbstractEnvironment`中的属性，该方法将往集合中添加指定名称的`PropertySource`；来看下addLast方法：

```java
public void addLast(PropertySource<?> propertySource) {  
        if (logger.isDebugEnabled()) {  
            logger.debug(String.format("Adding [%s] PropertySource with lowest search precedence",  
                    propertySource.getName()));  
        }  
        removeIfPresent(propertySource);  
        this.propertySourceList.add(propertySource);  
    }  
```

其中`removeIfPresent(propertySource)`从字面意义中也可以看出为如果存在该`PropertySource`的话则从集合中删除数据：

```java
protected void removeIfPresent(PropertySource<?> propertySource) {  
        this.propertySourceList.remove(propertySource);  
    } 
```


由于`PropertySource`中属性T泛型是不固定并对应内容也不固定，因此判断`PropertySource`在集合中的唯一性只能去看`name`，因此在`PropertySource`中重写`equals,hashCode`


```java
@Override  
    public boolean equals(Object obj) {  
        return (this == obj || (obj instanceof PropertySource &&  
                ObjectUtils.nullSafeEquals(this.name, ((PropertySource<?>) obj).name)));  
    }  
  
    /**  
     * Return a hash code derived from the {@code name} property  
     * of this {@code PropertySource} object.  
     */  
    @Override  
    public int hashCode() {  
        return ObjectUtils.nullSafeHashCode(this.name);  
    }  
```


从上可看出`name`标识`PropertySource`的唯一性。

至此`StandardEnvironment`的初始化完成.


## 创建Enviroment Bean


在bean中注入Enviroment实际为 Enviroment 接口的实现类，从类图中可以看出其子类颇多，具体在容器中是哪个子类就需要从代码获取答案。

在`SpringApplication.run(String... args)`中存在`refresh(context)`调用:


```java

protected void refresh(ApplicationContext applicationContext) {  
        Assert.isInstanceOf(AbstractApplicationContext.class, applicationContext);  
        ((AbstractApplicationContext) applicationContext).refresh();  
    }  
```

实际调用`AbstractApplicationContext`中的`refresh`方法，在`refresh`方法调用`prepareBeanFactory(beanFactory)`，其实现如下：


```java
protected void prepareBeanFactory(ConfigurableListableBeanFactory beanFactory) {  
         ...  
  
        // Register default environment beans.  
        if (!beanFactory.containsLocalBean(ENVIRONMENT_BEAN_NAME)) {  
            beanFactory.registerSingleton(ENVIRONMENT_BEAN_NAME, getEnvironment());  
        }  
        if (!beanFactory.containsLocalBean(SYSTEM_PROPERTIES_BEAN_NAME)) {  
            beanFactory.registerSingleton(SYSTEM_PROPERTIES_BEAN_NAME, getEnvironment().getSystemProperties());  
        }  
        if (!beanFactory.containsLocalBean(SYSTEM_ENVIRONMENT_BEAN_NAME)) {  
            beanFactory.registerSingleton(SYSTEM_ENVIRONMENT_BEAN_NAME, getEnvironment().getSystemEnvironment());  
        }  
    }  
```

 其中调用`beanFactory.registerSingleton(ENVIRONMENT_BEAN_NAME, getEnvironment())`注册名称为`environment`的bean；

```java
public ConfigurableEnvironment getEnvironment() {  
        if (this.environment == null) {  
            this.environment = createEnvironment();  
        }  
        return this.environment;  
    }
```

**其中`enviroment`变量为前面创建`**StandardServletEnvironment`；前后得到验证。     



## 实战：动态加载资源


**在实际项目中资源信息如果能够动态获取在修改线上产品配置时及其方便**，下面来展示一个加载动态获取资源的案例，而不是加载写死的`properties`文件信息
首先构造`PropertySource`，然后将其添加到`Enviroment`中:



### 构造PropertySource


```java

package com.lkl.springboot.config;  
  
import java.text.SimpleDateFormat;  
import java.util.Date;  
import java.util.HashMap;  
import java.util.Map;  
import java.util.Random;  
import java.util.concurrent.ConcurrentHashMap;  
import java.util.concurrent.Executors;  
import java.util.concurrent.ScheduledExecutorService;  
import java.util.concurrent.TimeUnit;  
  
import org.slf4j.Logger;  
import org.slf4j.LoggerFactory;  
import org.springframework.core.env.MapPropertySource;  
  
public class DynamicPropertySource extends MapPropertySource {  
  
    private static Logger log = LoggerFactory.getLogger(DynamicPropertySource.class);  
  
    private static ScheduledExecutorService scheduled = Executors.newScheduledThreadPool(1);  
    
    static {  
        scheduled.scheduleAtFixedRate(new Runnable() {  
            @Override  
            public void run() {  
                map = dynamicLoadMapInfo();  
            }  
  
        }, 1, 10, TimeUnit.SECONDS);  
    }  
  
    public DynamicPropertySource(String name) {  
        super(name, map);  
    }  
  
    private static Map<String, Object> map = new ConcurrentHashMap<String, Object>(64);  
  
    @Override  
    public Object getProperty(String name) {  
        return map.get(name);  
    }  
  
    //动态获取资源信息  
    private static Map<String, Object> dynamicLoadMapInfo() {  
        //通过http或tcp等通信协议获取配置信息  
        return mockMapInfo();  
    }  
  
    private static final SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd hh:mm:ss");  
  
    private static Map<String, Object> mockMapInfo() {  
        Map<String, Object> map = new HashMap<String, Object>();  
        int randomData = new Random().nextInt();  
        log.info("random data{};currentTime:{}", randomData, sdf.format(new Date()));  
        map.put("dynamic-info", randomData);  
        return map;  
    }  
}  
```

这里模拟动态获取配置信息；

### 添加到Enviroment

```java

package com.lkl.springboot.config;  
  
import javax.annotation.PostConstruct;  
  
import org.springframework.beans.factory.annotation.Autowired;  
import org.springframework.context.annotation.Configuration;  
import org.springframework.core.env.AbstractEnvironment;  
  
/**  
 * 加载动态配置信息  
 *   
 * @author shizhiyuan
 * @version $Id: DynamicConfig.java  
 */  
@Configuration  
public class DynamicConfig {  
    public static final String DYNAMIC_CONFIG_NAME = "dynamic_config";  
  
    @Autowired  
    AbstractEnvironment  environment;  
  
    @PostConstruct  
    public void init() {  
        environment.getPropertySources().addFirst(new DynamicPropertySource(DYNAMIC_CONFIG_NAME));  
    }  
  
}  
```