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
<!-- more -->
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