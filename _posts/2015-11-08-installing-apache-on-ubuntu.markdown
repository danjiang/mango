---
title: 安装 Apache
author: 但江
avatar: danjiang
location: 成都
category: programming
tag: ubuntu-apache
---

Ubuntu 是一个非常流行的 Linux 操作系统，很多虚拟主机服务商都有支持这个操作系统，Apache HTTP Server 也是很流行 Web 服务软件，在 Ubuntu 上安装 Apache HTTP Server 很方便。

![Apache HTTP Server]({{ site.image_base_url }}/apache-http-server.jpg)

## 检测 iptables

这一步是为了保证 Web 服务一旦运行就能通过浏览器访问。

你可能在 Ubuntu 上运行着防火墙，它可能阻断标准的 Web 服务端口，80 端口（常规链接），443 端口（安全链接）。

检测当前防火墙的规则：

{% highlight text %}
sudo /sbin/iptables -L
{% endhighlight %}

如何下规则可以保证严格的配置规则：

{% highlight text %}
-I INPUT -p tcp --dport 80 -m state --state NEW,ESTABLISHED -j ACCEPT
-I OUTPUT -p tcp --sport 80 -m state --state ESTABLISHED -j ACCEPT
-I INPUT -p tcp --dport 443 -m state --state NEW,ESTABLISHED -j ACCEPT
-I OUTPUT -p tcp --sport 443 -m state --state ESTABLISHED -j ACCEPT
{% endhighlight %}

## 安装 Apache

运行下列命令安装 apache

{% highlight text %}
sudo apt-get install apache2
{% endhighlight %}

安装的过程中会自动安装 apache 依赖的一些包。

### Server Name

在安装快结束，启动 apache 进程时，你可能看到如下警告：

{% highlight text %}
apache2: Could not reliably determine the server's fully qualified domain name, using 127.0.0.1 for ServerName
{% endhighlight %}

警告来自于 apache2，因为这是 apache 进程运行时的名称（ps or top 中看到的名称）

创建一个配置文件来存储你的server name：

{% highlight text %}
sudo nano /etc/apache2/conf.d/servername.conf
{% endhighlight %}

文件中输入如下行：

{% highlight text %}
ServerName demo
{% endhighlight %}

用你习惯的表示方法来替代 demo，不要使用你站点的域名，因为在配置虚拟主机时会用到。

修改完毕后，保存 servername.conf。

### Graceful restart

运行如下命令重启 apache 服务，你应该不会再看到警告信息，如果还有请检查前面的步骤。

{% highlight text %}
sudo /usr/sbin/apache2ctl graceful
{% endhighlight %}

## 使用 apache2ctl

apache2ctl 命令可以用来启动，停止 apache，与使用启动脚本（/etc/init.d/apache2）类似，运行 apache2ctl 本身可以所有的参数信息：

{% highlight text %}
$ /usr/sbin/apache2ctl
Usage: /usr/sbin/apache2ctl start|stop|restart|graceful|graceful-stop|configtest|status|fullstatus
       /usr/sbin/apache2ctl <apache2 args>
{% endhighlight %}

### Graceful

{% highlight text %}
graceful|graceful-stop
{% endhighlight %}

apache2ctl 的 graceful 参数会重启 apache 服务，但是不会影响已经存在的连接，graceful-stop 参数与 stop 类似，但是会让 apache 当前的连接完成任务后再停止掉，也就是说 web 服务重启或关闭，进程会继续在旧的配置下处理完已有的连接。

所以使用 graceful 参数在重启 web 服务时，不会打断用户的使用，但是实际应用中，graceful 参数执行不是很完美，有可能会出问题，如果修改了很多配置信息，最好使用 restart 更可靠些。

apache 启动脚本中 reload 参数实际上运行的就是 apache2ctl graceful。

### Configtest

{% highlight text %}
configtest
{% endhighlight %}

configtest 可以在不打断 web 服务情况下，检测配置的信息是否语法错误，但不一定能保证完全没有错误，能够检测常见的错误。

### Status

{% highlight text %}
status|fullstatus
{% endhighlight %}

此命令可以查看当前 web 服务的情况，需要激活 web 服务的 mod_status 模块。

## Apache 日志

默认情况下，apache 日志文件位置：

{% highlight text %}
/var/log/apache2/
{% endhighlight %}

通常有两个日志文件 access.log 和 error.log，access.log存储所有访问 apache 的请求信息，对于通信分析很有用处，error.log 文件存储的是 apache 报告的错误信息，包括模块报告错误信息。

## 测试服务

在浏览器中输入服务器的地址：

{% highlight text %}
http://123.45.67.890
{% endhighlight %}

会出现一段话告诉你 apache 运行成功，如果遇到问题，请查看 apache 是否正常启动，还有就是防火墙端口配置是否正确。
