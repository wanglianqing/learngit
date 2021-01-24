# SpringIoc容器的创建及初始化
## 一、SpringIco容器创建及初始化的主逻辑
AbstractApplicationContext中的refresh方法包含了整个Ioc容器的创建及初始化。让我们来逐一分析下这个refresh方法。
```java
@Override
public void refresh() throws BeansException, IllegalStateException {
    synchronized (this.startupShutdownMonitor) {
        // Prepare this context for refreshing.
        prepareRefresh();

        // Tell the subclass to refresh the internal bean factory.
        ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

        // Prepare the bean factory for use in this context.
        prepareBeanFactory(beanFactory);

        try {
            // Allows post-processing of the bean factory in context subclasses.
            postProcessBeanFactory(beanFactory);

            // Invoke factory processors registered as beans in the context.
            invokeBeanFactoryPostProcessors(beanFactory);

            // Register bean processors that intercept bean creation.
            registerBeanPostProcessors(beanFactory);

            // Initialize message source for this context.
            initMessageSource();

            // Initialize event multicaster for this context.
            initApplicationEventMulticaster();

            // Initialize other special beans in specific context subclasses.
            onRefresh();

            // Check for listener beans and register them.
            registerListeners();

            // Instantiate all remaining (non-lazy-init) singletons.
            finishBeanFactoryInitialization(beanFactory);

            // Last step: publish corresponding event.
            finishRefresh();
        }

        catch (BeansException ex) {
            if (logger.isWarnEnabled()) {
                logger.warn("Exception encountered during context initialization - " +
                        "cancelling refresh attempt: " + ex);
            }

            // Destroy already created singletons to avoid dangling resources.
            destroyBeans();

            // Reset 'active' flag.
            cancelRefresh(ex);

            // Propagate exception to caller.
            throw ex;
        }

        finally {
            // Reset common introspection caches in Spring's core, since we
            // might not ever need metadata for singleton beans anymore...
            resetCommonCaches();
        }
    }
}
```

## 二、prepareRefresh();
>Prepare this context for refreshing.
* 记录启动时间，和容器启动相关的变量
* initPropertySources();   
  默认实现为空的方法，供子类重写。
  在web相关的容器(`GenericWebApplicationContext`)中，会拿到`ServletContext`中的内容，然后放入容器中(environment)
* getEnvironment().validateRequiredProperties()
  跟进源码会看到，其实是验证`AbstractPropertyResolver`中的`Set<String> requiredProperties`定义的值是否在properties中存在，不在的会加异常。**但是不知道怎么在spring中用**
* this.earlyApplicationEvents = new LinkedHashSet<>();
  初始化了一个earlyApplicationEvents

## 三、ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();
>Tell the subclass to refresh the internal bean factory.
* refreshBeanFactory()
  默认没有实现，在`GenericApplicationContext`实现是设置refresh为true 并且设置beanFactory的serializationId
* getBeanFactory()
  默认没有实现，在`GenericApplicationContext`实现是得到之前创建的beanFactory类型为：`ConfigurableListableBeanFactory`
  
## 四、prepareBeanFactory(beanFactory);
>Prepare the bean factory for use in this context.
* 设置BeanFactory的类加载器，表达式解析器...
    ```java
    beanFactory.setBeanClassLoader(getClassLoader());
    beanFactory.setBeanExpressionResolver(new StandardBeanExpressionResolver(beanFactory.getBeanClassLoader()));
    beanFactory.addPropertyEditorRegistrar(new ResourceEditorRegistrar(this, getEnvironment()));
    ```
* 添加了BeanPostProcessor,`ApplicationContextAwareProcessor`
    ```java
    beanFactory.addBeanPostProcessor(new ApplicationContextAwareProcessor(this));
    ```
* 设置忽略自动装配的接接口
    ```java
    beanFactory.ignoreDependencyInterface(EnvironmentAware.class);
    beanFactory.ignoreDependencyInterface(EmbeddedValueResolverAware.class);
    beanFactory.ignoreDependencyInterface(ResourceLoaderAware.class);
    beanFactory.ignoreDependencyInterface(ApplicationEventPublisherAware.class);
    beanFactory.ignoreDependencyInterface(MessageSourceAware.class);
    beanFactory.ignoreDependencyInterface(ApplicationContextAware.class);
    ```
* 设置可以解析的自动装配，我们可以直接在任组件中自动注入的类型
    ```java
    beanFactory.registerResolvableDependency(BeanFactory.class, beanFactory);
    beanFactory.registerResolvableDependency(ResourceLoader.class, this);
    beanFactory.registerResolvableDependency(ApplicationEventPublisher.class, this);
    beanFactory.registerResolvableDependency(ApplicationContext.class, this);
    ```
* 添加了BeanPostProcessor,`ApplicationListenerDetector`
* 添加编译时的AspectJ支持
    ```java
    if (beanFactory.containsBean(LOAD_TIME_WEAVER_BEAN_NAME)) {
        beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
        // Set a temporary ClassLoader for type matching.
        beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
    }
    ```
* 给容器添加有用的组件
  environment -- ConfigurableEnvironment   
  systemProperties -- Map\<String, Object\>   
  systemEnvironment -- Map\<String, Object\>   
  

## 五、postProcessBeanFactory(beanFactory);
>Allows post-processing of the bean factory in context subclasses.   

子类通过重写这个方法在BeanFactory创建并预准备完成以后做进一步的设置

---
之前step是BeanFactory的创建和预准备工作

---

## 六、invokeBeanFactoryPostProcessors(beanFactory);
>Invoke factory processors registered as beans in the context.

1. BeanFactoryPostProcessor有两个重要的接口，`BeanDefinitionRegistryPostProcessor`和`BeanFactoryPostProcessor`并且`BeanDefinitionRegistryPostProcessor`继承自`BeanFactoryPostProcessor`   
2. AbstractApplicationContext有自己的beanFactoryPostProcessors,没注册到Ioc
```java
PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors(beanFactory, getBeanFactoryPostProcessors());
这段的getBeanFactoryPostProcessors()取得就是AbstractApplicationContext自己的beanFactoryPostProcessors
```

### 6-1 执行传入的BeanFactoryPostProcessor
* 判断传入BeanFactoryPostProcessor如果是`BeanDefinitionRegistryPostProcessor`先执行方法`postProcessBeanDefinitionRegistry`   

### 6-2 执行Ioc容器的`BeanDefinitionRegistryPostProcessor`
* 在所有的`BeanFactoryPostProcessor`中找出所有的`BeanDefinitionRegistryPostProcessor`
* 通过getBean发放在Ioc容器中取`BeanDefinitionRegistryPostProcessor`实例，这样没有实例的就创建了。
* 排序根据`PriorityOrdered`或`Ordered`,如没有上述两个接口则不拍序。
* 调用方法：`postProcessBeanDefinitionRegistry`     

**这里注意，以上步骤会重复3次**
+ 第一次对实现了`PriorityOrdered`接口的post processor
+ 第二次对实现了`Ordered`接口的post processor
+ 第三次对没实现上述2个接口的post processor

代码片段   
```java
List<BeanDefinitionRegistryPostProcessor> currentRegistryProcessors = new ArrayList<>();

// First, invoke the BeanDefinitionRegistryPostProcessors that implement PriorityOrdered.
String[] postProcessorNames =
        beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
for (String ppName : postProcessorNames) {
    if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
        currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
        processedBeans.add(ppName);
    }
}
sortPostProcessors(currentRegistryProcessors, beanFactory);
registryProcessors.addAll(currentRegistryProcessors);
invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry);
currentRegistryProcessors.clear();
```

### 6-3 执行传入的BeanFactoryPostProcessor
* 把所有参数传入的`BeanFactoryPostProcessor`中的`postProcessBeanFactory`都执行一遍   

### 6-4 执行Ioc容器中的`BeanFactoryPostProcessor`   
执行过程跟`BeanDefinitionRegistryPostProcessor`非常相似
* 在所有的`BeanFactoryPostProcessor`
* 通过getBean发放在Ioc容器中取`BeanFactoryPostProcessor`实例，这样没有实例的就创建了。
* 排序根据`PriorityOrdered`或`Ordered`,如没有上述两个接口则不拍序。
* 调用方法：`postProcessBeanFactory`       

**这里注意，以上步骤会重复3次**
+ 第一次对实现了`PriorityOrdered`接口的post processor
+ 第二次对实现了`Ordered`接口的post processor
+ 第三次对没实现上述2个接口的post processor


## 七、registerBeanPostProcessors(beanFactory);
>Register bean processors that intercept bean creation.

注册Bean的后置处理器，拦截整个bean的创建过程
```java
PostProcessorRegistrationDelegate.registerBeanPostProcessors(beanFactory, this);
```

BeaPostProcessor有4个重要接口，`BeanPostProcessor`,`InstantiationAwareBeanPostProcessor`,`SmartInstantiationAwareBeanPostProcessor`,`DestructionAwareBeanPostProcessor`和`MergedBeanDefinitionPostProcessor`
他们的执行时机是不一样的

* 从Ioc容器获取所有的BeanPostProcessor
    ```java
    String[] postProcessorNames = beanFactory.getBeanNamesForType(BeanPostProcessor.class, true, false);
    ```
* 注册了一个checker, `BeanPostProcessorChecker`
    ```java
    int beanProcessorTargetCount = beanFactory.getBeanPostProcessorCount() + 1 + postProcessorNames.length;
    beanFactory.addBeanPostProcessor(new BeanPostProcessorChecker(beanFactory, beanProcessorTargetCount));
    ```
* 把所有的BeanPostProcessor按照实现`PriorityOrdered`,`Ordered`和没实现，这3类分类。
    ```java
    List<BeanPostProcessor> priorityOrderedPostProcessors = new ArrayList<>(); //放入实现PriorityOrdered的BeanPostProcessor
    List<BeanPostProcessor> internalPostProcessors = new ArrayList<>();//放入MergedBeanDefinitionPostProcessor
    List<String> orderedPostProcessorNames = new ArrayList<>();//放入实现Ordered的beanPostProcessor
    List<String> nonOrderedPostProcessorNames = new ArrayList<>();//放入没有实现任何排序接口的BeanPostProcessor
    ```
  `MergedBeanDefinitionPostProcessor`是一个`BeanPostProcessor`也是`MergedBeanDefinitionPostProcessor`，所以它能被注册2次。
    ```java
    // First, register the BeanPostProcessors that implement PriorityOrdered.
    sortPostProcessors(priorityOrderedPostProcessors, beanFactory);
    registerBeanPostProcessors(beanFactory, priorityOrderedPostProcessors);
    // Next, register the BeanPostProcessors that implement Ordered.
    ...
    // Now, register all regular BeanPostProcessors.
    ...
    // Finally, re-register all internal BeanPostProcessors.
    ...
    ```
* 注册一个listener, `ApplicationListenerDetector`
    ```java
    beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(applicationContext));
    ```
  
## 八、initMessageSource();
>Initialize message source for this context.

初始化MessageSource，在SpringMVC中就是来做国际化用的，消息绑定，消息解析

## 九、initApplicationEventMulticaster();
>Initialize event multicaster for this context.

* 如果容器中没有多播器，就创建一个放入容器
    ```java
    this.applicationEventMulticaster = new SimpleApplicationEventMulticaster(beanFactory);
    beanFactory.registerSingleton(APPLICATION_EVENT_MULTICASTER_BEAN_NAME, this.applicationEventMulticaster);
    ```
  多播器是一个`SimpleApplicationEventMulticaster`继承自`AbstractApplicationEventMulticaster`初始化的时候传入一个BeanFactory.

## 十、onRefresh();
>Initialize other special beans in specific context subclasses.

留给子类做扩展

## 十一、registerListeners();
>Check for listener beans and register them.

* 向多播器注册AbstractApplicationContext在初始化过程中保存的ApplicationListener.
  我们SpringBoot启动分析是有一步是prepareContext->applyInitializers(context);
  在这一步我们会用initializer初始化我们`AnnotationConfigServletWebServerApplicationContext`,这个初始化包括添加Listener通过`AbstractApplicationContex`提供的`addApplicationListener`方法
  ```java
  for (ApplicationListener<?> listener : getApplicationListeners()) {
    getApplicationEventMulticaster().addApplicationListener(listener);
  }
  ```
* 向多播器注册`ApplicationListener`，并且我们注册的是一个没有实例话的Listener，注释写的很明白，这样做的目的是，不初始化他们让post processor有机会去增强他们
  ```java
    // Do not initialize FactoryBeans here: We need to leave all regular beans
    // uninitialized to let post-processors apply to them!
    String[] listenerBeanNames = getBeanNamesForType(ApplicationListener.class, true, false);
    for (String listenerBeanName : listenerBeanNames) {
        getApplicationEventMulticaster().addApplicationListenerBean(listenerBeanName);
    }
  ```
* 发布早期事件
   ```java
    // Publish early application events now that we finally have a multicaster...
    Set<ApplicationEvent> earlyEventsToProcess = this.earlyApplicationEvents;
    this.earlyApplicationEvents = null;
    if (!CollectionUtils.isEmpty(earlyEventsToProcess)) {
        for (ApplicationEvent earlyEvent : earlyEventsToProcess) {
            getApplicationEventMulticaster().multicastEvent(earlyEvent);
        }
    }   
  ```

## 十二、finishBeanFactoryInitialization(beanFactory);
>Instantiate all remaining (non-lazy-init) singletons.
### 12-1 实例化所有bean的流程
* 用beanFactory实例化所有的注册的bean    
  beanFactory.preInstantiateSingletons();
* 读取Bean定义信息依次初始化
* 初始化的bean必须是单实例，并且不能是抽象的，不能是懒加载的
* 如果是FactoryBean，用FactoryBean的方式取实例化bean
* 普通bean，直接getBean，有则返回，没有则创建，放入Ioc容器。     
  bean具体如何创建在之后介绍
* 所有bean都加载好了之后，如果bean是`SmartInitializingSingleton`执行以下操作
```java
for (String beanName : beanNames) {
    Object singletonInstance = getSingleton(beanName);
    if (singletonInstance instanceof SmartInitializingSingleton) {
        SmartInitializingSingleton smartSingleton = (SmartInitializingSingleton) singletonInstance;
        smartSingleton.afterSingletonsInstantiated();
    }
}
```

### 12-2 Spring Ioc容器getBean详解
* getBean委托doGetBean
```java
public Object getBean(String name) throws BeansException {
    return doGetBean(name, null, null, false);
}
```
*  doGetBean, 先判断容器有没有这个bean，有则直接返回，没有开始创建，这里有Spring中Bean创建的三级缓存概念。   
   三级缓存用来解决循环依赖
```java
// Cache of singleton objects: bean name to bean instance. 
private final Map<String, Object> singletonObjects = new ConcurrentHashMap<>(256);

// Cache of singleton factories: bean name to ObjectFactory. 
private final Map<String, ObjectFactory<?>> singletonFactories = new HashMap<>(16);

// Cache of early singleton objects: bean name to bean instance. 
private final Map<String, Object> earlySingletonObjects = new ConcurrentHashMap<>(16);

...

Object sharedInstance = getSingleton(beanName);
```
* 判断有没有父容器（SpringMVC），有的话，尝试先从父容器拿这个bean（也是getBean)
* 得到bean的定义信息
```java
RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
checkMergedBeanDefinition(mbd, beanName, args);
```
* 根据bean的定义信息，判断当前bean是否有dependsOn，有的化，用getBean先得到（创建）dependsOn的bean
* 最后，走到create bean instance，会根据同的scope来创建不同的bean
```java
if (mbd.isSingleton()) 
...
else if (mbd.isPrototype()) 
...
else
```
* 委托给createBean来创建单实例bean，这里getSingleton第二个参数是一个ObjectFactory接口,有一个抽象方法`T getObject() throws BeansException;`
  这里传入匿名类，最后还是通过createBean来真正的完成bean的**实例化，初始化**
```java
if (mbd.isSingleton()) {
    sharedInstance = getSingleton(beanName, () -> {
        try {
            return createBean(beanName, mbd, args);
        }
        catch (BeansException ex) {
            // Explicitly remove instance from singleton cache: It might have been put there
            // eagerly by the creation process, to allow for circular reference resolution.
            // Also remove any beans that received a temporary reference to the bean.
            destroySingleton(beanName);
            throw ex;
        }
    });
    bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
}
``` 

* 在getSingleton里，我们看到，创建完成的bean会被放入`singletonObjects`
```java
singletonObject = singletonFactory.getObject(); //singleFactory就是传入的匿名类
newSingleton = true;
...
addSingleton(beanName, singletonObject); //加入singletonObjects 缓存中
```

### 12-2 Spring Ioc容器createBean详解
首先createBean 是在子类`AbstractAutowireCapableBeanFactory`中实现的。   
以下是具体流程
* Prepare method overrides. 这个特性在spring中已经不常用
    ```java
    mbdToUse.prepareMethodOverrides();
    ```
* 执行第一个BeanPostProcessor：`InstantiationAwareBeanPostProcessor`,这里如果是aop会允许我们先创建一个代理对象.   
  执行逻辑很清晰，判断时候为`InstantiationAwareBeanPostProcessor`，然后先执行前置方法`applyBeanPostProcessorsBeforeInstantiation`，之后在执行后置方法`applyBeanPostProcessorsAfterInitialization`
```java
// Give BeanPostProcessors a chance to return a proxy instead of the target bean instance.
Object bean = resolveBeforeInstantiation(beanName, mbdToUse);
...
protected Object resolveBeforeInstantiation(String beanName, RootBeanDefinition mbd) {
    Object bean = null;
    if (!Boolean.FALSE.equals(mbd.beforeInstantiationResolved)) { 
        // Make sure bean class is actually resolved at this point.
        if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {//是否有
            Class<?> targetType = determineTargetType(beanName, mbd);
            if (targetType != null) {
                bean = applyBeanPostProcessorsBeforeInstantiation(targetType, beanName);//执行前置方法
                if (bean != null) {
                    bean = applyBeanPostProcessorsAfterInitialization(bean, beanName);//执行后置方法
                }
            }
        }
        mbd.beforeInstantiationResolved = (bean != null);
    }
    return bean;
}
```
* 执行doCreateBean去真正的创建bean
```java
Object beanInstance = doCreateBean(beanName, mbdToUse, args);
```
### 12-3 Spring Ioc容器doCreateBean详解
#### 12-3-1 实例化Bean
* 利用反射**实例化**Bean
```java
// Instantiate the bean.
BeanWrapper instanceWrapper = null;
if (mbd.isSingleton()) {
    instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
}
if (instanceWrapper == null) {
    instanceWrapper = createBeanInstance(beanName, mbd, args);//实例化bean
}
```
* 用MergedBeanDefinitionPostProcessor后置处理器去增强Bean
```java
applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);
...
protected void applyMergedBeanDefinitionPostProcessors(RootBeanDefinition mbd, Class<?> beanType, String beanName) {
    for (BeanPostProcessor bp : getBeanPostProcessors()) {
        if (bp instanceof MergedBeanDefinitionPostProcessor) {
            MergedBeanDefinitionPostProcessor bdp = (MergedBeanDefinitionPostProcessor) bp;
            bdp.postProcessMergedBeanDefinition(mbd, beanType, beanName);//调用
        }
    }
}
```

* 放入SingletonFactory缓存，解决循环依赖的
```java
addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
```
#### 12-3-1 初始化Bean
* 给bean的属性赋值populateBean(beanName, mbd, instanceWrapper);   
  + 拿到后置处理器`InstantiationAwareBeanPostProcessor`，然后调用`postProcessAfterInstantiation`来增强bean
    >postProcessAfterInstantiation: This is the ideal callback for performing custom field injection on the given bean instance, right before Spring's autowiring kicks in
      ```java
        // Give any InstantiationAwareBeanPostProcessors the opportunity to modify the
        // state of the bean before properties are set. This can be used, for example,
        // to support styles of field injection.
        if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
            for (BeanPostProcessor bp : getBeanPostProcessors()) {
                if (bp instanceof InstantiationAwareBeanPostProcessor) {
                    InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
                    if (!ibp.postProcessAfterInstantiation(bw.getWrappedInstance(), beanName)) {
                        return;
                    }
                }
            }
        }
      ```
  + 拿到后置处理器`InstantiationAwareBeanPostProcessor`，然后调用`postProcessProperties`来增强bean
    > postProcessProperties: Post-process the given property values before the factory applies them to the given bean, without any need for property descriptors.
    ```java
    InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
    PropertyValues pvsToUse = ibp.postProcessProperties(pvs, bw.getWrappedInstance(), beanName);
    ```
  + 最后在把Property赋值给Bean
    ```applyPropertyValues(beanName, mbd, bw, pvs);```

* 初始化bean exposedObject = initializeBean(beanName, exposedObject, mbd);
  + invokeAwareMethods(beanName, bean); 去set各种Aware，这里可以看到自定义bean可以实现那些Aware接口
      ```java
      if (bean instanceof Aware) {
            if (bean instanceof BeanNameAware) {
                ((BeanNameAware) bean).setBeanName(beanName);
            }
            if (bean instanceof BeanClassLoaderAware) {
                ClassLoader bcl = getBeanClassLoader();
                if (bcl != null) {
                    ((BeanClassLoaderAware) bean).setBeanClassLoader(bcl);
                }
            }
            if (bean instanceof BeanFactoryAware) {
                ((BeanFactoryAware) bean).setBeanFactory(AbstractAutowireCapableBeanFactory.this);
            }
      }
      ```
  + wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName); 执行BeanPostProcessor的前置处理器
      ```java
      @Override
      public Object applyBeanPostProcessorsBeforeInitialization(Object existingBean, String beanName)
        			throws BeansException {
        
        Object result = existingBean;
            for (BeanPostProcessor processor : getBeanPostProcessors()) {
                Object current = processor.postProcessBeforeInitialization(result, beanName);
                if (current == null) {
                    return result;
                }
                result = current;
            }
            return result;
        }
      ```
  + invokeInitMethods(beanName, wrappedBean, mbd); 调用 init-method 来初始化Bean。
  + wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName); 执行BeanPostProcessor的后置处理器
    ```java
    @Override
    public Object applyBeanPostProcessorsAfterInitialization(Object existingBean, String beanName)
    			throws BeansException {
        Object result = existingBean;
        for (BeanPostProcessor processor : getBeanPostProcessors()) {
            Object current = processor.postProcessAfterInitialization(result, beanName);
            if (current == null) {
                return result;
            }
            result = current;
        }
        return result;
    }
    ```

## 十三、finishRefresh();
>Last step: publish corresponding event.
* initLifecycleProcessor();初始化和生命周期有关的后置处理器，`LifecycleProcessor`   
  默认从容器中找一个，如果没有创建一个，`DefaultLifecycleProcessor`并加入容器。
* getLifecycleProcessor().onRefresh();
  拿到前面定义的生命周期处理器，回掉onRefresh();
* 用多播器器发布事件
```java
// Publish the final event.
publishEvent(new ContextRefreshedEvent(this));
```

## 十四、总结
* Spring容器在启动的时候，会先保存所有注册的bean的定义信息
  + xml注册的
  + annotation注册进来的
* Spring容器在合适的时机创建这些Bean
  + 通过getBean创建，创建以后保存到Ioc容易。
  + 统一创建剩下来的bean，finishBeanFactoryInitialization()
* 后置处理器BeanPostProcessor
  + 每一个bean创建完成，都会使用各种后置处理器进行处理，来增强bean功能   
    AutowiredAnnotationBeanPostProcessor: 处理自动注入
    AnnotationAwareAspectJAutoProxyCreator: 来做APO功能
    ...
* 事件驱动模型
  ApplicationListener：事件监听
  ApplicationEventMulticaster： 事件派发
