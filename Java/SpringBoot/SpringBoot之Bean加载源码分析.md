# SpringBoot之Bean加载源码分析

***
[toc]


@(SpringBoot)

*** 


本篇将阐述spring boot中bean的创建过程

(注释为Google翻译)


***


## refresh

首先来看`SpringApplication.run`方法中`refresh()`方法：


```java
	// Refresh the context  
    refresh(context);  
```
调用的是`AbstractApplicationContext`中的`refresh`方法

```java
protected void refresh(ApplicationContext applicationContext) {  
        Assert.isInstanceOf(AbstractApplicationContext.class, applicationContext);  
        ((AbstractApplicationContext) applicationContext).refresh();  
    }  
```
方法定义如下 :

```java
public void refresh() throws BeansException, IllegalStateException {  
        synchronized (this.startupShutdownMonitor) {  
            // Prepare this context for refreshing.  
            // 准备这个上下文来刷新
            prepareRefresh();  
  
            // Tell the subclass to refresh the internal bean factory.  
            // 告诉子类刷新内部bean工厂
            ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();  
  
            // Prepare the bean factory for use in this context.  
            // 准备在这种情况下使用的bean工厂
            prepareBeanFactory(beanFactory);  
  
            try {  
                // Allows post-processing of the bean factory in context subclasses.  
                // 允许在上下文子类中对bean工厂进行后处理。
                postProcessBeanFactory(beanFactory);  
  
                // Invoke factory processors registered as beans in the context.  
                // 在上下文中调用注册为bean的工厂处理器。
                invokeBeanFactoryPostProcessors(beanFactory);  
  
                // Register bean processors that intercept bean creation. 
                // 注册拦截bean创建的bean处理器。 
                registerBeanPostProcessors(beanFactory);  
  
                // Initialize message source for this context.  
                // 初始化此上下文的消息源。
                initMessageSource();  
  
                // Initialize event multicaster for this context.  
                // 初始化此上下文的事件多播器
                initApplicationEventMulticaster();  
  
                // Initialize other special beans in specific context subclasses.  
				// 在特定的上下文子类中初始化其他特殊的bean               
                onRefresh();  
  
                // Check for listener beans and register them.  
                // 检查监听器bean并注册它们。
                registerListeners();  
  
                // Instantiate all remaining (non-lazy-init) singletons. 
                // 实例化所有剩下的（非惰性初始化）单例。 
                finishBeanFactoryInitialization(beanFactory);  
  
                // Last step: publish corresponding event.  
                // 最后一步：发布相应的事件。
                finishRefresh();  
            }  
  
            catch (BeansException ex) {  
                logger.warn("Exception encountered during context initialization - cancelling refresh attempt", ex);  
  
                // Destroy already created singletons to avoid dangling resources.  
                // 销毁已经创建的单身人士以避免摇晃资源
                destroyBeans();  
  
                // Reset 'active' flag.  
                // 重置“有效”标志。
                cancelRefresh(ex);  
  
                // Propagate exception to caller.  
                // 向呼叫者传播异常。
                throw ex;  
            }  
        }  
    }  
```

该方法中涉及的过程非常多，需要一步步来分析

获取BeanFactory:


```java
ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();  
```
通过前面的文章应该知道对应的`BeanFactory`为`DefaultListableBeanFactory`
 
直奔主题来看如下方法:



```java
invokeBeanFactoryPostProcessors(beanFactory); 

--->

protected void invokeBeanFactoryPostProcessors(ConfigurableListableBeanFactory beanFactory) {  
        PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors(beanFactory,getBeanFactoryPostProcessors());  
    }  

```

首先来看`getBeanFactoryPostProcessors()`,其对应值为：`ConfigurationWarningsApplicationContextInitializer$ConfigurationWarningsPostProcessor、PropertySourceOrderingPostProcessor`

`ConfigurationWarningsApplicationContextInitializer`是在`ConfigurationWarningsApplicationContextInitializer`中执行:

```java
public void initialize(ConfigurableApplicationContext context) {  
        context.addBeanFactoryPostProcessor(new ConfigurationWarningsPostProcessor(  
                getChecks()));  
    }  
```

添加

`PropertySourceOrderingPostProcessor`是在`ConfigFileApplicationListener`执行

```java
protected void addPostProcessors(ConfigurableApplicationContext context) {  
        context.addBeanFactoryPostProcessor(new PropertySourceOrderingPostProcessor(  
                context));  
    } 
```
来看`invokeBeanFactoryPostProcessors`方法

```java
public static void invokeBeanFactoryPostProcessors(  
            ConfigurableListableBeanFactory beanFactory, List<BeanFactoryPostProcessor> beanFactoryPostProcessors) {  
  
        // Invoke BeanDefinitionRegistryPostProcessors first, if any.
        // 首先调用BeanDefinitionRegistryPostProcessor，如果有的话。  
        Set<String> processedBeans = new HashSet<String>();  
  
        if (beanFactory instanceof BeanDefinitionRegistry) {  
            ...//处理后处理器  
               
            String[] postProcessorNames =  
                    beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);  
  
            // First, invoke the BeanDefinitionRegistryPostProcessors that implement PriorityOrdered. 
            // 首先，调用实现PriorityOrdered的BeanDefinitionRegistryPostProcessor 
            List<BeanDefinitionRegistryPostProcessor> priorityOrderedPostProcessors = new ArrayList<BeanDefinitionRegistryPostProcessor>();  
            for (String ppName : postProcessorNames) {  
                if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {  
                    priorityOrderedPostProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));  
                    processedBeans.add(ppName);  
                }  
            }  
            OrderComparator.sort(priorityOrderedPostProcessors);  
            registryPostProcessors.addAll(priorityOrderedPostProcessors);  
            invokeBeanDefinitionRegistryPostProcessors(priorityOrderedPostProcessors, registry);  
  
            // Next, invoke the BeanDefinitionRegistryPostProcessors that implement Ordered. 
            // 接下来，调用实现Ordered的BeanDefinitionRegistryPostProcessors。
            postProcessorNames = beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);  
            List<BeanDefinitionRegistryPostProcessor> orderedPostProcessors = new ArrayList<BeanDefinitionRegistryPostProcessor>();  
            for (String ppName : postProcessorNames) {  
                if (!processedBeans.contains(ppName) && beanFactory.isTypeMatch(ppName, Ordered.class)) {  
                    orderedPostProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));  
                    processedBeans.add(ppName);  
                }  
            }  
            OrderComparator.sort(orderedPostProcessors);  
            registryPostProcessors.addAll(orderedPostProcessors);  
            invokeBeanDefinitionRegistryPostProcessors(orderedPostProcessors, registry);  
  
            // Finally, invoke all other BeanDefinitionRegistryPostProcessors until no further ones appear.  
            // 最后，调用所有其他的BeanDefinitionRegistryPostProcessor，直到没有其他的出现。
            boolean reiterate = true;  
            while (reiterate) {  
                reiterate = false;  
                postProcessorNames = beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);  
                for (String ppName : postProcessorNames) {  
                    if (!processedBeans.contains(ppName)) {  
                        BeanDefinitionRegistryPostProcessor pp = beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class);  
                        registryPostProcessors.add(pp);  
                        processedBeans.add(ppName);  
                        pp.postProcessBeanDefinitionRegistry(registry);  
                        reiterate = true;  
                    }  
                }  
            }  
  
            // Now, invoke the postProcessBeanFactory callback of all processors handled so far 执行后处理器  
            invokeBeanFactoryPostProcessors(registryPostProcessors, beanFactory);  
            invokeBeanFactoryPostProcessors(regularPostProcessors, beanFactory);  
        }  

```


```java
String[] postProcessorNames = beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false); 
```

按照`bean`的类型获取类型为`BeanDefinitionRegistryPostProcessor`的bean，这里获取到的bean名称为：
`org.springframework.context.annotation.internalConfigurationAnnotationProcessor`；对应的Class为`ConfigurationClassPostProcessor`

在前面文章中创建上下文的时候`beanfactory`创建了该bean。

经过排序后执行如下方法:

```java
invokeBeanDefinitionRegistryPostProcessors(priorityOrderedPostProcessors, registry);

--->

private static void invokeBeanDefinitionRegistryPostProcessors(  
            Collection<? extends BeanDefinitionRegistryPostProcessor> postProcessors, BeanDefinitionRegistry registry) {  
  
        for (BeanDefinitionRegistryPostProcessor postProcessor : postProcessors) {  
            postProcessor.postProcessBeanDefinitionRegistry(registry);  
        }  
    }   
```
实际调用`ConfigurationClassPostProcessor.postProcessBeanDefinitionRegistry`

```java
public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) {  
        ...//注册若干bean  
        processConfigBeanDefinitions(registry);  
    }  

//--->>processConfigBeanDefinitions(registry)如下：

public void processConfigBeanDefinitions(BeanDefinitionRegistry registry) {  
        Set<BeanDefinitionHolder> configCandidates = new LinkedHashSet<BeanDefinitionHolder>();  
        String[] candidateNames = registry.getBeanDefinitionNames();  
  
        for (String beanName : candidateNames) {  
            BeanDefinition beanDef = registry.getBeanDefinition(beanName);  
            if (ConfigurationClassUtils.isFullConfigurationClass(beanDef) ||  
                    ConfigurationClassUtils.isLiteConfigurationClass(beanDef)) {  
                if (logger.isDebugEnabled()) {  
                    logger.debug("Bean definition has already been processed as a configuration class: " + beanDef);  
                }  
            }  
            else if (ConfigurationClassUtils.checkConfigurationClassCandidate(beanDef, this.metadataReaderFactory)) {  
                configCandidates.add(new BeanDefinitionHolder(beanDef, beanName));  
            }  
        }  
  
        // Return immediately if no @Configuration classes were found  
        // 如果没有找到@Configuration类，立即返回
        if (configCandidates.isEmpty()) {  
            return;  
        }  
  
        // Detect any custom bean name generation strategy supplied through the enclosing application context  
        // 检测通过封闭应用程序上下文提供的任何自定义bean名称生成策略
        SingletonBeanRegistry singletonRegistry = null;  
        if (registry instanceof SingletonBeanRegistry) {  
            singletonRegistry = (SingletonBeanRegistry) registry;  
            if (!this.localBeanNameGeneratorSet && singletonRegistry.containsSingleton(CONFIGURATION_BEAN_NAME_GENERATOR)) {  
                BeanNameGenerator generator = (BeanNameGenerator) singletonRegistry.getSingleton(CONFIGURATION_BEAN_NAME_GENERATOR);  
                this.componentScanBeanNameGenerator = generator;  
                this.importBeanNameGenerator = generator;  
            }  
        }  
  
        // Parse each @Configuration class  
        // 解析每个@Configuration类
        ConfigurationClassParser parser = new ConfigurationClassParser(  
                this.metadataReaderFactory, this.problemReporter, this.environment,  
                this.resourceLoader, this.componentScanBeanNameGenerator, registry);  
  
        Set<ConfigurationClass> alreadyParsed = new HashSet<ConfigurationClass>(configCandidates.size());  
        do {  
            parser.parse(configCandidates);  
            parser.validate();  
  
            Set<ConfigurationClass> configClasses = new LinkedHashSet<ConfigurationClass>(parser.getConfigurationClasses());  
            configClasses.removeAll(alreadyParsed);  
  
            // Read the model and create bean definitions based on its content  
            // 阅读模型并根据其内容创建bean定义
            if (this.reader == null) {  
                this.reader = new ConfigurationClassBeanDefinitionReader(registry, this.sourceExtractor,  
                        this.problemReporter, this.metadataReaderFactory, this.resourceLoader, this.environment,  
                        this.importBeanNameGenerator, parser.getImportRegistry());  
            }  
            this.reader.loadBeanDefinitions(configClasses);  
            alreadyParsed.addAll(configClasses);  
  
            configCandidates.clear();  
            if (registry.getBeanDefinitionCount() > candidateNames.length) {  
                String[] newCandidateNames = registry.getBeanDefinitionNames();  
                Set<String> oldCandidateNames = new HashSet<String>(Arrays.asList(candidateNames));  
                Set<String> alreadyParsedClasses = new HashSet<String>();  
                for (ConfigurationClass configurationClass : alreadyParsed) {  
                    alreadyParsedClasses.add(configurationClass.getMetadata().getClassName());  
                }  
                for (String candidateName : newCandidateNames) {  
                    if (!oldCandidateNames.contains(candidateName)) {  
                        BeanDefinition beanDef = registry.getBeanDefinition(candidateName);  
                        if (ConfigurationClassUtils.checkConfigurationClassCandidate(beanDef, this.metadataReaderFactory) &&  
                                !alreadyParsedClasses.contains(beanDef.getBeanClassName())) {  
                            configCandidates.add(new BeanDefinitionHolder(beanDef, candidateName));  
                        }  
                    }  
                }  
                candidateNames = newCandidateNames;  
            }  
        }  
        while (!configCandidates.isEmpty());  
  
        // Register the ImportRegistry as a bean in order to support ImportAware @Configuration classes  
        // 将ImportRegistry注册为一个bean，以支持ImportAware @Configuration类
        if (singletonRegistry != null) {  
            if (!singletonRegistry.containsSingleton(IMPORT_REGISTRY_BEAN_NAME)) {  
                singletonRegistry.registerSingleton(IMPORT_REGISTRY_BEAN_NAME, parser.getImportRegistry());  
            }  
        }  
  
        if (this.metadataReaderFactory instanceof CachingMetadataReaderFactory) {  
            ((CachingMetadataReaderFactory) this.metadataReaderFactory).clearCache();  
        }  
    }  
```
又是一段很长的代码

```java
// 获取已经注册的bean名称，其信息为:
String[] candidateNames = registry.getBeanDefinitionNames();  
```
这里看到上一篇中创建的Application对应bean
```java
for (String beanName : candidateNames) {  
            BeanDefinition beanDef = registry.getBeanDefinition(beanName);  
            if (ConfigurationClassUtils.isFullConfigurationClass(beanDef) ||  
                    ConfigurationClassUtils.isLiteConfigurationClass(beanDef)) {  
                if (logger.isDebugEnabled()) {  
                    logger.debug("Bean definition has already been processed as a configuration class: " + beanDef);  
                }  
            }  
            else if (ConfigurationClassUtils.checkConfigurationClassCandidate(beanDef, this.metadataReaderFactory)) {  
                configCandidates.add(new BeanDefinitionHolder(beanDef, beanName));  
            }  
        }  
```
判断对应bean是否为配置文件bean（包含Configuration注解），经过筛选只有Application对应bean满足条件:

```java
ConfigurationClassParser parser = new ConfigurationClassParser(  
            this.metadataReaderFactory, this.problemReporter, this.environment,  
            this.resourceLoader, this.componentScanBeanNameGenerator, registry);
```
该代码构造了`Configuration`类解析器
执行

```java
parser.parse(configCandidates);  


--->

public void parse(Set<BeanDefinitionHolder> configCandidates) {  
        this.deferredImportSelectors = new LinkedList<DeferredImportSelectorHolder>();  
  
        for (BeanDefinitionHolder holder : configCandidates) {  
            BeanDefinition bd = holder.getBeanDefinition();  
            try {  
                if (bd instanceof AnnotatedBeanDefinition) {   //执行该部分代码  
                    parse(((AnnotatedBeanDefinition) bd).getMetadata(), holder.getBeanName());  
                }  
                else if (bd instanceof AbstractBeanDefinition && ((AbstractBeanDefinition) bd).hasBeanClass()) {  
                    parse(((AbstractBeanDefinition) bd).getBeanClass(), holder.getBeanName());  
                }  
                else {  
                    parse(bd.getBeanClassName(), holder.getBeanName());  
                }  
            }  
            catch (BeanDefinitionStoreException ex) {  
                throw ex;  
            }  
            catch (Exception ex) {  
                throw new BeanDefinitionStoreException(  
                        "Failed to parse configuration class [" + bd.getBeanClassName() + "]", ex);  
            }  
        }  
  
        processDeferredImportSelectors();  
    }  


// 调用 ---->

parse(((AnnotatedBeanDefinition) bd).getMetadata(), holder.getBeanName()); 




// 最终调用------------>

protected void processConfigurationClass(ConfigurationClass configClass) throws IOException {  
        if (this.conditionEvaluator.shouldSkip(configClass.getMetadata(), ConfigurationPhase.PARSE_CONFIGURATION)) {  
            return;  
        }  
  
        ConfigurationClass existingClass = this.configurationClasses.get(configClass);  
        if (existingClass != null) {  
            if (configClass.isImported()) {  
                if (existingClass.isImported()) {  
                    existingClass.mergeImportedBy(configClass);  
                }  
                // Otherwise ignore new imported config class; existing non-imported class overrides it.  
                return;  
            }  
            else {  
                // Explicit bean definition found, probably replacing an import.  
                // Let's remove the old one and go with the new one.  
                this.configurationClasses.remove(configClass);  
                for (Iterator<ConfigurationClass> it = this.knownSuperclasses.values().iterator(); it.hasNext(); ) {  
                    if (configClass.equals(it.next())) {  
                        it.remove();  
                    }  
                }  
            }  
        }  
  
        // Recursively process the configuration class and its superclass hierarchy.  
        SourceClass sourceClass = asSourceClass(configClass);  
        do {  
            sourceClass = doProcessConfigurationClass(configClass, sourceClass);  
        }  
        while (sourceClass != null);  
  
        this.configurationClasses.put(configClass, configClass);  
    }  

```

首先判断该bean是否被跳过（该部分代码上一篇已说明），然后对Class进行包装，调用`sourceClass = doProcessConfigurationClass(configClass,sourceClass)`处理Application类




## 解析Configuration注解
```java
protected final SourceClass doProcessConfigurationClass(ConfigurationClass configClass, SourceClass sourceClass) throws IOException {  
        // Recursively process any member (nested) classes first 
        // 首先递归处理任何成员（嵌套）类 
        processMemberClasses(configClass, sourceClass);  
  
        // Process any @PropertySource annotations  
        // 处理任何@PropertySource注释
        for (AnnotationAttributes propertySource : AnnotationConfigUtils.attributesForRepeatable(  
                sourceClass.getMetadata(), PropertySources.class, org.springframework.context.annotation.PropertySource.class)) {  
            if (this.environment instanceof ConfigurableEnvironment) {  
                processPropertySource(propertySource);  
            }  
            else {  
                logger.warn("Ignoring @PropertySource annotation on [" + sourceClass.getMetadata().getClassName() +  
                        "]. Reason: Environment must implement ConfigurableEnvironment");  
            }  
        }  
  
        // Process any @ComponentScan annotations  
        // 处理任何@ComponentScan注释
        AnnotationAttributes componentScan = AnnotationConfigUtils.attributesFor(sourceClass.getMetadata(), ComponentScan.class);  
        if (componentScan != null && !this.conditionEvaluator.shouldSkip(sourceClass.getMetadata(), ConfigurationPhase.REGISTER_BEAN)) {  
            // The config class is annotated with @ComponentScan -> perform the scan immediately 
            // 配置类用@ComponentScan注释 - >立即执行扫描 
            Set<BeanDefinitionHolder> scannedBeanDefinitions =  
                    this.componentScanParser.parse(componentScan, sourceClass.getMetadata().getClassName());  
            // Check the set of scanned definitions for any further config classes and parse recursively if necessary  
            // 检查任何进一步配置类的扫描定义集合，并在必要时递归解析
            for (BeanDefinitionHolder holder : scannedBeanDefinitions) {  
                if (ConfigurationClassUtils.checkConfigurationClassCandidate(holder.getBeanDefinition(), this.metadataReaderFactory)) {  
                    parse(holder.getBeanDefinition().getBeanClassName(), holder.getBeanName());  
                }  
            }  
        }  
  
        // Process any @Import annotations  
        // 处理任何@Import注释
        processImports(configClass, sourceClass, getImports(sourceClass), true);  
  
        // Process any @ImportResource annotations
        // 处理任何@ImportResource注释  
        if (sourceClass.getMetadata().isAnnotated(ImportResource.class.getName())) {  
            AnnotationAttributes importResource = AnnotationConfigUtils.attributesFor(sourceClass.getMetadata(), ImportResource.class);  
            String[] resources = importResource.getStringArray("value");  
            Class<? extends BeanDefinitionReader> readerClass = importResource.getClass("reader");  
            for (String resource : resources) {  
                String resolvedResource = this.environment.resolveRequiredPlaceholders(resource);  
                configClass.addImportedResource(resolvedResource, readerClass);  
            }  
        }  
  
        // Process individual @Bean methods  
        // Process individual @Bean methods 
        Set<MethodMetadata> beanMethods = sourceClass.getMetadata().getAnnotatedMethods(Bean.class.getName());  
        for (MethodMetadata methodMetadata : beanMethods) {  
            configClass.addBeanMethod(new BeanMethod(methodMetadata, configClass));  
        }  
  
        // Process superclass, if any  
        // 进程超类，如果有的话
        if (sourceClass.getMetadata().hasSuperClass()) {  
            String superclass = sourceClass.getMetadata().getSuperClassName();  
            if (!superclass.startsWith("java") && !this.knownSuperclasses.containsKey(superclass)) {  
                this.knownSuperclasses.put(superclass, configClass);  
                // Superclass found, return its annotation metadata and recurse  
                return sourceClass.getSuperClass();  
            }  
        }  
  
        // No superclass -> processing is complete  
        // 没有超类 - >处理完成
        return null;  
    }  
```
**到这里就看到了如何去解析`Application`类**

```java
processMemberClasses(configClass, sourceClass);  
```

处理其中内部类，解析内部类的过程和外部类相似，因此继续看下面的代码:



## 处理PropertySource注解


```java
		// Process any @PropertySource annotations  
		// 处理任何@PropertySource注释
        for (AnnotationAttributes propertySource : AnnotationConfigUtils.attributesForRepeatable(  
                sourceClass.getMetadata(), PropertySources.class, org.springframework.context.annotation.PropertySource.class)) {  
            if (this.environment instanceof ConfigurableEnvironment) {  
                processPropertySource(propertySource);  
            }  
            else {  
                logger.warn("Ignoring @PropertySource annotation on [" + sourceClass.getMetadata().getClassName() +  
                        "]. Reason: Environment must implement ConfigurableEnvironment");  
            }  
        }
```
其核心操作：

```java
private void processPropertySource(AnnotationAttributes propertySource) throws IOException {  
        String name = propertySource.getString("name");  
        String[] locations = propertySource.getStringArray("value");  
        boolean ignoreResourceNotFound = propertySource.getBoolean("ignoreResourceNotFound");  
        Assert.isTrue(locations.length > 0, "At least one @PropertySource(value) location is required");  
        for (String location : locations) {  
            try {  
                String resolvedLocation = this.environment.resolveRequiredPlaceholders(location);  
                Resource resource = this.resourceLoader.getResource(resolvedLocation);  
                ResourcePropertySource rps = (StringUtils.hasText(name) ?  
                        new ResourcePropertySource(name, resource) : new ResourcePropertySource(resource));  
                addPropertySource(rps);  
            }  
            catch (IllegalArgumentException ex) {  
                // from resolveRequiredPlaceholders  
                if (!ignoreResourceNotFound) {  
                    throw ex;  
                }  
            }  
            catch (FileNotFoundException ex) {  
                // from ResourcePropertySource constructor  
                if (!ignoreResourceNotFound) {  
                    throw ex;  
                }  
            }  
        }  
    }  
```
 通过注解中的信息获取资源信息，然后添加到`MutablePropertySourcespropertySources = ((ConfigurableEnvironment)this.environment).getPropertySources()`中，该内容前面已有讲述;





## 解析ComponentScan注解
```java
// Process any @ComponentScan annotations  
        AnnotationAttributes componentScan = AnnotationConfigUtils.attributesFor(sourceClass.getMetadata(), ComponentScan.class);  
        if (componentScan != null && !this.conditionEvaluator.shouldSkip(sourceClass.getMetadata(), ConfigurationPhase.REGISTER_BEAN)) {  
            // The config class is annotated with @ComponentScan -> perform the scan immediately  
            Set<BeanDefinitionHolder> scannedBeanDefinitions =  
                    this.componentScanParser.parse(componentScan, sourceClass.getMetadata().getClassName());  
            // Check the set of scanned definitions for any further config classes and parse recursively if necessary  
            for (BeanDefinitionHolder holder : scannedBeanDefinitions) {  
                if (ConfigurationClassUtils.checkConfigurationClassCandidate(holder.getBeanDefinition(), this.metadataReaderFactory)) {  
                    parse(holder.getBeanDefinition().getBeanClassName(), holder.getBeanName());  
                }  
            }  
        }  
```

`ComponentScan`注解的作用大家都明白，扫描执行路径下bean信息，那么具体是如何实现的？需要跟进去看代码，调用

```java
Set<BeanDefinitionHolder> scannedBeanDefinitions =  
                    this.componentScanParser.parse(componentScan,sourceClass.getMetadata().getClassName());  
```


```java
public Set<BeanDefinitionHolder> parse(AnnotationAttributes componentScan, final String declaringClass) {  
         ...//通过注解中的信息设置扫描器的参数信息  
        return scanner.doScan(StringUtils.toStringArray(basePackages));  
    }  
```

代码中忽略了扫描器对应的参数信息，直接看`doScan`方法:
```java
protected Set<BeanDefinitionHolder> doScan(String... basePackages) {  
    Assert.notEmpty(basePackages, "At least one base package must be specified");  
    Set<BeanDefinitionHolder> beanDefinitions = new LinkedHashSet<BeanDefinitionHolder>();  
    for (String basePackage : basePackages) {  
        Set<BeanDefinition> candidates = findCandidateComponents(basePackage);  
        for (BeanDefinition candidate : candidates) {  
            ScopeMetadata scopeMetadata = this.scopeMetadataResolver.resolveScopeMetadata(candidate);  
            candidate.setScope(scopeMetadata.getScopeName());  
            String beanName = this.beanNameGenerator.generateBeanName(candidate, this.registry);  
            if (candidate instanceof AbstractBeanDefinition) {  
                postProcessBeanDefinition((AbstractBeanDefinition) candidate, beanName);  
            }  
            if (candidate instanceof AnnotatedBeanDefinition) {  
                AnnotationConfigUtils.processCommonDefinitionAnnotations((AnnotatedBeanDefinition) candidate);  
            }  
            if (checkCandidate(beanName, candidate)) {  
                BeanDefinitionHolder definitionHolder = new BeanDefinitionHolder(candidate, beanName);  
                definitionHolder = AnnotationConfigUtils.applyScopedProxyMode(scopeMetadata, definitionHolder, this.registry);  
                beanDefinitions.add(definitionHolder);  
                registerBeanDefinition(definitionHolder, this.registry);  
            }  
        }  
    }  
    return beanDefinitions;  
```


```java

// 遍历basePackages信息，

Set<BeanDefinition> candidates = findCandidateComponents(basePackage);  


// --------->查询类路径下申明的组件信息

public Set<BeanDefinition> findCandidateComponents(String basePackage) {  
        Set<BeanDefinition> candidates = new LinkedHashSet<BeanDefinition>();  
        try {  
            String packageSearchPath = ResourcePatternResolver.CLASSPATH_ALL_URL_PREFIX +  
                    resolveBasePackage(basePackage) + "/" + this.resourcePattern;  
            Resource[] resources = this.resourcePatternResolver.getResources(packageSearchPath);  
            boolean traceEnabled = logger.isTraceEnabled();  
            boolean debugEnabled = logger.isDebugEnabled();  
            for (Resource resource : resources) {  
                if (traceEnabled) {  
                    logger.trace("Scanning " + resource);  
                }  
                if (resource.isReadable()) {  
                    try {  
                        MetadataReader metadataReader = this.metadataReaderFactory.getMetadataReader(resource);  
                        if (isCandidateComponent(metadataReader)) {  
                            ScannedGenericBeanDefinition sbd = new ScannedGenericBeanDefinition(metadataReader);  
                            sbd.setResource(resource);  
                            sbd.setSource(resource);  
                            if (isCandidateComponent(sbd)) {  
                                if (debugEnabled) {  
                                    logger.debug("Identified candidate component class: " + resource);  
                                }  
                                candidates.add(sbd);  
                            }  
                            else {  
                                if (debugEnabled) {  
                                    logger.debug("Ignored because not a concrete top-level class: " + resource);  
                                }  
                            }  
                        }  
                        else {  
                            if (traceEnabled) {  
                                logger.trace("Ignored because not matching any filter: " + resource);  
                            }  
                        }  
                    }  
                    catch (Throwable ex) {  
                        throw new BeanDefinitionStoreException(  
                                "Failed to read candidate component class: " + resource, ex);  
                    }  
                }  
                else {  
                    if (traceEnabled) {  
                        logger.trace("Ignored because not readable: " + resource);  
                    }  
                }  
            }  
        }  
        catch (IOException ex) {  
            throw new BeanDefinitionStoreException("I/O failure during classpath scanning", ex);  
        }  
        return candidates;  
    }  

// 看-------->
Resource[] resources = this.resourcePatternResolver.getResources(packageSearchPath);  


// -------->


public Resource[] getResources(String locationPattern) throws IOException {  
        Assert.notNull(locationPattern, "Location pattern must not be null");  
        if (locationPattern.startsWith(CLASSPATH_ALL_URL_PREFIX)) {  
            // a class path resource (multiple resources for same name possible)  
            if (getPathMatcher().isPattern(locationPattern.substring(CLASSPATH_ALL_URL_PREFIX.length()))) {  
                // a class path resource pattern  
                return findPathMatchingResources(locationPattern);  
            }  
            else {  
                // all class path resources with the given name  
                return findAllClassPathResources(locationPattern.substring(CLASSPATH_ALL_URL_PREFIX.length()));  
            }  
        }  
        else {  
            // Only look for a pattern after a prefix here  
            // (to not get fooled by a pattern symbol in a strange prefix).  
            int prefixEnd = locationPattern.indexOf(":") + 1;  
            if (getPathMatcher().isPattern(locationPattern.substring(prefixEnd))) {  
                // a file pattern  
                return findPathMatchingResources(locationPattern);  
            }  
            else {  
                // a single resource with the given name  
                return new Resource[] {getResourceLoader().getResource(locationPattern)};  
            }  
        }  
    }  


```

解析路径信息，这里spring有自己的一套继续规则，通过`findPathMatchingResources()`检索到指定类路径下所有的`*.class`文件，然后调用`findAllClassPathResources`解析Class文件

```java
protected Resource[] findAllClassPathResources(String location) throws IOException {  
        String path = location;  
        if (path.startsWith("/")) {  
            path = path.substring(1);  
        }  
        Set<Resource> result = doFindAllClassPathResources(path);  
        return result.toArray(new Resource[result.size()]);  
    }  
```

```java
protected Set<Resource> doFindAllClassPathResources(String path) throws IOException {  
        Set<Resource> result = new LinkedHashSet<Resource>(16);  
        ClassLoader cl = getClassLoader();  
        Enumeration<URL> resourceUrls = (cl != null ? cl.getResources(path) : ClassLoader.getSystemResources(path));  
        while (resourceUrls.hasMoreElements()) {  
            URL url = resourceUrls.nextElement();  
            result.add(convertClassLoaderURL(url));  
        }  
        if ("".equals(path)) {  
            // The above result is likely to be incomplete, i.e. only containing file system references.  
            // 上述结果可能不完整，即只包含文件系统引用。
            // We need to have pointers to each of the jar files on the classpath as well... 
            // 我们需要指向类路径上的每个jar文件以及...
            addAllClassLoaderJarRoots(cl, result);  
        }  
        return result;  
    }  
```


通过上面的代码可以发现，在获取到path路径以后spring采用类加载器获取指定Class文件对应的资源信息,获取完资源信息后调用:

```java
MetadataReader metadataReader = this.metadataReaderFactory.getMetadataReader(resource);  


//  解析资源信息对应的元数据------>

if (isCandidateComponent(metadataReader)) {  
                            ScannedGenericBeanDefinition sbd = new ScannedGenericBeanDefinition(metadataReader);  
                            sbd.setResource(resource);  
                            sbd.setSource(resource);  
                            if (isCandidateComponent(sbd)) {  
                                if (debugEnabled) {  
                                    logger.debug("Identified candidate component class: " + resource);  
                                }  
                                candidates.add(sbd);  
                            }  
                            else {  
                                if (debugEnabled) {  
                                    logger.debug("Ignored because not a concrete top-level class: " + resource);  
                                }  
                            }  
                        }  
                        else {  
                            if (traceEnabled) {  
                                logger.trace("Ignored because not matching any filter: " + resource);  
                            }  
                        }  
```

如果存在`Componment`注解修饰的Class文件则加入到`BeanDefinition`集合中返回。

```java
for (BeanDefinitionHolder holder : scannedBeanDefinitions) {  
                if (ConfigurationClassUtils.checkConfigurationClassCandidate(holder.getBeanDefinition(), this.metadataReaderFactory)) {  
                    parse(holder.getBeanDefinition().getBeanClassName(), holder.getBeanName());  
                }  
            }  
```
遍历扫描到的bean信息，如果为配置bean，则执行parse方法，该方法调用`processConfigurationClass`，形成一个递归的操作。

```java

```


```java

```

```java

```


```java

```


## 解析Import注解

```java

processImports(configClass, sourceClass, getImports(sourceClass), true);

```
处理`import`注解，该注解在spring boot中使用非常频繁

```java
private void processImports(ConfigurationClass configClass, SourceClass currentSourceClass,  
            Collection<SourceClass> importCandidates, boolean checkForCircularImports) throws IOException {  
  
         ...  
           
            this.importStack.push(configClass);  
            try {  
                for (SourceClass candidate : importCandidates) {  
                    if (candidate.isAssignable(ImportSelector.class)) {  
                        // Candidate class is an ImportSelector -> delegate to it to determine imports 
                        // 候选类是一个ImportSelector - >委托它来确定导入 
                        Class<?> candidateClass = candidate.loadClass();  
                        ImportSelector selector = BeanUtils.instantiateClass(candidateClass, ImportSelector.class);  
                        invokeAwareMethods(selector);  
                        if (this.deferredImportSelectors != null && selector instanceof DeferredImportSelector) {  
                            this.deferredImportSelectors.add(  
                                    new DeferredImportSelectorHolder(configClass, (DeferredImportSelector) selector));  
                        }  
                        else {  
                            String[] importClassNames = selector.selectImports(currentSourceClass.getMetadata());  
                            Collection<SourceClass> importSourceClasses = asSourceClasses(importClassNames);  
                            processImports(configClass, currentSourceClass, importSourceClasses, false);  
                        }  
                    }  
                    else if (candidate.isAssignable(ImportBeanDefinitionRegistrar.class)) {  
                        // Candidate class is an ImportBeanDefinitionRegistrar -> 
                        // 候选类是一个ImportBeanDefinitionRegistrar - > 
                        // delegate to it to register additional bean definitions  
                        // 委托它来注册更多的bean定义
                        Class<?> candidateClass = candidate.loadClass();  
                        ImportBeanDefinitionRegistrar registrar =  
                                BeanUtils.instantiateClass(candidateClass, ImportBeanDefinitionRegistrar.class);  
                        invokeAwareMethods(registrar);  
                        configClass.addImportBeanDefinitionRegistrar(registrar, currentSourceClass.getMetadata());  
                    }  
                    else {  
                        // Candidate class not an ImportSelector or ImportBeanDefinitionRegistrar -> 
                        // 候选类不是ImportSelector或ImportBeanDefinitionRegistrar - > 
                        // process it as an @Configuration class  
                        // 将其作为@Configuration类来处理
                        this.importStack.registerImport(  
                                currentSourceClass.getMetadata(), candidate.getMetadata().getClassName());  
                        processConfigurationClass(candidate.asConfigClass(configClass));  
                    }  
                }  
            }  
            catch (BeanDefinitionStoreException ex) {  
                throw ex;  
            }  
            catch (Exception ex) {  
                throw new BeanDefinitionStoreException("Failed to process import candidates for configuration class [" +  
                        configClass.getMetadata().getClassName() + "]", ex);  
            }  
            finally {  
                this.importStack.pop();  
            }  
        }  
    }  
```

如果Import注解中Class为`ImportSelector`子类，通过`invokeAwareMethods(selector)`设置`aware`值，
如果类型为`DeferredImportSelector`则添加到`deferredImportSelectors`集合中，待前面的`parser.parse(configCandidates)`方法中`processDeferredImportSelectors()`处理；

如果不是，则执行`selectImports`方法，将获取到的结果递归调用`processImports`，解析`selectImports`得到的结果

如果Import注解中Class为`ImportBeanDefinitionRegistrar`子类，则添加到`importBeanDefinitionRegistrars`中，注意该部分的数据在执行完`parser.parse(configCandidates)`后调用`this.reader.loadBeanDefinitions(configClasses)`解析

否则执行配置信息的解析操作。



## 解析ImportResource注解



```java
// Process any @ImportResource annotations  
        if (sourceClass.getMetadata().isAnnotated(ImportResource.class.getName())) {  
            AnnotationAttributes importResource = AnnotationConfigUtils.attributesFor(sourceClass.getMetadata(), ImportResource.class);  
            String[] resources = importResource.getStringArray("value");  
            Class<? extends BeanDefinitionReader> readerClass = importResource.getClass("reader");  
            for (String resource : resources) {  
                String resolvedResource = this.environment.resolveRequiredPlaceholders(resource);  
                configClass.addImportedResource(resolvedResource, readerClass);  
            }  
        }  
```


## 解析Bean注解



```java
Set<MethodMetadata> beanMethods = sourceClass.getMetadata().getAnnotatedMethods(Bean.class.getName());  
        for (MethodMetadata methodMetadata : beanMethods) {  
            configClass.addBeanMethod(new BeanMethod(methodMetadata, configClass));  
        } 
```


上面这两个注解相对来讲要简单一些，至此bean的解析完成，这里面涉及到多重递归，首先理清楚一条线才能把代码看明白。


--------



OVER



[原文](https://blog.csdn.net/liaokailin/article/details/49107209)
