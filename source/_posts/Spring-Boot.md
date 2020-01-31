---
title: Spring Boot
date: 2020-01-29 20:57:11
tags:
    - Java
    - Springboot
categories:
    - Technology
---

## 容器

### IOC

spring-context

- @Configuration

  告诉Spring这是一个配置类

- @Bean

  给容器中注册一个Bean，类型为返回值的类型，id默认是用方法
  默认都是单例

- AnnotationConfigApplicationContext
- @ComponetScan、@ComponetScans

  value：扫描@Service @Controller @Repositroy @Componet
  excludeFlters：排除的规则
  includeFilters：只需要包含的规则

	- FilterType.ANNOTATION

	  按照注解

	- FilterType.ASSIGNABLE_TYPE

	  按照给定类型

	- FilterType.ASPECTJ

	  使用ASPECTJ表达式

	- FilterType.REGEX

	  使用正则指定

	- FilterType.CUSTOM

	  自定义规则，必须实现TypeFilter类

- @Scope

	- SCOPE_PROTOTPE

	  多实例
	  IOC容器启动不回去调用方法床架你对象，获取的时候创建对象

	- SCOPE_SINGLETON

	  单实例（默认值）
	  IOC容器启动会调用方法创建对象放到IOC容器中

	- SCOPE_REQUEST

	  同一次请求创建一个实例

	- SCOPE_SESSION

	  同一个Session创建一个实例

- @Lazy 

  懒加载：容器启动不创建对象。第一次使用（获取）Bean对象时调用创建对象放入容器

- @Conditional

  按照一定的条件进行判断，满足条件给容器中注册Bean
  需要实现Condition 接口：ConditionContext 判断条件能使用的上下文；AnnotatedTypeMetadate 注释信息

- @Import

  导入组件，id默认是组件的全类名（com.xxx.xxx.xx）

	- ImportSelector

	  自定义逻辑返回需要导入的组件。
	  返回值就是导入到容器中的组件全类名，方法不要返回null。
	  AnnotationMetadata 当前标注@Import注解的类的所有注释信息。

	- ImportBeanDefinitionRegistrar

	  BeanDefinitionRegistry：BeanDefinition注册类。
	  调用registerBeanDefinition手工注册Bean

- FactoryBean

  实现FactoryBean接口
  1.工厂Bean获取的是调用getObject创建的对象
  2.要获取工程Bean本身，需要给id前面加一个&

### bean的生命周期

- 初始化

	- @Bean(initMethod)
	- InitializingBean

	  通过让Bean实现该接口，实现Bean初始化逻辑

	- @PostConstruct
	- BeanPostProcessor

	  Bean后置处理器，在Bean初始化前后进行一些处理工作

		- postProcessBeforeInitialization

		  在初始化之前

		- postProcessAfterInitialization

		  在初始化工作之后

		- ApplicationContextAwareProcessor

		  实现该接口，Spring容器可以将当前容器传给实现类

		- InitDestroyAnnotationBeanPostProcessor

		  处理@PostConstruct，@PreDestory注解

		- AutowiredAnnotationBeanPostProcessor

		  处理@Autowired注解

- 销毁

	- @Bean(destroyMethod)

	  多实例的bean，容器不会调用销毁方法

	- DisposableBean

	  通过实现该接口，实现销毁逻辑

	- @PreDestory

### 属性赋值

- @Value

	- 基本数值
	- SpEL  #{}
	- 配置文件 ${}

	  @PropertySource 读取外部配置文件，保存至环境变量中

### 自动装配

- @Autowired

  1.默认按照类型在容器中找相应的组件
  2.如果找到多个相同类型的组件，再将属性的名称作为组件的id组容易中寻找
  3.使用@Qualifier指定需要装配的组件的id
  4.自动装配默认一定要指定存在的属性（如果不强制指定，使用require=false）
  5.@Primary,让Spring进行自动装配的时候，默认是用首选的bean

	- 构造器

	  构造器用到的自定义类型的值从IOC容器中获取
	  如果组建只有一个有参构造器，这个有参构造器的@Autowired可以省略，参数位置的组件还是可以从容器中获取

	- 方法

	  标注在方法，Spring容器创建当前对象，就会调用方法完成赋值
	  方法使用的参数，自定义类型的值从IOC容器中获取

	- 属性

	  值从IOC容器中获取

- @Resource

  可以和@Autowired一样实现自动装配，默认按照属性名称进行装配
  没有支持@Primary功能，reqiured=false

- @Inject

  需要依赖javax.inject包，和Autowired一样，没有reqiured=false

- 实现xxxAware接口

  在创建对象的时候，会调用接口的回调方法
  把Spring底层的组件注册到自定义的Bean中
  xxxAware功能使用xxxProcessor处理的

	- ApplicationContextAware

	  获取ApplicationContext

	- BeanNameAware
	- EmbeddedValueResolverAware

- Profile

  加了环境表示的bean，只有这个环境被激活的时候才能注册到容器中，默认是default环境

	- 启动参数

	  -Dspring.profiles.active=xxx

	- 代码

	  1.创建一个aplicationContext
	  2.设置需要激活的环境.getEnvironment().setActiveProfiles()
	  3.注册朱配置类  .register()
	  4.启动刷新容器 .refresh()

### AOP

在程序运行期间动态的将某段代码切入到指定方法指定位置进行运行的编程方式
spring-aspects
容器中保存了组建的代理对象(cglib增强后的对象),这个对象里面保存了详细信息(比如增强器，目标对象)

- @Before

  在目标方法之前切入：切入点表达式（指定在那个方法切入）

- @After

  在目标方法之后切入：切入点表达式

- @PointCut

  切入点表达式

- @AfterReturning

  再放标方法正常返回之后运行

- @AfterThrowing

  在目标方法出现异常之后运行

- @Aspect

  告诉Spring当前类是一个切面类

- @EnableAspectJAutoProxy

  开启基于注解的AOP
  1.给容器中导入AspectJAutoProxyRegistrar，给容器中注册一个AnnotationAwareAspectJAutoProxyCreator

	- AnnotationAwareAspectJAutoProxyCreator

		- AspectJAwareAdvisorAutoProxyCreator

			- AbstractAdvisorAutoProxyCreator

				- AbstractAutoProxyCreator

					- SmartInstantiationAwareBeanPostProcessor
					- BeanFactroyAware

- JoinPoint

  JoinPoint一定要出现在参数表的第一位

- 目标方法执行

  1.CglibAopProxy.intercept()。拦截目标方法的执行。
  2.根据PoxyFactory对象获取将要执行的目标方法拦截器链
  	a)如果没有拦截器链，直接执行目标方法
  	b)如果有拦截器链，把需要执行的目标对象，目标方法，拦截器链等信息出入创建一个CglibMethodInvocation，并调用proceed()方法
  	c)拦截器链触发过程
  		1.如果没有拦截器执行目标方法，或者懒机器的索引和拦截器数组-1大小一样(指定到了最后一个拦截器)，执行目标方法。

### 声明式事务

1.导入相关依赖spring-jdbc
2.配置数据源 JdbcTmplate
3.给方法标注@Transactional
4.@EnableTransactionManagement 开启基于注解的事物
5.配置事物管理器管理事物

- @Transactional
- @EnableTransactionManagement

	- TransactionManagementConfigurationSelection

	  给容器中导入组件

		- AutoProxyRegistrar

		  给容器中注册InfrastructureAdvisorAutoProxyCreator组件

			- InfrastructureAdvisorAutoProxyCreator

			  利用后置处理器机制在对象创建后，包装对象，返回一个代理对象（增强器），代理对象执行方法利用拦截器链执行方法

		- ProxyTransactionManagementConfiguration

		  1.给容器中注册事物增强器
		  	事物增强器要用事物注解信息
		  	事物拦截器，保存了事物的属性信息，事物管理器，是一个MethodInterceptor，在目标方法执行时执行拦截器链

## 扩展原理

### BeanFactoryPostProcessor

BeanFactory后置处理器
	在BeanFactory标准初始化之后调用，所有的bean定义以保存在遭到beanFactory，但是bean还未创建

- BeanDefinitionRegistryPostProcessor

  在所有bean定义信息将要被加载，bean实力还未被创建时执行

### ApplicationListener

监听容器中发布的事件
	1.写一个监听器，ApplicationListener实现类，来监听某个事件ApplicationEvent机器子类
	2.把监听器加入到容器
	3.只要容器中有相关事件的腹部，我们就能监听到这个事件：
		ContextRefreshedEvent：容器舒心完成(所有bean都完全创建)会发布这个事件；
		ContextClosedEvent：关闭容器会发布这个事件
	4.发布一个事件，applicationContext.publishEvent()

### 容器的创建

1.Spring容器在启动的时候，先会保存所有注册进来的Bean的定义信息
	xml
	注解
2.Spring容器会合适的时机创建这些Bean
	用到这个bean的时候，利用getBean创建bean，创建好以后保存在容器中
	统一创建剩下所有的bean的时候，finshBeanFactoryInitialization()
3.后置处理器
	每个bean创建完成，都会使用各种后置处理器，来增强bean的功能
4.事件驱动模型
	ApplicationListener，事件监听
事件派发，ApplicationEvenMulticator

- refresh()

	- prepareRefresh()

	  刷新预处理工作:
	  清缓存
	  置状态

		- initPropertySources()

		  初始化属性设置，子类自定义个性化的属性设置方法。

		- getEnvironment().validateRequiredProperties()

		  检验属性的合法

		- earlyApplicationEvents

		  保存容器中的一起早起事件

	- obtainFreshBeanFactory()

	  获取BeanFactory

		- refreshBeanFactory()

		  刷新BeanFactory工厂，创建BeanFactory对象。

		- getBeanFactory()【DefaultListableBeanFactory】

		  获取之前创建好的BeanFactory

		- prepareBeanFactory()

		  BeanFactory的预准备工作（BeanFactory进行一些设置）
		  1.设置BeanFactory的类加载器，支持表达式解析器。。。
		  2.添加部分BeanPostProcessor【ApplicationContextAwareProcessor】
		  3.设置忽略的自动装配的接口，EnvironmentAware、EmbeddedValueResolverAware。。。
		  4.注册可以解析的自动装配，我们能直接在仍和组件中自动注入：BeanFactory、ResourceLoader、ApplicationEventPublisher、ApplicationContext
		  5.添加BeanPostProcessor【ApplicationListenerDetector】
		  6.添加编译是的AspectJ支持
		  7.给BeanFactory中注册一些能用的组件：environment【ConfigurableEnvironment】、systemProperties【Map】

		- postProcessBeanFactory()

		  BeanFactory准备完成后的后置处理工作，子类用过重写这个发方法来在BeanFactory初始化完成后进行更进一步的工作

		- invokeBeanFactoryPostProcessors()

		  执行BeanFactoryPostProcessor：BeanFactory的后置处理器，在BeanFactory标准初始化之后执行
		  
		  执行BeanFactoryPostProcessor的方法：
		  1.先执行BeanDefinitonRegistryPostProcessor
		  	获取所有的BeanDefinitonRegistryPostProcessor
		  	先执行实现了PriorityOrdered接口的
		  	再执行实现了Ordered接口的
		  	最后执行剩下的
		  2.再执行BeanFactoryPostProcessor的方法
		  	先执行实现了PriorityOrdered接口的
		  	再执行实现了Ordered接口的
		  	最后执行剩下的

	- registerBeanPostProcessors()

	  注册BeanPostProcessor，Bean的后置处理器。
	  1.获取所有的BeanPostProcessor，不同接口类型的BeanPostProcessor，在Bean船舰前后的执行实际是不一样的。
	  	BeanPostProcessor
	  	DestructionAwareBeanPostProcessor
	  	InstantiationAwareBeanPostProcessor
	  	SmartInstantiationAwareBeanPostProcessor
	  	MergedBeanDefinitionPostProcessor

	- initMessageSource()

	  初始化MessageSource组件（做国际化功能，消息绑定、解析）
	  1.获取BeanFactory
	  2.看容器中是否有id为messageSource的组件
	  	如果有赋值给messageSource，如果没有自己创建【DelegatingMessageSource】
	  3.把创建好的MessageSource注册在容器中

	- initApplicationEventMulticaster()

	  初始化事件派发器
	  1.从BeanFactory中获取applicationEventMulticaster的ApplicationEventMulticaster
	  2.如果没有配置，创建SimpleApplicationEventMulticaster
	  3.注册到容器中

	- onRefresh()

	  留给子容器的（子类）
	  1.子类重写这个方法，在容器刷新的时候可以自定义逻辑

	- registerLiseners()

	  给容器中将所有的项目的ApplicationListener注册进来。
	  1.从容器中拿到所有ApplicationListener组件
	  2.将每个监听器添加到事件派发器中
	  3.派发之前步骤产生的事件

	- finishBeanFactoryInitialization()

	  初始化剩下的所有单实例bean

		- beanFactory.preInstantiateSingletons()

		  初始化剩下的单实例bean
		  1.获取容器中所有的Bean定义
		  2.拿到Bean的定义信息
		  3.Bean不是抽象的，是单实例的，不是懒加载的
		  	1.判断是否是FactoryBean，是否是实现FactoryBean接口的Bean
		  	2.不是FactoryBean，用getBean创建对象
		  	3.doGetBean()
		  	4.先获取缓存中保存的单实例Bean。如果获取到，说明这个Bean之前被创建国（所有创建国的单实例Bean都是会被缓存起来的）
		  	5.换种种获取不到，开始Bean的创建对象流程
		  	6.标记当前bean已经被创建
		  	7.获取Bean的定义信息
		  	8.获取当前Bean依赖的其他Bean，如果有就按照getBean()把以来的Bean先创建出来。
		  	9.启动单实例Bean的创建流程
		  		1.createBean()
		  		2.resolveBeforeInstantiation()让BeanPostProcessor先拦截返回代理对象【InstantiationAwareBeanPostProcessors】提前执行
		  		3.如果前面的InstantiationAwareBeanPostProcessors没有返回代理对象
		  		4.doCreateBean()创建Bean
		  			1.创建Bean实例，createBeanInstance()利用工厂方法或者对象的构造器创建出Bean实例
		  			2.applyMergedBeanDefinitionPostProcessors()，调用【MergedBeanDefinitionPostProcessors】
		  			3.populateBean() Bean属性赋值

	- finishRefresh()

	  完成BeanFactory的初始化创建工作，IOC容器就创建完成

		- initLifecycleProcessor()

		  初始化和生命周期有关的后置处理器：LifecycleProcessor。
		  写一个LifecycleProcessor的实现类，可以在BeanFactory的onRefresh(),onClose()进行拦截

		- getLifecycleProcessor().onRefresh()

		  拿到前面定义的生命周期处理器，回调onRefresh()

		- publishEvent()

		  发布容器完成事件