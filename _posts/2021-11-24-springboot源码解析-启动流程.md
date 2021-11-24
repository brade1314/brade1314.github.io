---
layout:       post
title:        "springboot源码解析之启动流程"
date:         2021-11-24 18:00:00
author:       "Brade"
header-style: text
header-mask:  0.3
catalog:      true
tags:
- springboot
- Java
- 源码
---

# springboot 源码解析
本文中 `springboot` 采用的版本为 `2.5.6`.

## 1.1 `@SpringBootApplication` 注解
> 标注此注解的类,是 `SpringBoot` 主配置类, `SpringBoot` 就应该运行这个类的main方法来启动SpringBoot应用.

> 注解定义如下:
```java
    @Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@SpringBootConfiguration
@EnableAutoConfiguration
@ComponentScan(
        excludeFilters = {@Filter(
                type = FilterType.CUSTOM,
                classes = {TypeExcludeFilter.class}
        ), @Filter(
                type = FilterType.CUSTOM,
                classes = {AutoConfigurationExcludeFilter.class}
        )}
)
public @interface SpringBootApplication {}
```

### 1.1.1 `@SpringBootConfiguration` 注解
> 表示这是一个Spring Boot的配置类, 就是`Configuration`配置类.

### 1.1.2 `@EnableAutoConfiguration` 注解
> 开启自动配置功能, 使自动配置生效.

> 包含 `@AutoConfigurationPackage` ,`@Import({AutoConfigurationImportSelector.class})` 注解

> `AutoConfigurationPackage` : 重点是`@Import({Registrar.class})` ,默认将主配置类(`@SpringBootApplication`)所在的包及其子包里面的所有组件扫描到`Spring`容器中,
> 所以通常将 `@SpringBootApplication` 注解过的类放到最顶层.
```java
    static class Registrar implements ImportBeanDefinitionRegistrar, DeterminableImports {
        Registrar() {
        }

        public void registerBeanDefinitions(AnnotationMetadata metadata, BeanDefinitionRegistry registry) {
            AutoConfigurationPackages.register(registry, (String[])(new AutoConfigurationPackages.PackageImports(metadata)).getPackageNames().toArray(new String[0]));
        }

        public Set<Object> determineImports(AnnotationMetadata metadata) {
            return Collections.singleton(new AutoConfigurationPackages.PackageImports(metadata));
        }
    }
```

> `@Import({AutoConfigurationImportSelector.class})` : 导入组件的选择器, `#AutoConfigurationEntry` -> `getCandidateConfigurations`,
> 加载 `spring-boot-autoconfigure`包中 `META-INF/spring.factories` 文件中默认的自动配置类.
```java
    protected AutoConfigurationImportSelector.AutoConfigurationEntry getAutoConfigurationEntry(AnnotationMetadata annotationMetadata) {
        if (!this.isEnabled(annotationMetadata)) {
            return EMPTY_ENTRY;
        } else {
            AnnotationAttributes attributes = this.getAttributes(annotationMetadata);
            List<String> configurations = this.getCandidateConfigurations(annotationMetadata, attributes);
            configurations = this.removeDuplicates(configurations);
            Set<String> exclusions = this.getExclusions(annotationMetadata, attributes);
            this.checkExcludedClasses(configurations, exclusions);
            configurations.removeAll(exclusions);
            configurations = this.getConfigurationClassFilter().filter(configurations);
            this.fireAutoConfigurationImportEvents(configurations, exclusions);
            return new AutoConfigurationImportSelector.AutoConfigurationEntry(configurations, exclusions);
        }
    }
```

```java
    protected List<String> getCandidateConfigurations(AnnotationMetadata metadata, AnnotationAttributes attributes) {
        List<String> configurations = SpringFactoriesLoader.loadFactoryNames(this.getSpringFactoriesLoaderFactoryClass(), this.getBeanClassLoader());
        Assert.notEmpty(configurations, "No auto configuration classes found in META-INF/spring.factories. If you are using a custom packaging, make sure that file is correct.");
        return configurations;
    }
```

## 1.2 `SpringApplication#run(Class<?> primarySource, String... args)`
> 先创建 `SpringApplication` 实例,再调用对应的`run`方法
```java
    public static ConfigurableApplicationContext run(Class<?>[] primarySources, String[] args) {
        return (new SpringApplication(primarySources)).run(args);
    }
```
### 1.2.1 构造器
```java
    public SpringApplication(ResourceLoader resourceLoader, Class<?>... primarySources) {
        this.sources = new LinkedHashSet();
        this.bannerMode = Mode.CONSOLE;
        this.logStartupInfo = true;
        this.addCommandLineProperties = true;
        this.addConversionService = true;
        this.headless = true;
        this.registerShutdownHook = true;
        this.additionalProfiles = Collections.emptySet();
        this.isCustomEnvironment = false;
        this.lazyInitialization = false;
        this.applicationContextFactory = ApplicationContextFactory.DEFAULT;
        this.applicationStartup = ApplicationStartup.DEFAULT;
        this.resourceLoader = resourceLoader;
        Assert.notNull(primarySources, "PrimarySources must not be null");
        // 应用入口类
        this.primarySources = new LinkedHashSet(Arrays.asList(primarySources));
        // 应用类型  NONE,SERVLET(web),REACTIVE(采用netty)
        this.webApplicationType = WebApplicationType.deduceFromClasspath();
        // 引导初始化: 获取对应 Bootstrapper + BootstrapRegistryInitializer 类的配置
        this.bootstrapRegistryInitializers = this.getBootstrapRegistryInitializersFromSpringFactories();
        // 应用上下文初始化: 获取spring-boot-autoconfigure 中 META-INF/spring.factories 中 ApplicationContextInitializer
        this.setInitializers(this.getSpringFactoriesInstances(ApplicationContextInitializer.class));
        // 监听:获取spring-boot中META-INF/spring.factories 中 ApplicationListener
        this.setListeners(this.getSpringFactoriesInstances(ApplicationListener.class));
        // 应用程序入口主类
        this.mainApplicationClass = this.deduceMainApplicationClass();
    }
```

+ 1.2.1.1 `WebApplicationType#deduceFromClasspath` 获取应用类型
```java
    static WebApplicationType deduceFromClasspath() {
        if (ClassUtils.isPresent("org.springframework.web.reactive.DispatcherHandler", (ClassLoader)null) && !ClassUtils.isPresent("org.springframework.web.servlet.DispatcherServlet", (ClassLoader)null) && !ClassUtils.isPresent("org.glassfish.jersey.servlet.ServletContainer", (ClassLoader)null)) {
            return REACTIVE;
        } else {
            String[] var0 = SERVLET_INDICATOR_CLASSES;
            int var1 = var0.length;

            for(int var2 = 0; var2 < var1; ++var2) {
                String className = var0[var2];
                if (!ClassUtils.isPresent(className, (ClassLoader)null)) {
                    return NONE;
                }
            }

            return SERVLET;
        }
    }
```

+ 1.2.1.2 `SpringApplication#getBootstrapRegistryInitializersFromSpringFactories` 引导初始化
```java
    private List<BootstrapRegistryInitializer> getBootstrapRegistryInitializersFromSpringFactories() {
        ArrayList<BootstrapRegistryInitializer> initializers = new ArrayList();
        this.getSpringFactoriesInstances(Bootstrapper.class).stream().map((bootstrapper) -> {
            return bootstrapper::initialize;
        }).forEach(initializers::add);
        initializers.addAll(this.getSpringFactoriesInstances(BootstrapRegistryInitializer.class));
        return initializers;
    }
```
```java
    private <T> Collection<T> getSpringFactoriesInstances(Class<T> type) {
        return this.getSpringFactoriesInstances(type, new Class[0]);
    }

    private <T> Collection<T> getSpringFactoriesInstances(Class<T> type, Class<?>[] parameterTypes, Object... args) {
        ClassLoader classLoader = this.getClassLoader();
        Set<String> names = new LinkedHashSet(SpringFactoriesLoader.loadFactoryNames(type, classLoader));
        List<T> instances = this.createSpringFactoriesInstances(type, parameterTypes, classLoader, args, names);
        AnnotationAwareOrderComparator.sort(instances);
        return instances;
    }

    private <T> List<T> createSpringFactoriesInstances(Class<T> type, Class<?>[] parameterTypes, ClassLoader classLoader, Object[] args, Set<String> names) {
        List<T> instances = new ArrayList(names.size());
        Iterator var7 = names.iterator();

        while(var7.hasNext()) {
            String name = (String)var7.next();

            try {
                Class<?> instanceClass = ClassUtils.forName(name, classLoader);
                Assert.isAssignable(type, instanceClass);
                Constructor<?> constructor = instanceClass.getDeclaredConstructor(parameterTypes);
                T instance = BeanUtils.instantiateClass(constructor, args);
                instances.add(instance);
            } catch (Throwable var12) {
                throw new IllegalArgumentException("Cannot instantiate " + type + " : " + name, var12);
            }
        }

        return instances;
    }
```

> `SpringFactoriesLoader#loadFactoryNames`,解析 `spring.factories`文件,获取对应配置类
```java
    public static List<String> loadFactoryNames(Class<?> factoryType, @Nullable ClassLoader classLoader) {
        ClassLoader classLoaderToUse = classLoader;
        if (classLoader == null) {
            classLoaderToUse = SpringFactoriesLoader.class.getClassLoader();
        }

        String factoryTypeName = factoryType.getName();
        return (List)loadSpringFactories(classLoaderToUse).getOrDefault(factoryTypeName, Collections.emptyList());
    }

    private static Map<String, List<String>> loadSpringFactories(ClassLoader classLoader) {
        Map<String, List<String>> result = (Map)cache.get(classLoader);
        if (result != null) {
            return result;
        } else {
            HashMap result = new HashMap();

            try {
                // 获取 spring.factories 配置文件中的 url
                Enumeration urls = classLoader.getResources("META-INF/spring.factories");

                while(urls.hasMoreElements()) {
                    URL url = (URL)urls.nextElement();
                    UrlResource resource = new UrlResource(url);
                    Properties properties = PropertiesLoaderUtils.loadProperties(resource);
                    Iterator var6 = properties.entrySet().iterator();

                    while(var6.hasNext()) {
                        Entry<?, ?> entry = (Entry)var6.next();
                        String factoryTypeName = ((String)entry.getKey()).trim();
                        String[] factoryImplementationNames = StringUtils.commaDelimitedListToStringArray((String)entry.getValue());
                        String[] var10 = factoryImplementationNames;
                        int var11 = factoryImplementationNames.length;

                        for(int var12 = 0; var12 < var11; ++var12) {
                            String factoryImplementationName = var10[var12];
                            ((List)result.computeIfAbsent(factoryTypeName, (key) -> {
                                return new ArrayList();
                            })).add(factoryImplementationName.trim());
                        }
                    }
                }
                // 去重
                result.replaceAll((factoryType, implementations) -> {
                    return (List)implementations.stream().distinct().collect(Collectors.collectingAndThen(Collectors.toList(), Collections::unmodifiableList));
                });
                cache.put(classLoader, result);
                return result;
            } catch (IOException var14) {
                throw new IllegalArgumentException("Unable to load factories from location [META-INF/spring.factories]", var14);
            }
        }
    }
```

+ 1.2.1.3 `SpringApplication#deduceMainApplicationClass`, 应用程序入口主类
```java
    private Class<?> deduceMainApplicationClass() {
        try {
            StackTraceElement[] stackTrace = (new RuntimeException()).getStackTrace();
            StackTraceElement[] var2 = stackTrace;
            int var3 = stackTrace.length;

            for(int var4 = 0; var4 < var3; ++var4) {
                StackTraceElement stackTraceElement = var2[var4];
                // 如果方法名是 main 方法
                if ("main".equals(stackTraceElement.getMethodName())) {
                    return Class.forName(stackTraceElement.getClassName());
                }
            }
        } catch (ClassNotFoundException var6) {
        }

        return null;
    }
```

### 1.2.2 `SpringApplication#run` 方法
```java
    public ConfigurableApplicationContext run(String... args) {
        // 计时器
        StopWatch stopWatch = new StopWatch();
        stopWatch.start();
        // 引导上下文
        DefaultBootstrapContext bootstrapContext = this.createBootstrapContext();
        // 应用上下文
        ConfigurableApplicationContext context = null;
        // 无头属性:缺少显示设备、键盘或鼠标这些外设时
        this.configureHeadlessProperty();
        // 获取监听器:org.springframework.boot.context.event.EventPublishingRunListener
        SpringApplicationRunListeners listeners = this.getRunListeners(args);
        // 向监听器发通知 : starting
        listeners.starting(bootstrapContext, this.mainApplicationClass);

        try {
        // 参数
        ApplicationArguments applicationArguments = new DefaultApplicationArguments(args);
        // 准备环境:根据前面获取的监听构建
        ConfigurableEnvironment environment = this.prepareEnvironment(listeners, bootstrapContext, applicationArguments);
        // 配置忽略 Bean 信息
        this.configureIgnoreBeanInfo(environment);
        // 打印Spring图标
        Banner printedBanner = this.printBanner(environment);
        // 创建应用上下文
        context = this.createApplicationContext();
        // 应用启动
        context.setApplicationStartup(this.applicationStartup);
        // 准备上下文
        this.prepareContext(bootstrapContext, context, environment, listeners, applicationArguments, printedBanner);
        // 刷新上下文
        this.refreshContext(context);
        // 刷新后执行
        this.afterRefresh(context, applicationArguments);
        // 计时器结束
        stopWatch.stop();
        if (this.logStartupInfo) {
        (new StartupInfoLogger(this.mainApplicationClass)).logStarted(this.getApplicationLog(), stopWatch);
        }
        // 向监听器发通知 : started
        listeners.started(context);
        // 执行Runners
        this.callRunners(context, applicationArguments);
        } catch (Throwable var10) {
        this.handleRunFailure(context, var10, listeners);
        throw new IllegalStateException(var10);
        }

        try {
        // 向监听器发通知 : running
        listeners.running(context);
        return context;
        } catch (Throwable var9) {
        this.handleRunFailure(context, var9, (SpringApplicationRunListeners) null);
        throw new IllegalStateException(var9);
        }
        }
```

+ 1.2.2.1 `SpringApplication#createBootstrapContext`,引导上下文初始化
```java
    private DefaultBootstrapContext createBootstrapContext() {
        DefaultBootstrapContext bootstrapContext = new DefaultBootstrapContext();
        // 在构造器中已经从配置文件 spring.factories 获取
        this.bootstrapRegistryInitializers.forEach((initializer) -> {
        initializer.initialize(bootstrapContext);
        });
        return bootstrapContext;
        }
```

+ 1.2.2.2 `SpringApplication#configureHeadlessProperty`,无头属性设置
```java
    private void configureHeadlessProperty() {
        System.setProperty("java.awt.headless", System.getProperty("java.awt.headless", Boolean.toString(this.headless)));
        }
```

+ 1.2.2.3 `SpringApplication#getRunListeners`,获取监听器
```java
    private SpringApplicationRunListeners getRunListeners(String[] args) {
        Class<?>[] types = new Class[]{SpringApplication.class, String[].class};
        return new SpringApplicationRunListeners(logger, this.getSpringFactoriesInstances(SpringApplicationRunListener.class, types, this, args), this.applicationStartup);
        }
```

+ 1.2.2.4 `SpringApplication#prepareEnvironment`,准备环境
```java
    private ConfigurableEnvironment prepareEnvironment(SpringApplicationRunListeners listeners, DefaultBootstrapContext bootstrapContext, ApplicationArguments applicationArguments) {
        // 获取类型对应的 ConfigurableEnvironment
        ConfigurableEnvironment environment = this.getOrCreateEnvironment();
        // 配置环境
        this.configureEnvironment((ConfigurableEnvironment)environment, applicationArguments.getSourceArgs());
        ConfigurationPropertySources.attach((Environment)environment);
        listeners.environmentPrepared(bootstrapContext, (ConfigurableEnvironment)environment);
        DefaultPropertiesPropertySource.moveToEnd((ConfigurableEnvironment)environment);
        Assert.state(!((ConfigurableEnvironment)environment).containsProperty("spring.main.environment-prefix"), "Environment prefix cannot be set via properties.");
        this.bindToSpringApplication((ConfigurableEnvironment)environment);
        if (!this.isCustomEnvironment) {
        environment = (new EnvironmentConverter(this.getClassLoader())).convertEnvironmentIfNecessary((ConfigurableEnvironment)environment, this.deduceEnvironmentClass());
        }

        ConfigurationPropertySources.attach((Environment)environment);
        return (ConfigurableEnvironment)environment;
        }
```

> `SpringApplication#getOrCreateEnvironment` 获取环境
```java
    private ConfigurableEnvironment getOrCreateEnvironment() {
        if (this.environment != null) {
        return this.environment;
        } else {
        switch(this.webApplicationType) {
        case SERVLET:
        return new ApplicationServletEnvironment();
        case REACTIVE:
        return new ApplicationReactiveWebEnvironment();
default:
        return new ApplicationEnvironment();
        }
        }
        }
```

> `SpringApplication#configureEnvironment` 配置环境
```java
    protected void configureEnvironment(ConfigurableEnvironment environment, String[] args) {
        if (this.addConversionService) {
        environment.setConversionService(new ApplicationConversionService());
        }

        this.configurePropertySources(environment, args);
        this.configureProfiles(environment, args);
        }

protected void configurePropertySources(ConfigurableEnvironment environment, String[] args) {
        MutablePropertySources sources = environment.getPropertySources();
        if (!CollectionUtils.isEmpty(this.defaultProperties)) {
        DefaultPropertiesPropertySource.addOrMerge(this.defaultProperties, sources);
        }

        if (this.addCommandLineProperties && args.length > 0) {
        String name = "commandLineArgs";
        if (sources.contains(name)) {
        PropertySource<?> source = sources.get(name);
        CompositePropertySource composite = new CompositePropertySource(name);
        composite.addPropertySource(new SimpleCommandLinePropertySource("springApplicationCommandLineArgs", args));
        composite.addPropertySource(source);
        sources.replace(name, composite);
        } else {
        sources.addFirst(new SimpleCommandLinePropertySource(args));
        }
        }

        }

protected void configureProfiles(ConfigurableEnvironment environment, String[] args) {}
```

+ 1.2.2.5 `SpringApplication#configureIgnoreBeanInfo`, 配置忽略 Bean 信息
```java
    private void configureIgnoreBeanInfo(ConfigurableEnvironment environment) {
        // 如果为空,则设置默认值为 true
        if (System.getProperty("spring.beaninfo.ignore") == null) {
        Boolean ignore = (Boolean)environment.getProperty("spring.beaninfo.ignore", Boolean.class, Boolean.TRUE);
        System.setProperty("spring.beaninfo.ignore", ignore.toString());
        }

        }
```

+ 1.2.2.6 `SpringApplication#printBanner`, 打印 `Spring` 横幅
```java
    private Banner printBanner(ConfigurableEnvironment environment) {
        if (this.bannerMode == Mode.OFF) {
        return null;
        } else {
        ResourceLoader resourceLoader = this.resourceLoader != null ? this.resourceLoader : new DefaultResourceLoader((ClassLoader)null);
        SpringApplicationBannerPrinter bannerPrinter = new SpringApplicationBannerPrinter((ResourceLoader)resourceLoader, this.banner);
        return this.bannerMode == Mode.LOG ? bannerPrinter.print(environment, this.mainApplicationClass, logger) : bannerPrinter.print(environment, this.mainApplicationClass, System.out);
        }
        }
```

+ 1.2.2.7 `SpringApplication#createApplicationContext`, 根据应用类型创建应用上下文
```java
    // 构造器中初始化了上下文工厂, this.applicationContextFactory = ApplicationContextFactory.DEFAULT;
protected ConfigurableApplicationContext createApplicationContext() {
        return this.applicationContextFactory.create(this.webApplicationType);
        }
```

> 上下文工厂
```java
public interface ApplicationContextFactory {
    ApplicationContextFactory DEFAULT = (webApplicationType) -> {
        try {
            switch(webApplicationType) {
                case SERVLET:
                    return new AnnotationConfigServletWebServerApplicationContext();
                case REACTIVE:
                    return new AnnotationConfigReactiveWebServerApplicationContext();
                default:
                    return new AnnotationConfigApplicationContext();
            }
        } catch (Exception var2) {
            throw new IllegalStateException("Unable create a default ApplicationContext instance, you may need a custom ApplicationContextFactory", var2);
        }
    };

    ConfigurableApplicationContext create(WebApplicationType webApplicationType);

    static ApplicationContextFactory ofContextClass(Class<? extends ConfigurableApplicationContext> contextClass) {
        return of(() -> {
            return (ConfigurableApplicationContext)BeanUtils.instantiateClass(contextClass);
        });
    }

    static ApplicationContextFactory of(Supplier<ConfigurableApplicationContext> supplier) {
        return (webApplicationType) -> {
            return (ConfigurableApplicationContext)supplier.get();
        };
    }
}
```

+ 1.2.2.8 `SpringApplication#prepareContext`, 准备上下文
```java
    private void prepareContext(DefaultBootstrapContext bootstrapContext, ConfigurableApplicationContext context, ConfigurableEnvironment environment, SpringApplicationRunListeners listeners, ApplicationArguments applicationArguments, Banner printedBanner) {
        // 设置上下文环境
        context.setEnvironment(environment);
        // 上下文后置处理
        this.postProcessApplicationContext(context);
        // 调用初始化配置类
        this.applyInitializers(context);
        // 向监听器发通知: contextPrepared
        listeners.contextPrepared(context);
        // 引导上下文关闭
        bootstrapContext.close(context);
        if (this.logStartupInfo) {
        this.logStartupInfo(context.getParent() == null);
        this.logStartupProfileInfo(context);
        }
        // 将容器指定的参数封装成bean
        ConfigurableListableBeanFactory beanFactory = context.getBeanFactory();
        beanFactory.registerSingleton("springApplicationArguments", applicationArguments);
        if (printedBanner != null) {
        beanFactory.registerSingleton("springBootBanner", printedBanner);
        }

        if (beanFactory instanceof DefaultListableBeanFactory) {
        ((DefaultListableBeanFactory)beanFactory).setAllowBeanDefinitionOverriding(this.allowBeanDefinitionOverriding);
        }

        if (this.lazyInitialization) {
        context.addBeanFactoryPostProcessor(new LazyInitializationBeanFactoryPostProcessor());
        }
        // 获取启动类参数
        Set<Object> sources = this.getAllSources();
        Assert.notEmpty(sources, "Sources must not be empty");
        // 加载启动类
        this.load(context, sources.toArray(new Object[0]));
        // 向监听器发通知: contextLoaded
        listeners.contextLoaded(context);
        }
```

> `SpringApplication#postProcessApplicationContext`, 上下文后置处理
```java
    protected void postProcessApplicationContext(ConfigurableApplicationContext context) {
        if (this.beanNameGenerator != null) {
        context.getBeanFactory().registerSingleton("org.springframework.context.annotation.internalConfigurationBeanNameGenerator", this.beanNameGenerator);
        }

        if (this.resourceLoader != null) {
        if (context instanceof GenericApplicationContext) {
        ((GenericApplicationContext)context).setResourceLoader(this.resourceLoader);
        }

        if (context instanceof DefaultResourceLoader) {
        ((DefaultResourceLoader)context).setClassLoader(this.resourceLoader.getClassLoader());
        }
        }

        if (this.addConversionService) {
        context.getBeanFactory().setConversionService(context.getEnvironment().getConversionService());
        }

        }
```

> `SpringApplication#applyInitializers`, 调用初始化配置类
```java
    protected void applyInitializers(ConfigurableApplicationContext context) {
        // 在构造器中调用 setInitializers() 加载了
        Iterator var2 = this.getInitializers().iterator();
        // 遍历
        while(var2.hasNext()) {
        ApplicationContextInitializer initializer = (ApplicationContextInitializer)var2.next();
        Class<?> requiredType = GenericTypeResolver.resolveTypeArgument(initializer.getClass(), ApplicationContextInitializer.class);
        Assert.isInstanceOf(requiredType, context, "Unable to call initializer.");
        // 调用初始化方法
        initializer.initialize(context);
        }

        }
```

> `SpringApplication#getAllSources`, 获取启动类参数
```java
    public Set<Object> getAllSources() {
        Set<Object> allSources = new LinkedHashSet();
        // primarySources 即为启动类,构造器中初始化
        if (!CollectionUtils.isEmpty(this.primarySources)) {
        allSources.addAll(this.primarySources);
        }

        if (!CollectionUtils.isEmpty(this.sources)) {
        allSources.addAll(this.sources);
        }

        return Collections.unmodifiableSet(allSources);
        }
```

> `SpringApplication#load`, 加载启动类
```java
    protected void load(ApplicationContext context, Object[] sources) {
        if (logger.isDebugEnabled()) {
        logger.debug("Loading source " + StringUtils.arrayToCommaDelimitedString(sources));
        }

        BeanDefinitionLoader loader = this.createBeanDefinitionLoader(this.getBeanDefinitionRegistry(context), sources);
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

+ 1.2.2.9 `SpringApplication#refreshContext` 刷新上下文
```java
  private void refreshContext(ConfigurableApplicationContext context) {
        if (this.registerShutdownHook) {
        shutdownHook.registerApplicationContext(context);
        }

        this.refresh(context);
        }

protected void refresh(ConfigurableApplicationContext applicationContext) {
        applicationContext.refresh();
        }
```

> `AbstractApplicationContext#refresh`
```java
    public void refresh() throws BeansException, IllegalStateException {
synchronized(this.startupShutdownMonitor) {
        StartupStep contextRefresh = this.applicationStartup.start("spring.context.refresh");
        // 准备刷新
        this.prepareRefresh();
        // 初始化BeanFactory,解析XML
        ConfigurableListableBeanFactory beanFactory = this.obtainFreshBeanFactory();
        // 准备BeanFactory
        this.prepareBeanFactory(beanFactory);

        try {
        // BeanFactory 后置流程处理,由子类实现进行处理 
        this.postProcessBeanFactory(beanFactory);
        StartupStep beanPostProcess = this.applicationStartup.start("spring.context.beans.post-process");
        // 激活 BeanFactory 处理器
        this.invokeBeanFactoryPostProcessors(beanFactory);
        // 注册拦截Bean创建的Bean处理器
        this.registerBeanPostProcessors(beanFactory);
        beanPostProcess.end();
        // 初始化资源文件
        this.initMessageSource();
        // 初始化事件广播器
        this.initApplicationEventMulticaster();
        // 由子类实现处理
        this.onRefresh();
        // 注册 listener 
        this.registerListeners();
        // 完成 BeanFactory 设置转换器
        this.finishBeanFactoryInitialization(beanFactory);
        // 完成完成操作
        this.finishRefresh();
        } catch (BeansException var10) {
        if (this.logger.isWarnEnabled()) {
        this.logger.warn("Exception encountered during context initialization - cancelling refresh attempt: " + var10);
        }

        this.destroyBeans();
        this.cancelRefresh(var10);
        throw var10;
        } finally {
        this.resetCommonCaches();
        contextRefresh.end();
        }

        }
        }

protected void prepareRefresh() {
        this.startupDate = System.currentTimeMillis();
        this.closed.set(false);
        this.active.set(true);
        if (this.logger.isDebugEnabled()) {
        if (this.logger.isTraceEnabled()) {
        this.logger.trace("Refreshing " + this);
        } else {
        this.logger.debug("Refreshing " + this.getDisplayName());
        }
        }

        this.initPropertySources();
        this.getEnvironment().validateRequiredProperties();
        if (this.earlyApplicationListeners == null) {
        this.earlyApplicationListeners = new LinkedHashSet(this.applicationListeners);
        } else {
        this.applicationListeners.clear();
        this.applicationListeners.addAll(this.earlyApplicationListeners);
        }

        this.earlyApplicationEvents = new LinkedHashSet();
        }

protected ConfigurableListableBeanFactory obtainFreshBeanFactory() {
        this.refreshBeanFactory();
        return this.getBeanFactory();
        }

protected void prepareBeanFactory(ConfigurableListableBeanFactory beanFactory) {
        beanFactory.setBeanClassLoader(this.getClassLoader());
        if (!shouldIgnoreSpel) {
        beanFactory.setBeanExpressionResolver(new StandardBeanExpressionResolver(beanFactory.getBeanClassLoader()));
        }

        beanFactory.addPropertyEditorRegistrar(new ResourceEditorRegistrar(this, this.getEnvironment()));
        beanFactory.addBeanPostProcessor(new ApplicationContextAwareProcessor(this));
        beanFactory.ignoreDependencyInterface(EnvironmentAware.class);
        beanFactory.ignoreDependencyInterface(EmbeddedValueResolverAware.class);
        beanFactory.ignoreDependencyInterface(ResourceLoaderAware.class);
        beanFactory.ignoreDependencyInterface(ApplicationEventPublisherAware.class);
        beanFactory.ignoreDependencyInterface(MessageSourceAware.class);
        beanFactory.ignoreDependencyInterface(ApplicationContextAware.class);
        beanFactory.ignoreDependencyInterface(ApplicationStartupAware.class);
        beanFactory.registerResolvableDependency(BeanFactory.class, beanFactory);
        beanFactory.registerResolvableDependency(ResourceLoader.class, this);
        beanFactory.registerResolvableDependency(ApplicationEventPublisher.class, this);
        beanFactory.registerResolvableDependency(ApplicationContext.class, this);
        beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(this));
        if (!NativeDetector.inNativeImage() && beanFactory.containsBean("loadTimeWeaver")) {
        beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
        beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
        }

        if (!beanFactory.containsLocalBean("environment")) {
        beanFactory.registerSingleton("environment", this.getEnvironment());
        }

        if (!beanFactory.containsLocalBean("systemProperties")) {
        beanFactory.registerSingleton("systemProperties", this.getEnvironment().getSystemProperties());
        }

        if (!beanFactory.containsLocalBean("systemEnvironment")) {
        beanFactory.registerSingleton("systemEnvironment", this.getEnvironment().getSystemEnvironment());
        }

        if (!beanFactory.containsLocalBean("applicationStartup")) {
        beanFactory.registerSingleton("applicationStartup", this.getApplicationStartup());
        }

        }

protected void invokeBeanFactoryPostProcessors(ConfigurableListableBeanFactory beanFactory) {
        PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors(beanFactory, this.getBeanFactoryPostProcessors());
        if (!NativeDetector.inNativeImage() && beanFactory.getTempClassLoader() == null && beanFactory.containsBean("loadTimeWeaver")) {
        beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
        beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
        }

        }
```
> `AbstractApplicationContext#registerBeanPostProcessors` 注册拦截Bean创建的Bean处理器
```java
protected void registerBeanPostProcessors(ConfigurableListableBeanFactory beanFactory) {
        PostProcessorRegistrationDelegate.registerBeanPostProcessors(beanFactory, this);
        }
```

> `AbstractApplicationContext#initMessageSource` 初始化资源文件
```java
protected void initMessageSource() {
        ConfigurableListableBeanFactory beanFactory = this.getBeanFactory();
        if (beanFactory.containsLocalBean("messageSource")) {
        this.messageSource = (MessageSource)beanFactory.getBean("messageSource", MessageSource.class);
        if (this.parent != null && this.messageSource instanceof HierarchicalMessageSource) {
        HierarchicalMessageSource hms = (HierarchicalMessageSource)this.messageSource;
        if (hms.getParentMessageSource() == null) {
        hms.setParentMessageSource(this.getInternalParentMessageSource());
        }
        }

        if (this.logger.isTraceEnabled()) {
        this.logger.trace("Using MessageSource [" + this.messageSource + "]");
        }
        } else {
        DelegatingMessageSource dms = new DelegatingMessageSource();
        dms.setParentMessageSource(this.getInternalParentMessageSource());
        this.messageSource = dms;
        beanFactory.registerSingleton("messageSource", this.messageSource);
        if (this.logger.isTraceEnabled()) {
        this.logger.trace("No 'messageSource' bean, using [" + this.messageSource + "]");
        }
        }

        }
```

> `AbstractApplicationContext#initApplicationEventMulticaster`初始化事件广播器
```java
protected void initApplicationEventMulticaster() {
        ConfigurableListableBeanFactory beanFactory = this.getBeanFactory();
        if (beanFactory.containsLocalBean("applicationEventMulticaster")) {
        this.applicationEventMulticaster = (ApplicationEventMulticaster)beanFactory.getBean("applicationEventMulticaster", ApplicationEventMulticaster.class);
        if (this.logger.isTraceEnabled()) {
        this.logger.trace("Using ApplicationEventMulticaster [" + this.applicationEventMulticaster + "]");
        }
        } else {
        this.applicationEventMulticaster = new SimpleApplicationEventMulticaster(beanFactory);
        beanFactory.registerSingleton("applicationEventMulticaster", this.applicationEventMulticaster);
        if (this.logger.isTraceEnabled()) {
        this.logger.trace("No 'applicationEventMulticaster' bean, using [" + this.applicationEventMulticaster.getClass().getSimpleName() + "]");
        }
        }

        }
```

> `AbstractApplicationContext#registerListeners` 注册 listener
```java
protected void registerListeners() {
        Iterator var1 = this.getApplicationListeners().iterator();

        while(var1.hasNext()) {
        ApplicationListener<?> listener = (ApplicationListener)var1.next();
        this.getApplicationEventMulticaster().addApplicationListener(listener);
        }

        String[] listenerBeanNames = this.getBeanNamesForType(ApplicationListener.class, true, false);
        String[] var7 = listenerBeanNames;
        int var3 = listenerBeanNames.length;

        for(int var4 = 0; var4 < var3; ++var4) {
        String listenerBeanName = var7[var4];
        this.getApplicationEventMulticaster().addApplicationListenerBean(listenerBeanName);
        }

        Set<ApplicationEvent> earlyEventsToProcess = this.earlyApplicationEvents;
        this.earlyApplicationEvents = null;
        if (!CollectionUtils.isEmpty(earlyEventsToProcess)) {
        Iterator var9 = earlyEventsToProcess.iterator();

        while(var9.hasNext()) {
        ApplicationEvent earlyEvent = (ApplicationEvent)var9.next();
        this.getApplicationEventMulticaster().multicastEvent(earlyEvent);
        }
        }

        }
```

> `AbstractApplicationContext#finishBeanFactoryInitialization` 完成 BeanFactory 设置转换器
```java
protected void finishBeanFactoryInitialization(ConfigurableListableBeanFactory beanFactory) {
        if (beanFactory.containsBean("conversionService") && beanFactory.isTypeMatch("conversionService", ConversionService.class)) {
        beanFactory.setConversionService((ConversionService)beanFactory.getBean("conversionService", ConversionService.class));
        }

        if (!beanFactory.hasEmbeddedValueResolver()) {
        beanFactory.addEmbeddedValueResolver((strVal) -> {
        return this.getEnvironment().resolvePlaceholders(strVal);
        });
        }

        String[] weaverAwareNames = beanFactory.getBeanNamesForType(LoadTimeWeaverAware.class, false, false);
        String[] var3 = weaverAwareNames;
        int var4 = weaverAwareNames.length;

        for(int var5 = 0; var5 < var4; ++var5) {
        String weaverAwareName = var3[var5];
        this.getBean(weaverAwareName);
        }

        beanFactory.setTempClassLoader((ClassLoader)null);
        beanFactory.freezeConfiguration();
        beanFactory.preInstantiateSingletons();
        }
```

+ 1.2.2.10 `SpringApplication#afterRefresh`, 刷新后执行
> 模板方法, 由子类实现处理
```java
    protected void afterRefresh(ConfigurableApplicationContext context, ApplicationArguments args) {
        }
```

+ 1.2.2.11 `SpringApplication#callRunners`, 执行Runners
```java
    private void callRunners(ApplicationContext context, ApplicationArguments args) {
        List<Object> runners = new ArrayList();
        runners.addAll(context.getBeansOfType(ApplicationRunner.class).values());
        runners.addAll(context.getBeansOfType(CommandLineRunner.class).values());
        AnnotationAwareOrderComparator.sort(runners);
        Iterator var4 = (new LinkedHashSet(runners)).iterator();

        while(var4.hasNext()) {
        Object runner = var4.next();
        if (runner instanceof ApplicationRunner) {
        this.callRunner((ApplicationRunner)runner, args);
        }

        if (runner instanceof CommandLineRunner) {
        this.callRunner((CommandLineRunner)runner, args);
        }
        }

        }
```
