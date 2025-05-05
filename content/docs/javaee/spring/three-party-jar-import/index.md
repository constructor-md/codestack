---
title: SpringBoot 引入第三方 Jar 包
draft: false
weight: 5
---

# SpringBoot 引入第三方 Jar 包
SpringBoot 通过 SPI 机制引入第三方 Jar 包

## SPI 机制
JAVA SPI：Server Provider Interface 服务提供者接口，服务发现机制  
以 JDBC 为例    
JDBC：通常执行流程
1. 加载数据库驱动
2. 通过DriverManager对象，获取Connection连接对象
3. 创建Statement对象执行SQL语句
4. 处理ResultSet结果集
5. 关闭连接

但是其中驱动加载时，Java 仅提供了一个接口，具体的驱动实现类是由各个数据库厂商提供的
- 接口不能被实例化，要想实例化，就必须知道具体驱动类的全限定名
- 即需要知道驱动类的全限定名  
   
Java 开发者想出在项目下的 ClassPath 目录中，创建 META-INF/services 文件夹  
在文件夹内创建**以实现接口全限定名为名**的文件，**内容为实现类的全限定名**  
通过IO获取所有的全限定名，将指定的Class文件实例化存储到容器中，完成第三方的实现类的实例化  
通过java.util.ServiceLoader.load方法实现SPI  
  
   
## SpringBoot 自动装配
SpringBoot的自动装配：  
- SpringBoot 开发中必不可少需要将第三方框架的 Jar 包内容加载到 IOC 容器中
- 但是SpringBoot默认加载启动类所在包或子包的内容，或者指定包扫描路径，指令包扫描路径在很多第三方引入的时候显然太繁琐
- 由于包名未知，不能通过扫描注入
SpringBoot 实现思路：
- SpringBoot规定了要接入SpringBoot的第三方框架，需要在Classpath目录下的META-INF文件中定义spring.factories文件
- 在其中定义需要被加载到IOC容器的类
- SpringBoot启动时会自动扫描ClassPath目录下的所有META-INF的spring.factories文件，读取其中的要注册的类，通过反射进行实例化
    
SPI机制让接口和具体实现类解耦，使得可以根据具体的业务情况启用或替换具体组件