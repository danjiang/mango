---
title: HTTP 服务器性能优化
author: 但江
location: 成都
category: programming
---

![Apache HTTP Server](/images/apache-http-server.jpg)

#### 关键指标

做压力测试之前需要知道的总请求数，并发用户数，请求资源类型，由 apache ab 对 apache 的首页进行一次压力测试 `ab -n1000 -c10 http://192.168.5.2/`，结果为:

	Concurrency Level:      10
	Time taken for tests:   3.531 seconds
	Complete requests:      1000
	Failed requests:        0
	Write errors:           0
	Total transferred:      434000 bytes
	HTML transferred:       247000 bytes
	Requests per second:    283.19 [#/sec] (mean)
	Time per request:       35.313 [ms] (mean)
	Time per request:       3.531 [ms] (mean, across all concurrent requests)
	Transfer rate:          120.02 [Kbytes/sec] received

需要关注的信息：

- Requests per second 即为吞吐率 Throughput，指 Web 服务器单位时间内处理的请求数；
- Time per request 即为用户平均请求等待时间，主要用于衡量服务器在一定并发用户数的情况下，对于单个用户的服务质量；
- Time per request ( across all concurrent requests ) 即为服务器平均请求处理时间，用于衡量服务器的整体服务质量；
- Transfer rate 这个统计项可以很好地说明服务器在处理能力达到极限时，其出口带宽的需求量，带宽为数据发送速度。

#### 优化策略

**CPU**

- 系统负载：单位时间内运行队列中就绪等待的进程平均值，`cat /proc/loadavg`；
- 进程切换/上下文切换：尽量减少上下文切换次数，减少进程数；
- IOWait；
- 系统调用：进程从用户态切换到内核态，完成系统调用，应减少不必要的系统调用。

**内存**

- 多进程：使用内存相对多，如 Apache；
- 单进程：使用内存相对少，如 Lighttpd，Nginx。

**I/O 模型**

- 概念：I/O 模型本质区别在于 CPU 的参与方式，同步和异步是指访问数据的机制，同步一般指主动请求并等待 I/O 操作完毕的方式，当数据就绪后在读写的时候必须阻塞，异步则指主动请求数据后便可以继续处理其他任务，随后等待 I/O 操作完毕的通知，这可以使进程在数据读写时也不发生阻塞；
- 多路 I/O 就绪通知：提供了对大量文件描述符就绪检查的高性能方案，它允许进程通过一种方法来同时监视所有文件描述符，并可以快速获得所有就绪的文件描述符，然后只针对这些文件描述符进行数据访问，常见策略有 select，poll，SIGIO，epoll；
- 特别：内存映射，直接 I/O，sendfile

**长连接 Keep-Alive**

一次 TCP 连接中持续发送多份数据而不断开连接。

#### 并发策略

- 一个进程处理一个连接，非阻塞 I/O：Apache Prefork 模式，由主进程预先创建一定数量的子进程；
- 一个进程处理多个连接，非阻塞 I/O：Lighttpd Worker 模式。
