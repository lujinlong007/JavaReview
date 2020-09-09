## Spring生命周期，流程梳理
	   Bean的实例化：通过构造器或者工厂方法等创建一个实例
     执行一系列的Aware接口：这些接口大家在工作中应该经常会见到，比如BeanClassLoaderAware、ApplicationContextAware等实现了Aware接口的类，如果你的Bean继承了这些接口并实现了对应的回调方法，那么Spring就会在Bean初始化的相应阶段触发回调方法。刚刚我们提到的在@StreamListener加载流程中被调用postProcessAfterInitialization方法就是在这个阶段执行的
     BeanPostProcessor.postProcessBeforeInitialization：在Bean初始化之前执行的操作
        BeanPostProcessor.postProcessAfterInitialization：在Bean初始化之后执行的操作
       SmartInitializingSingleton.afterSingletonsInstantiated：在所有Bean都完成实例化之后调用
     SmartLifecycle.start：当这一步执行完成以后，就可以认为容器已经成功加载了这个Bean
      SmartLifecycle.stop和DisposableBean.destroy：这两个步骤在应用关闭的时候会执行（Application.close)
## Spring扩展点作用
## Spring IOC AOP 基本原理
	- 动态代理
	- BeanPostProcessor 作用？
	- ApplicationContextAware 的作用和使用？
	- BeanNameAware与BeanFactoryAware的先后顺序？
	- InitializingBean 和 BeanPostProcessor 的after方法先后顺序？
	- ApplicationListener监控的Application事件有哪些？
	- Spring模块装配的概念，比如@EnableScheduling @EnableRetry @EnableAsync，@Import注解的作用？
	- ImportBeanDefinitionRegistrar 扩展点用于做什么事情？
	- ClassPathBeanDefinitionScanner 的作用？
	- NamespaceHandlerSupport 命名空间扩展点的作用？
	- 如何实现动态注入一个Bean？
	- 如何把自定义注解所在的Class 初始化注入到Spring容器？
	- BeanDefinition指的是什么，与BeanDefinitionHolder的区别，Spring如何存储BeanDefinition实例？
	- ASM 与 CGlib 
	- Spring的条件装配，自动装配
