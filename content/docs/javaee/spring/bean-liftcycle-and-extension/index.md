---
title: Bean 生命周期与 SpringBoot 扩展点 
draft: false
weight: 4
---


# Bean 生命周期与 SpringBoot 扩展点 
生命周期：   
    从对象的创建到销毁的过程   

从 SpringBootApplication.run 方法出发，执行到 AbstractApplication.refresh 方法   
从此处开始进行 BeanFactory 等资源的准备、XML/注解的扫描   
1. Spring 创建 BeanFactory ，工厂扫描 XML、Java 注解等，生成 BeanDefinition 对象
2. 调用工厂方法根据 BeanDefinition 对象通过反射生成 Bean 实例，完成实例化
3. Spring 将值和 Bean 引用注入到 Bean 对应属性
4. 如果 Bean 实现了 BeanNameAware 接口，Spring 将 Bean 的 ID 传递给 setBeanName 方法
5. 如果 Bean 实现了 BeanFactoryAware 接口，Spring 调用 setBeanFactory 将 BeanFactory 实例传入
6. 如果 Bean 实现了 ApplicationContextAware 接口，Spring 调用 setApplicationAware 将应用上下文传入
7. 如果 Bean 实现了 BeanPostProcessor 接口，Spring 调用 postProcessBeforeInitialization 方法
8. 如果 Bean 中有 @PostConstruct 方法，执行该方法
9. 如果 Bean 实现了 InitilizingBean 接口，Spring 调用 afterPropertiesSet 方法；如果 @Bean 声明了 initMethod，Spring 再调用该方法
10. 如果 Bean 实现了 BeanPostProcessor 接口，Spring 调用 postProcessorAfterInitialization 方法
11. 此时 Bean 的属性已经设置和前后置操作完毕，完成了初始化。Bean 将一直存在于应用上下文中，直到应用上下文被销毁
12. 如果 Bean 中有 @PreDestory 方法，执行该方法
13. 如果 Bean 实现了 DisposableBean 接口，Spring 将调用 destory 方法。如果 @Bean 声明了 destoryMethod，Spring 再调用该方法。

