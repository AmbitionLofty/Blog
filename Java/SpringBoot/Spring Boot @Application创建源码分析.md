# Spring Boot @Application创建源码分析

***
[toc]
***

@(SpringBoot)

*** 

在spring boot的启动时，利用的是编写的`Application`类，使用了注解`@SpringBootApplication`，本篇将阐述该Bean的加载过程:

```java
@SpringBootApplication  
public class Application {  
    public static void main(String[] args) {  
        SpringApplication app = new SpringApplication(Application.class);   
        app.addListeners(new MyApplicationStartedEventListener());  
        app.run(args);  
    }  
} 
```

## Application 


上篇中讲述了上下文的创建，在run方法中接下来会执行

```
load(context, sources.toArray(new Object[sources.size()]));
```

  

这个的`sources`表示的为`Application`类，在创建`SpringApplication`时手动传递

```
SpringApplication app = new SpringApplication(Application.class);  
```


`load`方法如下：


```java
protected void load(ApplicationContext context, Object[] sources) {  
        if (this.log.isDebugEnabled()) {  
            this.log.debug("Loading source "  
                    + StringUtils.arrayToCommaDelimitedString(sources));  
        }  
        BeanDefinitionLoader loader = createBeanDefinitionLoader(  
                getBeanDefinitionRegistry(context), sources);  
        if (this.beanNameGenerator != null) {  
            loader.setBeanNameGenerator(this.beanNameGenerator);  
        }  
        if (this.resourceLoader != null) {  
            loader.setResourceLoader(this.resourceLoader);  
        }  
        if (this.environment != null) {  
            loader.setEnvironment(this.environment);  
        }  
        loader.load();  
    }  

```

调用`loader.load()`:


```java
private int load(Object source) {  
        Assert.notNull(source, "Source must not be null");  
        if (source instanceof Class<?>) {  
            return load((Class<?>) source);  
        }  
        if (source instanceof Resource) {  
            return load((Resource) source);  
        }  
        if (source instanceof Package) {  
            return load((Package) source);  
        }  
        if (source instanceof CharSequence) {  
            return load((CharSequence) source);  
        }  
        throw new IllegalArgumentException("Invalid source type " + source.getClass());  
    }  
```


执行`load((Class<?>) source)` :

```java

private int load(Class<?> source) {  
        if (isGroovyPresent()) {  
            // Any GroovyLoaders added in beans{} DSL can contribute beans here  
            if (GroovyBeanDefinitionSource.class.isAssignableFrom(source)) {  
                GroovyBeanDefinitionSource loader = BeanUtils.instantiateClass(source,  
                        GroovyBeanDefinitionSource.class);  
                load(loader);  
            }  
        }  
        if (isComponent(source)) {  
            this.annotatedReader.register(source);  
            return 1;  
        }  
        return 0;  
    }
```

**`isComponent`判断`Application`是否存在注解`Compent`:**


```java

private boolean isComponent(Class<?> type) {  
        // This has to be a bit of a guess. The only way to be sure that this type is  
        // eligible is to make a bean definition out of it and try to instantiate it.  
        if (AnnotationUtils.findAnnotation(type, Component.class) != null) {  
            return true;  
        }  
        // Nested anonymous classes are not eligible for registration, nor are groovy  
        // closures  
        if (type.getName().matches(".*\\$_.*closure.*") || type.isAnonymousClass()  
                || type.getConstructors() == null || type.getConstructors().length == 0) {  
            return false;  
        }  
        return true;  
    }  
```

`AnnotationUtils.findAnnotation(type, Component.class)` 工具类获取执行类对应的注解信息，该工具类在自己编码代码时可用得到

**由于`Application`使用注解`@SpringBootApplication`，其定义如下:**


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


发现不存在`Compoment注解`，是不是表明`Application`不是一个`Component`呢？其实不然，来看下`@Configuration`注解:


```java

@Target(ElementType.TYPE)  
@Retention(RetentionPolicy.RUNTIME)  
@Documented  
@Component  
public @interface Configuration {  
  
    /**  
     * Explicitly specify the name of the Spring bean definition associated  
     * with this Configuration class.  If left unspecified (the common case),  
     * a bean name will be automatically generated.  
     * <p>The custom name applies only if the Configuration class is picked up via  
     * component scanning or supplied directly to a {@link AnnotationConfigApplicationContext}.  
     * If the Configuration class is registered as a traditional XML bean definition,  
     * the name/id of the bean element will take precedence.  
     * @return the specified bean name, if any  
     * @see org.springframework.beans.factory.support.DefaultBeanNameGenerator  
     */  
    String value() default "";  
  
}  
```

发现`Configuration注解`上存在`Component`注解，表明`Application`为`Component`

接下来执行 **load中** 的`this.annotatedReader.register(source)`:

```java

public void register(Class<?>... annotatedClasses) {  
        for (Class<?> annotatedClass : annotatedClasses) {  
            registerBean(annotatedClass);  
        }  
    }  
```


调用`registerBean`注册`Application`对应的bean信息:


```java
public void registerBean(Class<?> annotatedClass, String name,  
            @SuppressWarnings("unchecked") Class<? extends Annotation>... qualifiers) {  
  
        AnnotatedGenericBeanDefinition abd = new AnnotatedGenericBeanDefinition(annotatedClass);  
        if (this.conditionEvaluator.shouldSkip(abd.getMetadata())) {  
            return;  
        }  
  
        ScopeMetadata scopeMetadata = this.scopeMetadataResolver.resolveScopeMetadata(abd);  
        abd.setScope(scopeMetadata.getScopeName());  
        String beanName = (name != null ? name : this.beanNameGenerator.generateBeanName(abd, this.registry));  
        AnnotationConfigUtils.processCommonDefinitionAnnotations(abd);  
        if (qualifiers != null) {  
            for (Class<? extends Annotation> qualifier : qualifiers) {  
                if (Primary.class.equals(qualifier)) {  
                    abd.setPrimary(true);  
                }  
                else if (Lazy.class.equals(qualifier)) {  
                    abd.setLazyInit(true);  
                }  
                else {  
                    abd.addQualifier(new AutowireCandidateQualifier(qualifier));  
                }  
            }  
        }  
  
        BeanDefinitionHolder definitionHolder = new BeanDefinitionHolder(abd, beanName);  
        definitionHolder = AnnotationConfigUtils.applyScopedProxyMode(scopeMetadata, definitionHolder, this.registry);  
        BeanDefinitionReaderUtils.registerBeanDefinition(definitionHolder, this.registry);  
    }  

```


首先来看:

```java
if (this.conditionEvaluator.shouldSkip(abd.getMetadata())) {  
            return;  
        }  
```

判断是否需要跳过，其代码如下：

```java

public boolean shouldSkip(AnnotatedTypeMetadata metadata, ConfigurationPhase phase) {  
        if (metadata == null || !metadata.isAnnotated(Conditional.class.getName())) {  
            return false;  
        }  
  
        if (phase == null) {  
            if (metadata instanceof AnnotationMetadata &&  
                    ConfigurationClassUtils.isConfigurationCandidate((AnnotationMetadata) metadata)) {  
                return shouldSkip(metadata, ConfigurationPhase.PARSE_CONFIGURATION);  
            }  
            return shouldSkip(metadata, ConfigurationPhase.REGISTER_BEAN);  
        }  
  
        List<Condition> conditions = new ArrayList<Condition>();  
        for (String[] conditionClasses : getConditionClasses(metadata)) {  
            for (String conditionClass : conditionClasses) {  
                Condition condition = getCondition(conditionClass, this.context.getClassLoader());  
                conditions.add(condition);  
            }  
        }  
  
        AnnotationAwareOrderComparator.sort(conditions);  
  
        for (Condition condition : conditions) {  
            ConfigurationPhase requiredPhase = null;  
            if (condition instanceof ConfigurationCondition) {  
                requiredPhase = ((ConfigurationCondition) condition).getConfigurationPhase();  
            }  
            if (requiredPhase == null || requiredPhase == phase) {  
                if (!condition.matches(this.context, metadata)) {  
                    return true;  
                }  
            }  
        }  
  
        return false;  
    }  
```

**该代码判断`Application`上是否存在`Conditional`注解，如果不满足`Conditional`对应条件则该bean不被创建；**


## Conditional注解


代码分析到这里可以先看看`Conditional`注解的使用，其定义为：

```java
@Retention(RetentionPolicy.RUNTIME)  
@Target({ElementType.TYPE, ElementType.METHOD})  
public @interface Conditional {  
    /**  
     * All {@link Condition}s that must {@linkplain Condition#matches match}  
     * in order for the component to be registered.  
     */  
    Class<? extends Condition>[] value();  
  
}  
```

从源码可以看出，首先判断`Application`上是否存在`Conditional`，如果存在，则获取`Conditional`注解中的value数组值，对应的Class必须实现`Condition`接口：


```java

public interface Condition {   
    boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata);
}  
```

如果`matches`返回`true` 表明该bean需要被创建，否则表明该bean不需要被创建:


明白了该注解的用法后，来一个实际案例:

创建`ConditionBean`，使用注解`@Conditional(MyCondition.class)`调用MyCondition类:

```java
  
import org.springframework.context.annotation.Conditional;  
import org.springframework.stereotype.Component;  
  
@Component("MyCondition")  
@Conditional(MyCondition.class)  
public class ConditionBean {  
}  
```


```java
/**  
 * 自定义condition  修改返回值，查看bean是否创建  
 *   
 * @author liaokailin  
 */  
public class MyCondition implements Condition {  
  
    /**  
     * 返回true 生成bean  
     * 返回false 不生成bean   
     */  
    @Override  
    public boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata) {  
        Map<String, Object> map = metadata.getAnnotationAttributes(Component.class.getName());  
        return "MyCondition".equals(map.get("value").toString());  
    }  
  
}  
```

**`MyCondition`实现接口`Condition`，在`matches`方法中获取bean上注解`Component`信息，如果bean名称等于MyCondition返回true，否则返回false，bean不会被创建。**

回到前面Application的分析，Application上不存在Conditional，因此`shouldSkip`返回false，代码继续执行:

```java
ScopeMetadata scopeMetadata = this.scopeMetadataResolver.resolveScopeMetadata(abd);  
```

**处理Scope注解信息，默认是单例bean**    执行 :

```java
AnnotationConfigUtils.processCommonDefinitionAnnotations(abd);  
```

处理一些常见注解信息:

```java
static void processCommonDefinitionAnnotations(AnnotatedBeanDefinition abd, AnnotatedTypeMetadata metadata) {  
        if (metadata.isAnnotated(Lazy.class.getName())) {  
            abd.setLazyInit(attributesFor(metadata, Lazy.class).getBoolean("value"));  
        }  
        else if (abd.getMetadata() != metadata && abd.getMetadata().isAnnotated(Lazy.class.getName())) {  
            abd.setLazyInit(attributesFor(abd.getMetadata(), Lazy.class).getBoolean("value"));  
        }  
  
        if (metadata.isAnnotated(Primary.class.getName())) {  
            abd.setPrimary(true);  
        }  
        if (metadata.isAnnotated(DependsOn.class.getName())) {  
            abd.setDependsOn(attributesFor(metadata, DependsOn.class).getStringArray("value"));  
        }  
  
        if (abd instanceof AbstractBeanDefinition) {  
            AbstractBeanDefinition absBd = (AbstractBeanDefinition) abd;  
            if (metadata.isAnnotated(Role.class.getName())) {  
                absBd.setRole(attributesFor(metadata, Role.class).getNumber("value").intValue());  
            }  
            if (metadata.isAnnotated(Description.class.getName())) {  
                absBd.setDescription(attributesFor(metadata, Description.class).getString("value"));  
            }  
        }  
    }  
```

处理`Lazy、Primary、DependsOn、Role、Description`等注解;

最后调用:

```java
BeanDefinitionReaderUtils.registerBeanDefinition(definitionHolder, this.registry); 
```

注册bean信息，在注册bean信息之前通过:
```
String beanName = (name != null ? name : this.beanNameGenerator.generateBeanName(abd, this.registry));  
```

获取bean名称;bean的注册调用为:

```
registry.registerBeanDefinition(beanName, definitionHolder.getBeanDefinition());  

```
该代码在上篇中已有说明。



此时Application对应bean已创建完成。


***

over


***

[原文](http://blog.csdn.net/liaokailin/article/details/49048557)














