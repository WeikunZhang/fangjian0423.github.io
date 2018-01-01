title: SpringBoot源码分析之Spring容器的refresh过程
date: 2017-05-10 19:55:21
tags:
- springboot
- java
- springboot源码分析
categories: springboot

----------------

上一篇文章中，我们分析了SpringBoot的启动过程：构造SpringApplication并调用它的run方法。其中构造SpringApplication的时候会初始化一些监听器和初始化器；run方法调用的过程中会有对应的监听器监听，并且会创建Spring容器。

Spring容器创建之后，会调用它的refresh方法，refresh的时候会做很多事情：比如完成配置类的解析、各种BeanFactoryPostProcessor和BeanPostProcessor的注册、国际化配置的初始化、web内置容器的构造等等。

我们来分析一下这个refresh过程。

<!--more-->

还是以web程序为例，那么对应的Spring容器为AnnotationConfigEmbeddedWebApplicationContext。它的refresh方法调用了父类AbstractApplicationContext的refresh方法：

```java
    public void refresh() throws BeansException, IllegalStateException {
      // refresh过程只能一个线程处理，不允许并发执行
      synchronized (this.startupShutdownMonitor) {
        prepareRefresh();
        ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();
        prepareBeanFactory(beanFactory);
        try {
          postProcessBeanFactory(beanFactory);
          invokeBeanFactoryPostProcessors(beanFactory);
          registerBeanPostProcessors(beanFactory);
          initMessageSource();
          initApplicationEventMulticaster();
          onRefresh();
          registerListeners();
          finishBeanFactoryInitialization(beanFactory);
          finishRefresh();
        }
        catch (BeansException ex) {
          if (logger.isWarnEnabled()) {
            logger.warn("Exception encountered during context initialization - " +
                "cancelling refresh attempt: " + ex);
          }
          destroyBeans();
          cancelRefresh(ex);
          throw ex;
        }
        finally {
          resetCommonCaches();
        }
      }
    }
```

## prepareRefresh方法

表示在真正做refresh操作之前需要准备做的事情：

1. 设置Spring容器的启动时间，撤销关闭状态，开启活跃状态。
2. 初始化属性源信息(Property)
3. 验证环境信息里一些必须存在的属性

## prepareBeanFactory方法

从Spring容器获取BeanFactory(Spring Bean容器)并进行相关的设置为后续的使用做准备：

1. 设置classloader(用于加载bean)，设置表达式解析器(解析bean定义中的一些表达式)，添加属性编辑注册器(注册属性编辑器)
2. 添加ApplicationContextAwareProcessor这个BeanPostProcessor。取消ResourceLoaderAware、ApplicationEventPublisherAware、MessageSourceAware、ApplicationContextAware、EnvironmentAware这5个接口的自动注入。因为ApplicationContextAwareProcessor把这5个接口的实现工作做了
3. 设置特殊的类型对应的bean。BeanFactory对应刚刚获取的BeanFactory；ResourceLoader、ApplicationEventPublisher、ApplicationContext这3个接口对应的bean都设置为当前的Spring容器
4. 注入一些其它信息的bean，比如environment、systemProperties等

## postProcessBeanFactory方法

BeanFactory设置之后再进行后续的一些BeanFactory操作。

不同的Spring容器做不同的操作。比如GenericWebApplicationContext容器会在BeanFactory中添加ServletContextAwareProcessor用于处理ServletContextAware类型的bean初始化的时候调用setServletContext或者setServletConfig方法(跟ApplicationContextAwareProcessor原理一样)。

AnnotationConfigEmbeddedWebApplicationContext对应的postProcessBeanFactory方法：

```java
    @Override
    protected void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) {
      // 调用父类EmbeddedWebApplicationContext的实现
      super.postProcessBeanFactory(beanFactory);
      // 查看basePackages属性，如果设置了会使用ClassPathBeanDefinitionScanner去扫描basePackages包下的bean并注册
      if (this.basePackages != null && this.basePackages.length > 0) {
        this.scanner.scan(this.basePackages);
      }
      // 查看annotatedClasses属性，如果设置了会使用AnnotatedBeanDefinitionReader去注册这些bean
      if (this.annotatedClasses != null && this.annotatedClasses.length > 0) {
        this.reader.register(this.annotatedClasses);
      }
    }
```

父类EmbeddedWebApplicationContext的实现：

```java
    @Override
    protected void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) {
      beanFactory.addBeanPostProcessor(
          new WebApplicationContextServletContextAwareProcessor(this));
      beanFactory.ignoreDependencyInterface(ServletContextAware.class);
    }
```

## invokeBeanFactoryPostProcessors方法

在Spring容器中找出实现了BeanFactoryPostProcessor接口的processor并执行。Spring容器会委托给PostProcessorRegistrationDelegate的invokeBeanFactoryPostProcessors方法执行。

介绍两个接口：

1. BeanFactoryPostProcessor：用来修改Spring容器中已经存在的bean的定义，使用ConfigurableListableBeanFactory对bean进行处理
2. BeanDefinitionRegistryPostProcessor：继承BeanFactoryPostProcessor，作用跟BeanFactoryPostProcessor一样，只不过是使用BeanDefinitionRegistry对bean进行处理

基于web程序的Spring容器AnnotationConfigEmbeddedWebApplicationContext构造的时候，会初始化内部属性AnnotatedBeanDefinitionReader reader，这个reader构造的时候会在BeanFactory中注册一些post processor，包括BeanPostProcessor和BeanFactoryPostProcessor(比如ConfigurationClassPostProcessor、AutowiredAnnotationBeanPostProcessor)：

    AnnotationConfigUtils.registerAnnotationConfigProcessors(this.registry);

invokeBeanFactoryPostProcessors方法处理BeanFactoryPostProcessor的逻辑如下：

从Spring容器中找出BeanDefinitionRegistryPostProcessor类型的bean(这些processor是在容器刚创建的时候通过构造AnnotatedBeanDefinitionReader的时候注册到容器中的)，然后按照优先级分别执行，优先级的逻辑如下：

1. 实现PriorityOrdered接口的BeanDefinitionRegistryPostProcessor先全部找出来，然后排序后依次执行
2. 实现Ordered接口的BeanDefinitionRegistryPostProcessor找出来，然后排序后依次执行
3. 没有实现PriorityOrdered和Ordered接口的BeanDefinitionRegistryPostProcessor找出来执行并依次执行

接下来从Spring容器内查找BeanFactoryPostProcessor接口的实现类，然后执行(如果processor已经执行过，则忽略)，这里的查找规则跟上面查找BeanDefinitionRegistryPostProcessor一样，先找PriorityOrdered，然后是Ordered，最后是两者都没。

这里需要说明的是ConfigurationClassPostProcessor这个processor是优先级最高的被执行的processor(实现了PriorityOrdered接口)。这个ConfigurationClassPostProcessor会去BeanFactory中找出所有有@Configuration注解的bean，然后使用ConfigurationClassParser去解析这个类。ConfigurationClassParser内部有个Map<ConfigurationClass, ConfigurationClass>类型的configurationClasses属性用于保存解析的类，ConfigurationClass是一个对要解析的配置类的封装，内部存储了配置类的注解信息、被@Bean注解修饰的方法、@ImportResource注解修饰的信息、ImportBeanDefinitionRegistrar等都存储在这个封装类中。

这里ConfigurationClassPostProcessor最先被处理还有另外一个原因是如果程序中有自定义的BeanFactoryPostProcessor，那么这个PostProcessor首先得通过ConfigurationClassPostProcessor被解析出来，然后才能被Spring容器找到并执行。(ConfigurationClassPostProcessor不先执行的话，这个Processor是不会被解析的，不会被解析的话也就不会执行了)。

在我们的程序中，只有主类RefreshContextApplication有@Configuration注解(@SpringBootApplication注解带有@Configuration注解)，所以这个配置类会被ConfigurationClassParser解析。解析过程如下：

1. 处理@PropertySources注解：进行一些配置信息的解析
2. 处理@ComponentScan注解：使用ComponentScanAnnotationParser扫描basePackage下的需要解析的类(@SpringBootApplication注解也包括了@ComponentScan注解，只不过basePackages是空的，空的话会去获取当前@Configuration修饰的类所在的包)，并注册到BeanFactory中(这个时候bean并没有进行实例化，而是进行了注册。具体的实例化在finishBeanFactoryInitialization方法中执行)。对于扫描出来的类，递归解析
3. 处理@Import注解：先递归找出所有的注解，然后再过滤出只有@Import注解的类，得到@Import注解的值。比如查找@SpringBootApplication注解的@Import注解数据的话，首先发现@SpringBootApplication不是一个@Import注解，然后递归调用修饰了@SpringBootApplication的注解，发现有个@EnableAutoConfiguration注解，再次递归发现被@Import(EnableAutoConfigurationImportSelector.class)修饰，还有@AutoConfigurationPackage注解修饰，再次递归@AutoConfigurationPackage注解，发现被@Import(AutoConfigurationPackages.Registrar.class)注解修饰，所以@SpringBootApplication注解对应的@Import注解有2个，分别是@Import(AutoConfigurationPackages.Registrar.class)和@Import(EnableAutoConfigurationImportSelector.class)。找出所有的@Import注解之后，开始处理逻辑：
     1. 遍历这些@Import注解内部的属性类集合
     2. 如果这个类是个ImportSelector接口的实现类，实例化这个ImportSelector，如果这个类也是DeferredImportSelector接口的实现类，那么加入ConfigurationClassParser的deferredImportSelectors属性中让第6步处理。否则调用ImportSelector的selectImports方法得到需要Import的类，然后对这些类递归做@Import注解的处理
     3. 如果这个类是ImportBeanDefinitionRegistrar接口的实现类，设置到配置类的importBeanDefinitionRegistrars属性中
     4. 其它情况下把这个类入队到ConfigurationClassParser的importStack(队列)属性中，然后把这个类当成是@Configuration注解修饰的类递归重头开始解析这个类
4. 处理@ImportResource注解：获取@ImportResource注解的locations属性，得到资源文件的地址信息。然后遍历这些资源文件并把它们添加到配置类的importedResources属性中
5. 处理@Bean注解：获取被@Bean注解修饰的方法，然后添加到配置类的beanMethods属性中
6. 处理DeferredImportSelector：处理第3步@Import注解产生的DeferredImportSelector，进行selectImports方法的调用找出需要import的类，然后再调用第3步相同的处理逻辑处理

这里@SpringBootApplication注解被@EnableAutoConfiguration修饰，@EnableAutoConfiguration注解被@Import(EnableAutoConfigurationImportSelector.class)修饰，所以在第3步会找出这个@Import修饰的类EnableAutoConfigurationImportSelector，这个类刚好实现了DeferredImportSelector接口，接着就会在第6步被执行。第6步selectImport得到的类就是自动化配置类。

EnableAutoConfigurationImportSelector的selectImport方法会在spring.factories文件中找出key为EnableAutoConfiguration对应的值，有81个，这81个就是所谓的自动化配置类(XXXAutoConfiguration)。

ConfigurationClassParser解析完成之后，被解析出来的类会放到configurationClasses属性中。然后使用ConfigurationClassBeanDefinitionReader去解析这些类。

这个时候这些bean只是被加载到了Spring容器中。下面这段代码是ConfigurationClassBeanDefinitionReader的解析bean过程：

```java
    public void loadBeanDefinitions(Set<ConfigurationClass> configurationModel) {
      TrackedConditionEvaluator trackedConditionEvaluator = new TrackedConditionEvaluator();
      for (ConfigurationClass configClass : configurationModel) {
        // 对每一个配置类，调用loadBeanDefinitionsForConfigurationClass方法
        loadBeanDefinitionsForConfigurationClass(configClass, trackedConditionEvaluator);
      }
    }

    private void loadBeanDefinitionsForConfigurationClass(ConfigurationClass configClass,
        TrackedConditionEvaluator trackedConditionEvaluator) {
      // 使用条件注解判断是否需要跳过这个配置类
      if (trackedConditionEvaluator.shouldSkip(configClass)) {
        // 跳过配置类的话在Spring容器中移除bean的注册
        String beanName = configClass.getBeanName();
        if (StringUtils.hasLength(beanName) && this.registry.containsBeanDefinition(beanName)) {
          this.registry.removeBeanDefinition(beanName);
        }
        this.importRegistry.removeImportingClassFor(configClass.getMetadata().getClassName());
        return;
      }

      if (configClass.isImported()) {
        // 如果自身是被@Import注释所import的，注册自己
        registerBeanDefinitionForImportedConfigurationClass(configClass);
      }
      // 注册方法中被@Bean注解修饰的bean
      for (BeanMethod beanMethod : configClass.getBeanMethods()) {
        loadBeanDefinitionsForBeanMethod(beanMethod);
      }
      // 注册@ImportResource注解注释的资源文件中的bean
      loadBeanDefinitionsFromImportedResources(configClass.getImportedResources());
      // 注册@Import注解中的ImportBeanDefinitionRegistrar接口的registerBeanDefinitions
      loadBeanDefinitionsFromRegistrars(configClass.getImportBeanDefinitionRegistrars());
    }
```

invokeBeanFactoryPostProcessors方法总结来说就是从Spring容器中找出BeanDefinitionRegistryPostProcessor和BeanFactoryPostProcessor接口的实现类并按照一定的规则顺序进行执行。 其中ConfigurationClassPostProcessor这个BeanDefinitionRegistryPostProcessor优先级最高，它会对项目中的@Configuration注解修饰的类(@Component、@ComponentScan、@Import、@ImportResource修饰的类也会被处理)进行解析，解析完成之后把这些bean注册到BeanFactory中。需要注意的是这个时候注册进来的bean还没有实例化。

下面这图就是对ConfigurationClassPostProcessor后置器的总结：

![](http://7x2wh6.com1.z0.glb.clouddn.com/configuration-annotation-process.png)

## registerBeanPostProcessors方法

从Spring容器中找出的BeanPostProcessor接口的bean，并设置到BeanFactory的属性中。之后bean被实例化的时候会调用这个BeanPostProcessor。

该方法委托给了PostProcessorRegistrationDelegate类的registerBeanPostProcessors方法执行。这里的过程跟invokeBeanFactoryPostProcessors类似：

1. 先找出实现了PriorityOrdered接口的BeanPostProcessor并排序后加到BeanFactory的BeanPostProcessor集合中
2. 找出实现了Ordered接口的BeanPostProcessor并排序后加到BeanFactory的BeanPostProcessor集合中
3. 没有实现PriorityOrdered和Ordered接口的BeanPostProcessor加到BeanFactory的BeanPostProcessor集合中

这些已经存在的BeanPostProcessor在postProcessBeanFactory方法中已经说明，都是由AnnotationConfigUtils的registerAnnotationConfigProcessors方法注册的。这些BeanPostProcessor包括有AutowiredAnnotationBeanPostProcessor(处理被@Autowired注解修饰的bean并注入)、RequiredAnnotationBeanPostProcessor(处理被@Required注解修饰的方法)、CommonAnnotationBeanPostProcessor(处理@PreDestroy、@PostConstruct、@Resource等多个注解的作用)等。

如果是自定义的BeanPostProcessor，已经被ConfigurationClassPostProcessor注册到容器内。

这些BeanPostProcessor会在这个方法内被实例化(通过调用BeanFactory的getBean方法，如果没有找到实例化的类，就会去实例化)。

## initMessageSource方法

在Spring容器中初始化一些国际化相关的属性。

## initApplicationEventMulticaster方法

在Spring容器中初始化事件广播器，事件广播器用于事件的发布。

在[SpringBoot源码分析之SpringBoot的启动过程](http://fangjian0423.github.io/2017/04/30/springboot-startup-analysis/)中分析过，EventPublishingRunListener这个SpringApplicationRunListener会监听事件，其中发生contextPrepared事件的时候EventPublishingRunListener会把事件广播器注入到BeanFactory中。

所以initApplicationEventMulticaster不再需要再次注册，只需要拿出BeanFactory中的事件广播器然后设置到Spring容器的属性中即可。如果没有使用SpringBoot的话，Spring容器得需要自己初始化事件广播器。

## onRefresh方法

一个模板方法，不同的Spring容器做不同的事情。

比如web程序的容器AnnotationConfigEmbeddedWebApplicationContext中会调用createEmbeddedServletContainer方法去创建内置的Servlet容器。

目前SpringBoot只支持3种内置的Servlet容器：

1. Tomcat
2. Jetty
3. Undertow

## registerListeners方法

把Spring容器内的时间监听器和BeanFactory中的时间监听器都添加的事件广播器中。

然后如果存在early event的话，广播出去。

## finishBeanFactoryInitialization方法

实例化BeanFactory中已经被注册但是未实例化的所有实例(懒加载的不需要实例化)。

比如invokeBeanFactoryPostProcessors方法中根据各种注解解析出来的类，在这个时候都会被初始化。

实例化的过程各种BeanPostProcessor开始起作用。

## finishRefresh方法

refresh做完之后需要做的其他事情。

1. 初始化生命周期处理器，并设置到Spring容器中(LifecycleProcessor)
2. 调用生命周期处理器的onRefresh方法，这个方法会找出Spring容器中实现了SmartLifecycle接口的类并进行start方法的调用
3. 发布ContextRefreshedEvent事件告知对应的ApplicationListener进行响应的操作
4. 调用LiveBeansView的registerApplicationContext方法：如果设置了JMX相关的属性，则就调用该方法
5. 发布EmbeddedServletContainerInitializedEvent事件告知对应的ApplicationListener进行响应的操作

## 总结

Spring容器的refresh过程就是上述11个方法的介绍。内容还是非常多的，本文也只是说了个大概，像bean的实例化过程没有具体去分析，这方面的内容以后会看情况去做分析。

这篇文章也是为之后的文章比如内置Servlet容器的创建启动、条件注解的使用等打下基础。

## 例子

写了一个例子用来验证容器的refresh过程，包括bean解析，processor的使用、Lifecycle的使用等。

可以启动项目debug去看看对应的过程，这样对Spring容器会有一个更好的理解。

地址在：https://github.com/fangjian0423/springboot-analysis/tree/master/springboot-refresh-context
