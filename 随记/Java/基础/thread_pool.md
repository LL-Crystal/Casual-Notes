# Spring Boot默认线程池

在 Spring Boot 中，默认情况下会使用一些线程池来处理不同类型的任务，默认的线程池都有哪些呢

1、Tomcat 线程池：
>Spring Boot 使用嵌入式的 Tomcat 作为默认的 Web 服务器。
Tomcat 的默认线程池最大线程数为 200。如果你的 Web 应用同时接收超过 200 个 HTTP 请求，一些请求将被放入队列等待可用线程
---
2、调度线程池：
>Spring Boot 中的调度任务（例如定时任务）使用一个默认的线程池来执行。
默认情况下，调度线程池的大小为 1。你可以在 application.properties 中通过修改 spring.task.scheduling.pool.size 属性来设置线程池大小。例如，spring.task.scheduling.pool.size=20 将设置线程池大小为 20
---
3、客户端请求线程池：
>如果你使用的是嵌入式的 Tomcat，Spring Boot 提供了一个属性来控制客户端请求线程池的大小。
默认情况下，该属性的值为 0，这将使用 Tomcat 的默认值（200）作为客户端请求线程池的大小。如果你使用的是 Spring Boot 2.3 或更高版本，该属性的名称是 server.tomcat.threads.max
---

Tomcat 线程池和客户端请求线程池在 Spring Boot 应用中是不同的线程池,区别在于：

1、Tomcat 线程池
>Tomcat 是 Spring Boot 默认的嵌入式 Web 服务器。
Tomcat 线程池用于处理传入的 HTTP 请求。它负责处理客户端发起的连接、接收请求、处理请求、发送响应等操作。
---
2、客户端请求线程池
>客户端请求线程池用于处理从应用程序到外部服务的客户端请求，例如向其他服务发起的 HTTP 请求、数据库查询等。
在 Spring Boot 中，你可以通过配置 server.tomcat.threads.max 属性来设置客户端请求线程池的大小。默认情况下，该属性的值为 0，这将使用 Tomcat 线程池的默认值（200）作为客户端请求线程池的大小。
---

如果是网关应用，可能还有其他线程池

1、如果是Spring cloud gateway
Netty 线程池：
>Spring Cloud Gateway 使用 Netty 来处理请求。
默认情况下，Netty 的工作线程数（即处理业务逻辑的线程数）等于 CPU 的核数。
您可以通过配置参数来调整线程池的大小，例如 server.netty.workerThreads。
---
Hystrix 线程池：
>如果您在 Spring Cloud Gateway 中使用 Hystrix 进行熔断和隔离，每个不同的外部请求 URL 都会使用一个独立的 Hystrix 线程池。
> 您可以为每个 URL 配置不同的线程池参数，或者使用默认线程池配置。
---
@Async 线程池：
>如果您在 Spring Cloud Gateway 中使用 @Async 注解来执行异步方法，Spring 默认会搜索与之关联的线程池定义。
> 它会查找唯一的 org.springframework.core.task.TaskExecutor bean 或名为 "taskExecutor" 的 java.util.concurrent.Executor bean。
---

2、如果是Spring WebFlux网关
Netty 线程池：
>如果您在 Netty 上运行 Spring WebFlux，它使用的是 Netty 的线程池。
具体来说，WebFlux 使用名为 reactor-http 的线程池来处理外部请求
---

Reactor 线程池：
>WebFlux 内部使用 Reactor 库，基于异步和事件驱动的响应式编程。
> 默认情况下，它使用的是 Reactor 的线程池。
> 您可以通过配置参数来调整线程池的大小，例如 react.netty.ioWorkerCount（即 reactor-http 线程数量）
---