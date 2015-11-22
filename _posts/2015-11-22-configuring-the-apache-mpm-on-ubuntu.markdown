---
title: Ubuntu Apache 快速上手指南 - Apache MPM
author: 但江
location: 成都
category: programming
---

有一部分 apache 的安装是 MPM，是 Multi-Processing Method（多路处理模块）的简称，MPM 决定 apache 用什么机制来处理多路连接，在 [Ubuntu Apache 快速上手指南 - Apache 配置文件][1] 中，我们已经知道 apache 配置文件的放置位置，本文将会详细阐述 MPM 的详细配置，你会知道如何根据你的环境优化设置。

#### 不同之处

首先，我们需要知道 apache 有几种 MPM 可以使用，但是，主要的 MPM 都是 worker 模型和 prefork 模型。

worker MPM 主要通过子进程创建的新线程来处理连接请求，而 prefork MPM 会创建新的进程来处理连接请求，worker MPM 更有效率，但是有些模块在此模型不太稳定，安装这些模块时会用 prefork MPM 模型来替代 worker MPM 模型，worker MPM 模型比较老，但是兼容性更高。

大多数人都不知道这两种 MPM 之间的性能差异，你得先知道有这个东西，如果你发现你的站点有性能问题，举列来说，你想切换到 worker MPM，尽管你用的有些模块不推荐这么做，如通过 apt-get 安装 PHP 时会自动将 apache 切换到 prefork MPM，现在，新版的 PHP 可以和 worker MPM 支持一起编译，像这样的情况，你就需要查看相关模块的文档来了解它对 MPM 的支持情况。

需要注意一个关键点，apt-get 安装的 apache 默认使用 worker MPM，但是有的模块（包括 PHP）会切换到 prefork MPM。

#### 运行的什么类型？

查看你的 apache 安装使用的那个 MPM，最简单的方法就是查看 aptitude MPM 的安装，运行下面的命令来查看可用的和已安装的 MPM：

	aptitude search apache2-mpm-

输出像这样：

	p  apache2-mpm-event  - Apache HTTP Server - event driven model           
	p  apache2-mpm-itk    - multiuser MPM for Apache 2.2                     
	i A apache2-mpm-prefork - Apache HTTP Server - traditional non-threaded model
	p  apache2-mpm-worker  - Apache HTTP Server - high speed threaded model

你只需要了解第一列的含义，p 说明这是一个可用的包，i 说明这是一个已经安装的包，上列中 apache2-mpm-prefork 前面有 i，说明 apache 安装使用的是 preform MPM。

另外一种更确定的方法，运行如下命令：

	/usr/sbin/apache2ctl -l

会列出编译了的模块：

	Compiled in modules:
	  core.c
	  mod_log_config.c
	  mod_logio.c
	  prefork.c
	  http_core.c
	  mod_so.c

不是所有 apache 使用的模块，一些编译到 apache 基础中的模块，包括 MPM，上例中你看到是 prefork。

#### IfModule

在我们深入每种 MPM 配置之前，我们看下 apache 怎么知道什么配置需要读取，apache 配置文件中的 IfModule 指令用来检测当某个模块被加载时，才使用下面这一段指令。

注意 IfModule 使用的模块名称是在源代码中的名称，对于 SSL 模块的 IfModule 块像这样：

	<IfModule mod_ssl.c>

如果 mod_ssl 是激活的，IfModule 块中的配置都会被读取，所以，你可以放置像 Listen 443 这样的指令。

同样的，你想知道那一部分是那个 MPM 的配置内容，你可以查看主配置文件中 IfModule 相关部分，prefork MPM 配置块，可以从这里找到：

	<IfModule mpm_prefork_module>

包含 worker MPM 的配置块，从这里找到：

	<IfModule mpm_worker_module>

了解了 IfModule 指令，你看不同的 MPM 配置块有相同的指令，不同的值时，不会感到奇怪，这些配置不会冲突，因为 apache 只会加载一个 MPM。

在你以后安装配置自己的模块时，IfModule 很有用处，将指令放置在 IfModule 模块中，即使有的模块没有加载，你的 Web 服务仍然能够正常启动。

#### 配置MPM

为什么在 apache 使用教程中需要这么关注 MPM，MPM 如何工作的，也就说你的服务中 MPM 怎么配置的会影响服务对内存的使用量，近一步会影响你的站点有很多访问时，服务怎么响应，所以要在本文中关注 MPM，如果配置对于你的环境不适合时，它影响的不仅是性能，也会影响服务的稳定性。

MPM 的细节看起来会有些头疼，你可以先整体浏览一下 MPM 的配置信息，在以后建立了虚拟主机后再回头来详细了解。

因为服务器环境情况不同，我们不能提供像__就设置成这样__的信息，我们将查看每种 MPM 的默认配置，强调怎么修改会影响你的服务，配置都很相似，你会注意到描述中有很多重复。

MPM 的默认设置适合与 1GB 内存的服务器，在浏览默认配置时请记住这一点，尤其是 MaxClients 的配置。

#### prefork MPM

默认配置：

	<IfModule mpm_prefork_module>
	  StartServers         5
	  MinSpareServers      5
	  MaxSpareServers      10
	  MaxClients           150
	  MaxRequestsPerChild  0
	</IfModule>

再次强调，MPM 模型会创建一个新的进程来处理连接。

**StartServers**

启动时创建的服务子进程，用来处理进来的连接，如果你能预见很高的流量，你应该增加此值，从而服务一启动就准备好处理大量的连接。

**MinSpareServers**

空闲时保留的最小数量的服务子进程。

**MaxSpareServers**

空闲时保留的最大数量的服务子进程，超过此数量的进程将被杀掉。

**MaxClients**

apache 可以同时处理的最大请求数量，超过此数量的请求将在队列中排队，直到有空闲的进程来处理请求。

MaxClients 不是你能有的最大数量的访问者，而是在一个时刻能够处理的最大数量的请求。

记得 KeepAliveTimeout 设置很小是为了空闲用户的连接可以被尽快回收来处理新用户的连接，每个活动连接都要使用内存，数量逐渐逼近 MaxClients 的总数，如果到达了你设置的最大数量，Web 用户会被卡住等待连接的释放。

你肯定想 MaxClients 足够高，访问者不用等待就能连接到你的网站，但是不能高到 apache 需要获取的内存超过了你服务器的内存，如果超过服务器的内存，它将会到 swap 内存中，非常的缓慢，你肯定不想这样。

对于 prefork MPM，apache 创建一个新的进程来处理一个新的连接，MaxClients 也就意味着 apache 能够创建的最大数量的进程，内存肯定是制约因素。

优化 preform MPM 配置最直接的方式就是在一定访问量情况下，查看每个 apache 进程使用的内存量，计算下你能留给 apache 的内存有多少（也要考虑其他程序需要使用的内存，如 MySQL），然后除以你认为每个 apache 进程需要的内存量，你就能够得到 MaxClients 大概需要的值，如果你不想遇到什么麻烦，256MB 内存的虚拟主机设置 MaxClients 为 40 比较好，从这里逐渐扩展。

**MaxRequestsPerChild**

设置一个子进程在被杀掉前可以处理多少请求，默认值是零，说明永远不会被杀掉。

如果改变默认值为一个有限的数量，在服务不忙的时候，会减少进程的数量，从而释放内存。释放出来的内存能干什么呢？如果其他软件需要内存，在服务有负载的情况下都是需要的，不可能在服务器静悄悄的情况下有什么软件需要内存的。

####worker MPM

默认配置：

	<IfModule mpm_worker_module>
	  StartServers          2
	  MinSpareThreads       25
	  MaxSpareThreads       75
	  ThreadLimit           64
	  ThreadsPerChild       25
	  MaxClients            150
	  MaxRequestsPerChild   0
	</IfModule>

worker MPM 模型中，每一个连接被一个线程处理，被 apache 启动的每一个子进程有很多线程，这个模式可以处理很高的负载量，导致的问题是模块不能建立线程安全。

**StartServers**

同 prefork MPM 的 StartServers。

**MinSpareThreads**

保留的最小数量的线程，如果当前数量的子进程不能够提供足够数量的空闲线程，同时没有超过 MaxClients，一个新的子进行将会被创建来提供更多的空闲线程。

**MaxSpareThreads**

保留的最大数量的线程，超过了这个数量，额外的子进程将会被杀掉，只要保证不低于 MinSpareThreads。

**ThreadLimit**

是所有服务线程总数的硬限制，必须等于或大于 ThreadsPerChild，需要重启来生效配置，你应该先改变 ThreadsPerChild，再来改变这个配置的值。

**ThreadsPerChild**

每个子进程允许建立的线程数，当一个子进程开始，所有的线程也同时开始，意思就是一个子线程所有潜在连接需要的内存都被分配了，直到进程结束才会被释放。

**MaxClients**

同 prefork MPM 的 MaxClients。

**MaxRequestsPerChild**

同 prefork MPM 的 MaxRequestsPerChild。

[1]: /programming/2015/11/14/apache-configuration-files-on-ubuntu/
