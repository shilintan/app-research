springBoot中jetty tomcat undertow对比

1）tomcat优点是稳定性好、可靠性高、支持 Servlet 和 JSP 等标准。构建中小型的 Web 应用程序，可以选择 Tomcat。另外，Spring Boot 默认使用的是 Tomcat
2）Jetty 是一个轻量级的 Web 服务器，它的优点是启动速度快、内存占用小、支持 HTTP/2 和 WebSocket 等新特性。Jetty 适合用于构建高并发、低延迟的 Web 应用程序。社区活跃度高
3）undertow基于 NIO 的 Web 服务器，性能最高、内存占用更小、支持 HTTP/2 和 WebSocket 等新特性。社区活跃度较低Undertow 适合用于构建高并发、低延迟的 Web 应用程序。

如果需要构建中小型的 Web 应用程序，可以选择 Tomcat。
如果需要更多的功能支持和更活跃的社区，可以选择 Jetty。
如果需要更高的性能和更小的内存占用，可以选择 Undertow；

因为编程项目需要用到websocket、对并发要求高，所以采用了undertow。
其配置如下：
server:
  # 内置服务器启动端口，启动多个实例一定要指定(-Dserver.port="xxxx")
  port: 8082
  undertow:
    # 设置IO线程数, 它主要执行非阻塞的任务,它们会负责多个连接, 默认设置每个CPU核心一个线程
    io-threads: 16
    # 阻塞任务线程池, 当执行类似servlet请求阻塞操作, undertow会从这个线程池中取得线程,它的值设置取决于系统的负载
    worker-threads: 1024
这里io-threads和worker-threads是两个线程池的大小，其中io-threads用于处理非阻塞的任务，所以对应的线程个数一般等于cpu核数
worker-thread是用于执行业务逻辑的线程池，例如处理请求、计算响应等。这些操作通常是阻塞的，所以其线程个数一般是根据业务进行确认，如核数*8或者更大。

还不理解，那这里给出实例：
假设有一个 Web 应用程序，它需要处理大量的请求，并且每个请求都需要进行一些计算。为了提高服务器的性能，我们可以使用 Undertow 服务器，并设置两个线程池：server.undertow.io-threads 和 work-threads。

当一个请求到达服务器时，它首先会被 server.undertow.io-threads 中的线程处理。这些线程会读取请求并将其转发给 work-threads 中的线程池。work-threads 中的线程会执行计算任务，并将结果返回给 server.undertow.io-threads 中的线程。最后，server.undertow.io-threads 中的线程将响应写回客户端。

这个过程中，server.undertow.io-threads 中的线程不会被阻塞，因为它们只是负责读取请求和写入响应。而 work-threads 中的线程会被阻塞，因为它们需要执行计算任务。通过这种方式，我们可以充分利用服务器的资源，提高并发处理能力，同时保证每个请求都能够得到及时的响应。

其实io-threads有点类似netty的bossgroup，worker-threads有点类似work group

你们undertow使用的是http什么版本？
使用的是默认的1.1，没有开启2.0，如果需要开启2.0需要增加如下配置：
server.http2.enabled=true