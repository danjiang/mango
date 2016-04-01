---
title: Apache 关键配置
author: 但江
avatar: danjiang
location: 成都
category: programming
tag: ubuntu-apache
---

我们已经知道 [如何安装 Apache][1]，以及 [Apache 配置文件的放置位置][2]，还有 [MPM 的设置][3]，本文中我们浏览一些 Apache 常见的关键配置，让 Apache 更安全，更有效率。

![Apache HTTP Server](/images/apache-http-server.jpg)

## Timeout

{% highlight text %}
Timeout 300
{% endhighlight %}

等待请求，执行请求和返回响应，三个步骤总的最大时间（秒），减少这个时间可以降低 DOS 攻击的影响。

## KeepAlive

{% highlight text %}
KeepAlive On
{% endhighlight %}

你应该设置 KeepAlive 的值为 On，它对客户端请求的文件，图片等这些保持一个连接，如果没有 KeepAlive，客户端向服务端请求的用来显示在页面上的每一个元素都需要建立新的连接，保持一个客户端可以重复利用的连接，让服务端处理客户端更加有效率。

其他的 KeepAlive 的配置如下：

## MaxKeepAliveRequests

{% highlight text %}
MaxKeepAliveRequests 100
{% endhighlight %}

现在我们知道了持久连接，这个值可以设置每个连接可以处理的最多请求数量，设置的高，性能就高，你应该自己试验下设置多少合适，如果你的网站有很多图片，JS 脚本这些，把 MaxKeepAliveRequests 调高到 500 试试。

## KeepAliveTimeout

{% highlight text %}
KeepAliveTimeout 15
{% endhighlight %}

一个持久连接等待下一个客户端的请求，最多等待多少时间，默认设置算是比较高的，减少到 2 或 3 秒，如果过了等待的时间，连接就会被杀掉。

优化这个设置的最好办法是自己模拟大多数访问者使用的方式，注意观察你点击连接的频率，如果你发现用户浏览一个页面，需要花一定的时间来进行阅读，比如博客，设置较低的值可以尽快地释放连接在用户访问页面后，可以让出连接给其他访客，因为现在访客需要花写时间来阅读页面，所以不会对现在的用户有什么影响，如果你发现用户短时间内频繁地点击链接，比如在线调查，每页一个问题，你的站点设置 KeepAliveTimeout 值高些对用户的响应更好。

## HostnameLookups

{% highlight text %}
HostnameLookups Off
{% endhighlight %}

如果你希望用户满意，节省流量，把这个设置成 Off。

开启这个选项，可以以通过 DNS 查询获取访客的主机名信息，记录到 apache 的日志中，影响效率，如果你想知道访客的 hostname 信息，建议使用 logresolve (located in /usr/sbin/logresolve)。

## ServerName

默认没有设置

可以查看 [如何安装Apache][1] 中关于 ServerName 的说明。

## AccessFileName

{% highlight text %}
AccessFileName .htaccess
{% endhighlight %}

在以后讲述虚拟主机的时候会深入这个指令，你现在只需要知道 AccessFileName 指定一个文件名的文件可以被添加到服务目录中，文件中的内容可以覆盖主配置文件中的配置，你应该考虑在主配置文件中取消这个功能，以后在特定的虚拟主机配置中启用这个功能。

## ErrorLog

{% highlight text %}
ErrorLog /var/log/apache2/error.log
{% endhighlight %}

apache 默认的放置错误日志的位置，虚拟主机中可以用其他指令来覆盖这个配置。

## Security Settings

安全相关的配置信息在如下文件中，我们在后面关注下 ServerTokens 和 ServerSignature：

{% highlight text %}
/etc/apache2/conf.d/security
{% endhighlight %}

## ServerTokens

默认配置

{% highlight text %}
ServerTokens OS
{% endhighlight %}

ServerTokens 的配置会影响在 HTTP 响应头中显示多少关于 Apache 版本和使用的模块的信息。

默认的配置（OS）会发送下面这样的信息：

{% highlight text %}
Apache/2.2.14 (Ubuntu) Server
{% endhighlight %}

这个信息大多数用户是不关心的，甚至压根都不知道，但是假如有用户想要入侵你的服务器，他应该会比较关心这个信息，刚好他知道你这个 Ubuntu 安装版本的 Apache 的某个漏洞，那不是悲剧了，我们设置下，让他不知道这个信息，安心一点。

可以设置选项和示例输出：

{% highlight text %}
Full —— Apache/2.2.14 (Ubuntu) PHP/5.3.2-1ubuntu4.1 with Suhosin-Patch
OS —— Apache/2.2.14 (Ubuntu) Server
Minimal —— Apache/2.2.14 Server
Minor —— Apache/2.2 Server
Major —— Apache/2 Serve
Prod —— Apache Server
{% endhighlight %}

## ServerSignature

默认设置

{% highlight text %}
ServerSignature On
{% endhighlight %}

apache 服务会自己生成如404和目录浏览的页面，这些页面会包含一个 footer，上面会有一些关于服务器的信息和管理员的联系方式，像下面这样：

![Apache 404](/images/apache-404.png)

有一些浏览器会替换这个默认的错误页面为自己定义的页面，通常会有一个 **View page source** 的选项，你就可以到原来的样子。

ServerSignature 的配置选项：

- Off：没有 footer。
- On：有 footer，会根据 ServerTokens 设置显示相应的信息。
- EMail：添加 email 链接的信息，邮件地址根据虚拟主机中 ServerAdmin 的配置。

注意这些配置信息在虚拟主机的配置中可以被修改，有可能这个虚拟主机相关的访问就会有 footer。

[1]: /programming/2015/11/28/apache-key-configuration-on-ubuntu/
[2]: /programming/2015/11/14/apache-configuration-files-on-ubuntu/
[3]: /programming/2015/11/22/configuring-the-apache-mpm-on-ubuntu/
