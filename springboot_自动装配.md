# Spring Boot 自动装配

## 一、@SpringBootApplication注解
@SpringBootApplication 是一个复合注解。他由以下几个注解组成
* @SpringBootConfiguration
    + @Configuration
* @EnableAutoConfiguration
    + @AutoConfigurationPackage
        * @Import({Registrar.class})
    + @Import({AutoConfigurationImportSelector.class})
* @ComponentScan

`@SpringBootConfiguration`的底层注解就是一个`@Configuration`,表示当前类是一个Java配置类。   
`@ComponentScan`扫描注解。   
`@EnableAutoConfiguration`是自动装配的关键。他下面有2个关键的`@Import`注解.

## 二、@Import注解
`@Import`注解可以有3种用法。
* `@Import`注解里包含一个普通的JavaBean.class,他会把这个普通的JavaBean直接注册给BeanFactory
* `@Import`注解里包含了一个实现了`ImportSelector`接口的类，`@Import`注解把方法`String[] selectImports(AnnotationMetadata importingClassMetadata);` 
    返回的类信息都注册给BeanFactory
* `@Import`注解里包含了一个实现了`ImportBeanDefinitionRegistrar`接口的类，`@Import`注解会按照自定义的方式
    把类注册给BeanFactory. `void registerBeanDefinitions(AnnotationMetadata metadata, BeanDefinitionRegistry registry)`

## 三、@Import注解在SpringBoot中的应用。
### @Import({AutoConfigurationImportSelector.class})
AutoConfigurationImportSelector类声明
```java
AutoConfigurationImportSelector.class
public class AutoConfigurationImportSelector implements DeferredImportSelector, BeanClassLoaderAware,
		ResourceLoaderAware, BeanFactoryAware, EnvironmentAware, Ordered
```
`getAutoConfigurationEntry` 返回自动配置的entry
```java
AutoConfigurationImportSelector.class
@Override
public String[] selectImports(AnnotationMetadata annotationMetadata) {
    if (!isEnabled(annotationMetadata)) {
        return NO_IMPORTS;
    }
    AutoConfigurationEntry autoConfigurationEntry = getAutoConfigurationEntry(annotationMetadata);
    return StringUtils.toStringArray(autoConfigurationEntry.getConfigurations());
}
```
`getCandidateConfigurations` 返回所有的configurations
```java
AutoConfigurationImportSelector.class
protected AutoConfigurationEntry getAutoConfigurationEntry(AnnotationMetadata annotationMetadata) {
    if (!isEnabled(annotationMetadata)) {
        return EMPTY_ENTRY;
    }
    AnnotationAttributes attributes = getAttributes(annotationMetadata);
    List<String> configurations = getCandidateConfigurations(annotationMetadata, attributes);
    configurations = removeDuplicates(configurations);
    Set<String> exclusions = getExclusions(annotationMetadata, attributes);
    checkExcludedClasses(configurations, exclusions);
    configurations.removeAll(exclusions);
    configurations = getConfigurationClassFilter().filter(configurations);
    fireAutoConfigurationImportEvents(configurations, exclusions);
    return new AutoConfigurationEntry(configurations, exclusions);
}
```
这里注意：`loadFactoryNames`只load`EnableAutoConfiguration.class`这个类型
```java
AutoConfigurationImportSelector.class
protected List<String> getCandidateConfigurations(AnnotationMetadata metadata, AnnotationAttributes attributes) {
    List<String> configurations = SpringFactoriesLoader.loadFactoryNames(getSpringFactoriesLoaderFactoryClass(),
            getBeanClassLoader());
    Assert.notEmpty(configurations, "No auto configuration classes found in META-INF/spring.factories. If you "
            + "are using a custom packaging, make sure that file is correct.");
    return configurations;
}
protected Class<?> getSpringFactoriesLoaderFactoryClass() {
		return EnableAutoConfiguration.class;
}

```
`FACTORIES_RESOURCE_LOCATION=META-INF/spring.factories`
最终把spring.factories里定义的所有键值对返回，就是所有的AutoConfiguration
```java
SpringFactoriesLoader.class 
public static List<String> loadFactoryNames(Class<?> factoryType, @Nullable ClassLoader classLoader) {
    String factoryTypeName = factoryType.getName(); // 这里取到的名字就是：org.springframework.boot.autoconfigure.EnableAutoConfiguration
    return loadSpringFactories(classLoader).getOrDefault(factoryTypeName, Collections.emptyList());
}

private static Map<String, List<String>> loadSpringFactories(@Nullable ClassLoader classLoader) {
    MultiValueMap<String, String> result = cache.get(classLoader);
    if (result != null) {
        return result;
    }

    try {
        Enumeration<URL> urls = (classLoader != null ?
                classLoader.getResources(FACTORIES_RESOURCE_LOCATION) :
                ClassLoader.getSystemResources(FACTORIES_RESOURCE_LOCATION));
        result = new LinkedMultiValueMap<>();
        while (urls.hasMoreElements()) {
            URL url = urls.nextElement();
            UrlResource resource = new UrlResource(url);
            Properties properties = PropertiesLoaderUtils.loadProperties(resource);
            for (Map.Entry<?, ?> entry : properties.entrySet()) {
                String factoryTypeName = ((String) entry.getKey()).trim();
                for (String factoryImplementationName : StringUtils.commaDelimitedListToStringArray((String) entry.getValue())) {
                    result.add(factoryTypeName, factoryImplementationName.trim());
                }
            }
        }
        cache.put(classLoader, result);
        return result;
    }
    catch (IOException ex) {
        throw new IllegalArgumentException("Unable to load factories from location [" +
                FACTORIES_RESOURCE_LOCATION + "]", ex);
    }
}
```

### @Import({Registrar.class})
```java
AutoConfigurationPackages.class 
static class Registrar implements ImportBeanDefinitionRegistrar, DeterminableImports {

    @Override
    public void registerBeanDefinitions(AnnotationMetadata metadata, BeanDefinitionRegistry registry) {
        register(registry, new PackageImports(metadata).getPackageNames().toArray(new String[0]));
    }

    @Override
    public Set<Object> determineImports(AnnotationMetadata metadata) {
        return Collections.singleton(new PackageImports(metadata));
    }

}
```
```java
AutoConfigurationPackages.class 
public static void register(BeanDefinitionRegistry registry, String... packageNames) {
    if (registry.containsBeanDefinition(BEAN)) {
        BeanDefinition beanDefinition = registry.getBeanDefinition(BEAN);
        ConstructorArgumentValues constructorArguments = beanDefinition.getConstructorArgumentValues();
        constructorArguments.addIndexedArgumentValue(0, addBasePackages(constructorArguments, packageNames));
    }
    else {
        GenericBeanDefinition beanDefinition = new GenericBeanDefinition();
        beanDefinition.setBeanClass(BasePackages.class);
        beanDefinition.getConstructorArgumentValues().addIndexedArgumentValue(0, packageNames);
        beanDefinition.setRole(BeanDefinition.ROLE_INFRASTRUCTURE);
        registry.registerBeanDefinition(BEAN, beanDefinition);
    }
}
```
AutoConfigurationPackages是一个抽象类，里面定义了很多static的class用来注册bean
**注意**：这里实际上是像BeanFactory里注册了一个`BasePackages.class`，BasePackages是AutoConfigurationPackages中的一个静态类。   
这里packageName就是@SpringBootApplication所在包的名字。**这里也解释了只有这在main方法的子包才能被SpringBoot加载的原因**

## 四、自动装配的一个例子 -- RedisAutoConfiguration
```java
@Configuration(proxyBeanMethods = false)
@ConditionalOnClass(RedisOperations.class)
@EnableConfigurationProperties(RedisProperties.class)
@Import({ LettuceConnectionConfiguration.class, JedisConnectionConfiguration.class })
public class RedisAutoConfiguration {

	@Bean
	@ConditionalOnMissingBean(name = "redisTemplate")
	public RedisTemplate<Object, Object> redisTemplate(RedisConnectionFactory redisConnectionFactory)
			throws UnknownHostException {
		RedisTemplate<Object, Object> template = new RedisTemplate<>();
		template.setConnectionFactory(redisConnectionFactory);
		return template;
	}

	@Bean
	@ConditionalOnMissingBean
	public StringRedisTemplate stringRedisTemplate(RedisConnectionFactory redisConnectionFactory)
			throws UnknownHostException {
		StringRedisTemplate template = new StringRedisTemplate();
		template.setConnectionFactory(redisConnectionFactory);
		return template;
	}

}
```
* @Configuration(proxyBeanMethods = false)    
  这是一个配置类，proxyBeanMethods = false 代表配置类中的@Bean注解里的method调用了同类中的@Bean的method，不会向SpringIoc 容器里找，而是直接创建。
* @ConditionalOnClass(RedisOperations.class)
  只有class path中有RedisOperations.class 配置类才生效。
* @EnableConfigurationProperties(RedisProperties.class)
  使RedisProperties.class配置生效，**注意**：SpringBoot中的每一个自动配置都会对应这么一个Properties.class,这个跟我们在application.properties里的配置项对应
* @Import({ LettuceConnectionConfiguration.class, JedisConnectionConfiguration.class })
  SpringBoot导入LettuceConnectionConfiguration，JedisConnectionConfiguration.
* @ConditionalOnMissingBean(name = "redisTemplate")
  只有当我们SpringIoc容器中没有redisTemplate才会给我们自动装配一个RedisTemplate
* @ConditionalOnMissingBean
  只有当我们SpringIoc容器中没有StringRedisTemplate才会给我们自动装配一个StringRedisTemplate

>所以SpringBoot会按照上述规则给我们自动装配Redis

## 五、Spring Boot Auto Configuration原理 
### ConfigurationClassPostProcessor 类分析
SpringBoot实际用`ConfigurationClassPostProcessor`这个`BeanFactoryPostProcessor`来注册自动装配的类信息.
`ConfigurationClassPostProcessor`类声明,它实现了`BeanDefinitionRegistryPostProcessor`接口，`BeanDefinitionRegistryPostProcessor`接口继承自`BeanFactoryPostProcessor`接口
```java
public class ConfigurationClassPostProcessor implements BeanDefinitionRegistryPostProcessor,
		PriorityOrdered, ResourceLoaderAware, BeanClassLoaderAware, EnvironmentAware


public interface BeanDefinitionRegistryPostProcessor extends BeanFactoryPostProcessor {
	void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) throws BeansException;
}

public interface BeanFactoryPostProcessor {
	void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException;
}
```
在`ConfigurationClassPostProcessor`中，要关注`void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry)` 和`void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory)`

### ConfigurationClassPostProcessor何时注册到Ioc容器
在SpringBoot启动流程分析中我们知道，SpringBoot最终的Ioc容器是一个`AnnotationConfigServletWebServerApplicationContext`,通过反射创建出来。
```java
构造方法
AnnotationConfigServletWebServerApplicationContext.class 
public AnnotationConfigServletWebServerApplicationContext() {
    this.reader = new AnnotatedBeanDefinitionReader(this);
    this.scanner = new ClassPathBeanDefinitionScanner(this);
}
```
在这个构造方法里，初始化了一个`AnnotatedBeanDefinitionReader`, 这个构造方法最后会给Ioc容器注册一个AnnotationConfigProcessors，也就是`ConfigurationClassPostProcessor`   
在`AnnotationConfigUtils`中注册了`ConfigurationClassPostProcessor`
```java
AnnotatedBeanDefinitionReader.class 
public AnnotatedBeanDefinitionReader(BeanDefinitionRegistry registry) {
    this(registry, getOrCreateEnvironment(registry));
}

public AnnotatedBeanDefinitionReader(BeanDefinitionRegistry registry, Environment environment) {
    Assert.notNull(registry, "BeanDefinitionRegistry must not be null");
    Assert.notNull(environment, "Environment must not be null");
    this.registry = registry;
    this.conditionEvaluator = new ConditionEvaluator(registry, environment, null);
    AnnotationConfigUtils.registerAnnotationConfigProcessors(this.registry); //重点
}

AnnotationConfigUtils.class 
if (!registry.containsBeanDefinition(CONFIGURATION_ANNOTATION_PROCESSOR_BEAN_NAME)) {
    RootBeanDefinition def = new RootBeanDefinition(ConfigurationClassPostProcessor.class);// post processor
    def.setSource(source);
    beanDefs.add(registerPostProcessor(registry, def, CONFIGURATION_ANNOTATION_PROCESSOR_BEAN_NAME));
}
```
### ConfigurationClassPostProcessor何时初始化
`ConfigurationClassPostProcessor`是一个`BeanFactoryPostProcessor` 所以它会在Ioc容器刷新的时候被调用。   
AbstractApplicationContext.refresh()->invokeBeanFactoryPostProcessors(beanFactory);
1. 首先委托`PostProcessorRegistrationDelegate`处理invokeBeanFactoryPostProcessors
```java
AbstractApplicationContext
protected void invokeBeanFactoryPostProcessors(ConfigurableListableBeanFactory beanFactory) {
		PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors(beanFactory, getBeanFactoryPostProcessors());
```
2. `ConfigurationClassPostProcessor`目前只是注册到BeanFacotry，**但是还没有初始化**。以下是初始化逻辑
```java
PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors

List<BeanDefinitionRegistryPostProcessor> currentRegistryProcessors = new ArrayList<>();
// First, invoke the BeanDefinitionRegistryPostProcessors that implement PriorityOrdered.
String[] postProcessorNames =
        beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);//匹配
for (String ppName : postProcessorNames) {
    if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) { //匹配
        currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));//getBean创建实例
        processedBeans.add(ppName);
    }
}
sortPostProcessors(currentRegistryProcessors, beanFactory);
registryProcessors.addAll(currentRegistryProcessors);
invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry); //调用
currentRegistryProcessors.clear();
```
`ConfigurationClassPostProcessor`是一个`BeanDefinitionRegistryPostProcessor`并且是一个`PriorityOrdered`
所匹配上述代码条件，通过getBean会完成在Ioc容器中的实例化。   
invokeBeanDefinitionRegistryPostProcessors 调用 postProcessBeanDefinitionRegistry。

### ConfigurationClassPostProcessor如何注册我们的自动装配
上面分析过后我们来到了ConfigurationClassPostProcessor的postProcessBeanDefinitionRegistry方法。   
* 获得Ioc容器中的所有带`@Configuration`注解的bean，作为候选bean 
    ```java
    List<BeanDefinitionHolder> configCandidates = new ArrayList<>();
    String[] candidateNames = registry.getBeanDefinitionNames();
    
    for (String beanName : candidateNames) {
        BeanDefinition beanDef = registry.getBeanDefinition(beanName);
        if (beanDef.getAttribute(ConfigurationClassUtils.CONFIGURATION_CLASS_ATTRIBUTE) != null) { //判断是否有@Configure注解
            if (logger.isDebugEnabled()) {
                logger.debug("Bean definition has already been processed as a configuration class: " + beanDef);
            }
        }
        else if (ConfigurationClassUtils.checkConfigurationClassCandidate(beanDef, this.metadataReaderFactory)) {
            configCandidates.add(new BeanDefinitionHolder(beanDef, beanName));
        }
    }
    
    // Return immediately if no @Configuration classes were found
    if (configCandidates.isEmpty()) {
        return;
    }
    ```
* 用`ConfigurationClassParser`去解析每一个候选的bean
    ```java
    ConfigurationClassParser parser = new ConfigurationClassParser(
                    this.metadataReaderFactory, this.problemReporter, this.environment,
                    this.resourceLoader, this.componentScanBeanNameGenerator, registry);
    Set<BeanDefinitionHolder> candidates = new LinkedHashSet<>(configCandidates);
    do {
                parser.parse(candidates);
        ...
    }while (!candidates.isEmpty());
    ```

* parser的doProcessConfigurationClass方法最终解析候选bean
   根据代码我们知道，parser依次解析`@PropertySource`注解，`@ComponentScan`注解，`@Import`注解和`@ImportResource`注解,   
   其中@Import注解是我们自动装配的关键，我们着重分析  
  + Process any @PropertySource annotations
    ```java
    ConfigurationClassParser
    // Process any @PropertySource annotations
    for (AnnotationAttributes propertySource : AnnotationConfigUtils.attributesForRepeatable(
            sourceClass.getMetadata(), PropertySources.class,
            org.springframework.context.annotation.PropertySource.class)) {
        if (this.environment instanceof ConfigurableEnvironment) {
            processPropertySource(propertySource);
        }
        else {
            logger.info("Ignoring @PropertySource annotation on [" + sourceClass.getMetadata().getClassName() +
                    "]. Reason: Environment must implement ConfigurableEnvironment");
        }
    }
    ```
  + Process any @ComponentScan annotations
      ```java
      // Process any @ComponentScan annotations
        Set<AnnotationAttributes> componentScans = AnnotationConfigUtils.attributesForRepeatable(
            sourceClass.getMetadata(), ComponentScans.class, ComponentScan.class);
        if (!componentScans.isEmpty() &&
            !this.conditionEvaluator.shouldSkip(sourceClass.getMetadata(), ConfigurationPhase.REGISTER_BEAN)) {
        for (AnnotationAttributes componentScan : componentScans) {
            // The config class is annotated with @ComponentScan -> perform the scan immediately
            Set<BeanDefinitionHolder> scannedBeanDefinitions =
                    this.componentScanParser.parse(componentScan, sourceClass.getMetadata().getClassName());
            // Check the set of scanned definitions for any further config classes and parse recursively if needed
            for (BeanDefinitionHolder holder : scannedBeanDefinitions) {
                BeanDefinition bdCand = holder.getBeanDefinition().getOriginatingBeanDefinition();
                if (bdCand == null) {
                    bdCand = holder.getBeanDefinition();
                }
                if (ConfigurationClassUtils.checkConfigurationClassCandidate(bdCand, this.metadataReaderFactory)) {
                    parse(bdCand.getBeanClassName(), holder.getBeanName());
                }
            }
        }
      ```
  +  Process any @Import annotations
      ```java
        // Process any @Import annotations
        processImports(configClass, sourceClass, getImports(sourceClass), filter, true);
      ```
  + Process any @ImportResource annotations   
      ```java
        // Process any @ImportResource annotations
        AnnotationAttributes importResource =
                AnnotationConfigUtils.attributesFor(sourceClass.getMetadata(), ImportResource.class);
        if (importResource != null) {
            String[] resources = importResource.getStringArray("locations");
            Class<? extends BeanDefinitionReader> readerClass = importResource.getClass("reader");
            for (String resource : resources) {
                String resolvedResource = this.environment.resolveRequiredPlaceholders(resource);
                configClass.addImportedResource(resolvedResource, readerClass);
            }
        } 
      ```
### @Import原理
* `if (candidate.isAssignable(ImportSelector.class))`
   处理import了ImportSelector实现类的情况
* `else if (candidate.isAssignable(ImportBeanDefinitionRegistrar.class))`
   处理ImportBeanDefinitionRegistrar实现类的情况
* `else ` 
   处理普通Bean
跟之前的阐述@Import的用法是一样的
```java
private void processImports(ConfigurationClass configClass, SourceClass currentSourceClass,
        Collection<SourceClass> importCandidates, Predicate<String> exclusionFilter,
        boolean checkForCircularImports) {

    if (importCandidates.isEmpty()) {
        return;
    }

    if (checkForCircularImports && isChainedImportOnStack(configClass)) {
        this.problemReporter.error(new CircularImportProblem(configClass, this.importStack));
    }
    else {
        this.importStack.push(configClass);
        try {
            for (SourceClass candidate : importCandidates) {
                if (candidate.isAssignable(ImportSelector.class)) {
                    // Candidate class is an ImportSelector -> delegate to it to determine imports
                    Class<?> candidateClass = candidate.loadClass();
                    ImportSelector selector = ParserStrategyUtils.instantiateClass(candidateClass, ImportSelector.class,
                            this.environment, this.resourceLoader, this.registry);
                    Predicate<String> selectorFilter = selector.getExclusionFilter();
                    if (selectorFilter != null) {
                        exclusionFilter = exclusionFilter.or(selectorFilter);
                    }
                    if (selector instanceof DeferredImportSelector) {
                        this.deferredImportSelectorHandler.handle(configClass, (DeferredImportSelector) selector);
                    }
                    else {
                        String[] importClassNames = selector.selectImports(currentSourceClass.getMetadata());
                        Collection<SourceClass> importSourceClasses = asSourceClasses(importClassNames, exclusionFilter);
                        processImports(configClass, currentSourceClass, importSourceClasses, exclusionFilter, false);
                    }
                }
                else if (candidate.isAssignable(ImportBeanDefinitionRegistrar.class)) {
                    // Candidate class is an ImportBeanDefinitionRegistrar ->
                    // delegate to it to register additional bean definitions
                    Class<?> candidateClass = candidate.loadClass();
                    ImportBeanDefinitionRegistrar registrar =
                            ParserStrategyUtils.instantiateClass(candidateClass, ImportBeanDefinitionRegistrar.class,
                                    this.environment, this.resourceLoader, this.registry);
                    configClass.addImportBeanDefinitionRegistrar(registrar, currentSourceClass.getMetadata());
                }
                else {
                    // Candidate class not an ImportSelector or ImportBeanDefinitionRegistrar ->
                    // process it as an @Configuration class
                    this.importStack.registerImport(
                            currentSourceClass.getMetadata(), candidate.getMetadata().getClassName());
                    processConfigurationClass(candidate.asConfigClass(configClass), exclusionFilter);
                }
            }
        }
        catch (BeanDefinitionStoreException ex) {
            throw ex;
        }
        catch (Throwable ex) {
            throw new BeanDefinitionStoreException(
                    "Failed to process import candidates for configuration class [" +
                    configClass.getMetadata().getClassName() + "]", ex);
        }
        finally {
            this.importStack.pop();
        }
    }
}
```

## 六、如何自定义Starter
