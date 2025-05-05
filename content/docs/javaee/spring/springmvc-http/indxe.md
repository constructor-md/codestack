---
title: SpringMVC 接收 HTTP
draft: false
weight: 6
---

# SpringMVC 接收 HTTP

- SpringMVC 是一个基于 Spring 的 Web 框架，运行于 Tomcat 等 Servlet 容器上
- SpringBoot 内置 Tomcat 容器

当使用原生 Java Servlet 实现 Web 服务器，运行于 Tomcat 上时

1. 用户发起一个 HTTP 请求，Tomcat 收到请求，解析 HTTP 报文
2. Tomcat 根据 URL 找到对应的 Servlet，构建 HttpServletRequest 和 HttpServletResponce 对象，调用 servlet 的 service 方法，将对象引用传入该方法
3. Service 方法获取 Http 请求相关的信息，区分请求类型调用不同的方法，将处理结果放入 Responce 对象。
4. Tomcat 将 Responce 对象封装成 Http 响应报文，返回给客户端。

当使用 SpringMVC 实现 Web 容器，运行于 Tomcat 上时：

1. 用户发起一个 HTTP 请求，Tomcat 收到请求，生成一个线程解析 HTTP 报文
2. Tomcat 生成 HttpRequest 和 HttpResponce 对象，调用 HttpServlet.service 方法，最终将对象转发给 DispatcherServlet.doService 方法（即全流程只有一个 servlet 存在，与 URL 无关）
3. DispatcherServlet 是统一访问点，将请求委托给其他业务处理器。doService 方法调用 doDispatch 方法执行分发
4. 在 Tomcat 初始化时，通知 Spring 初始化容器，SpringMVC 会遍历容器中的 Bean，找到每一个 Controller 的所有方法访问的 url，和 Controller 保存到一个 Map 中。（HandlerMapping 组件获得了 URL 和 Controller 的关系）
5. 分发时 doDispatch 方法根据 URL 找到 Controller，找到 Controller 中对应的方法。将 request 的参数等和方法上的参数根据注解等进行绑定，最后通过反射调用方法。
6. 具体业务逻辑执行完后，回到 DispatcherServlet，进行后续处理，封装视图等
7. Tomcat 将响应对象封装成 HTTP 响应报文，返回给客户端

- 拦截器是在 Spring 中起作用，具体在分发前执行前置拦截器逻辑，分发后执行后置拦截器逻辑
- 过滤器是在 Tomcat 中起作用，具体在进入 servlet 前后进行预处理
- 切面是在具体方法前起作用，具体在调用方法逻辑前后，会处理注解、切点等的逻辑
