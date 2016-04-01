---
title: Apache 配置文件
author: 但江
avatar: danjiang
location: 成都
category: programming
tag: ubuntu-apache
---

在这篇文章中，我们将要看一下通过 apt-get 安装的 apache 在 ubuntu 上的配置文件放置位置和简短功能介绍。

![Apache HTTP Server](/images/apache-http-server.jpg)

## apache 配置文件目录

我们看一下 apache 从那里获取配置信息，如果你通过 [Ubuntu Apache 快速上手指南 - 安装 Apache][1] 安装了 apache，你会发现 apache 配置文件的根目录：

{% highlight text %}
/etc/apache2/
{% endhighlight %}

如果你通过源代码来编译 apache，配置文件的位置会有所不同，配置文件的目录位置取决于你安装时指定的，可能在 /usr/local，注意 apache 的文档中描述的配置文件目录是基于源代码安装的方式，本文中是根据 apt-get 安装的方式来描述的配置文件目录。

查看一下配置文件目录中的内容：

{% highlight text %}
$ ls /etc/apache2
{% endhighlight %}

你会看到下面的信息：

![Apache Configuration Files](/images/apache-configuration-files.jpg)

我们会从浏览核心配置文件 apache2.conf 开始。

## 主配置文件

apache 启动读取的第一个配置文件：

{% highlight text %}
/etc/apache2/apache2.conf
{% endhighlight %}

这个文件的注释很完整，你可以浏览一下文件的内容，了解下默认配置是什么样的，你能怎么进行修改，在后面的文章中会描述其中重要的，现在可以忽略掉。

现在需要关注一个重要的指令 Include。

### Include 指令

我们可以把所有的配置内容硬塞进主配置文件，但是每次打开这样一个巨大的文件再做一点修改会显得很笨拙，所以 apache 团队提供了一种添加其他配置文件的方法。

Include 指令 告诉 apache 读取另外一个配置并解析作为配置选项，同样，可以指向一个目录，告诉 apache 读取目录中的所有文件，每次添加一个新的站点，不需要修改巨大的文件。

为什么现在要介绍这个指令？我会指出主配置文件中通过 Include 指令会读取的文件和目录，你会明白附加的配置文件怎么被使用的，你才知道如何放置自己的配置文件。

我们来看看会用到那些附加配置文件

### conf.d

{% highlight text %}
Include /etc/apache2/conf.d/
{% endhighlight %}

conf.d 目录放置的是通常的片段语句，你可以看一下，我们安装 apache 的时候，想设置 ServerName 指令，我们不是修改的主配置文件，而是在 conf.d 下创建了一个新文件来写 ServerName 指令。

当想往 apache 中添加指令而又不知道放在那里时，可以考虑放在 conf.d 下的新建文件中，避免修改主配置文件，使用 conf.d 可以轻松隔离最新配置修改信息（查看修改时间），让你快速找到添加的指令，从而避免把主配置文件改坏了的风险。

### ports.conf

{% highlight text %}
Include /etc/apache2/ports.conf
{% endhighlight %}

ports.conf 文件包含指令告诉 apache 监听 80 端口（默认HTTP通信端口）和 443 端口（SSL支持）。

如果你想 apache 监听其他端口，只需添加更多的 Listen 指令。

### sites-available

默认情况下 sites-available 目录中只有一个文件 default，这个文件包含设置一个默认站点的指令，如文档根目录和目录权限这些。

我们会在后面讨论添加虚拟主机，default 文件对于添加站点到 web 服务是一个好的示例。

你已经注意到我并没有列出指向 sites-available 的 Include 指令，这是因为没有这样的配置，apache 不会查看这个目录，而是另外一个相似的目录...

### sites-enabled

{% highlight text %}
Include /etc/apache2/sites-enabled/
{% endhighlight %}

apache 主配置文件中包含了 sites-enabled 目录，而不是 sites-available 目录，目录名称应该表明了这样做的原因：在 sites-enabled 目录中软链接有指向 sites-available 目录中的配置文件时，一个站点才算激活（可以通过浏览器访问）。

默认情况 sites-enabled 目录中一个软链接指向 sites-available 目录中 default 配置文件，软连接 default 前面的 000 是为了保证这是默认站点，apache 读取配置文件根据文件名的字母顺序，前面加 000 就保证了这是 apache 最先读取的配置文件。

为什么需要默认站点配置？因为，当我们说到虚拟主机时，某人通过 www.example.com 访问你的 web 服务，你有一个针对 www.example.com 站点配置文件，apache 会使用这个配置文件，如果它找不到可以匹配域名的配置文件，它就会使用默认的站点配置。

通过 apache 特定的命令来激活和停止站点配置非常的方便。

### a2ensite 和 a2dissite 命令

a2ensite 和 a2dissite 两个命令控制站点从失效到激活状态之间的转换。

先在浏览器里访问你的默认站点，看到 "It works!"。现在运行命令停止默认站点：

{% highlight text %}
$ sudo /usr/sbin/a2dissite default
{% endhighlight %}

重启apache会重新读取配置文件：

{% highlight text %}
$ sudo /usr/sbin/apache2ctl graceful
{% endhighlight %}

再次访问站点，你会看到 Not found 的错误，因为你停止了默认站点，apache 不知道怎么处理，你可以再看一下 sites-enabled 目录中指向 default 配置的软连接不见了。

重激活默认站点，再次运行 a2ensite 和 reload apache：

{% highlight text %}
$ sudo /usr/sbin/a2ensite default
$ sudo /usr/sbin/apache2ctl graceful
{% endhighlight %}

现在通过浏览器访问站点又可以看到 It works! ，sites-enabled 目录中软连接也再次出现。

a2ensite 和 a2dissite 两个命令来管理站点很有用，同样也可以用来处理站点维护的问题，比如，你可以定义个站点配置来处理站点维护中，访问者看到就是这个取代主站点的页面，只需要停止主站点配置文件，激活维护站点配置文件，当你完成主站点升级的工作，再重新激活主站点配置文件，停止维护站点配置文件，重启加载 apache。

### a2enmod 和 a2dismod 命令

a2enmod 和 a2dismod 命令工作方式同 a2ensite 和 a2dissite 两个命令，只是它们操作的是模块的配置文件，而不是站点的配置文件。

如果，你想激活 userdir 模块（此模块允许 apache 将用户主目录作为站点暴露出来），你可以运行：

{% highlight text %}
$ sudo /usr/sbin/a2enmod userdir
{% endhighlight %}

同样，如果你不想使用 status 模块（可以在页面上查看 apache 服务的状态），你可以这样停止：

{% highlight text %}
$ sudo /usr/sbin/a2dismod status
{% endhighlight %}

如果你改变了模块的加载，记得要重新加载或重启 apache 来使改变有效。

### envvars

文件中包含一些 apache 脚本使用的环境变量，如 apache2ctl，当你改变了 apache 运行的用户和组时才需要修改此文件。

### httpd.conf

{% highlight text %}
Include /etc/apache2/httpd.conf
{% endhighlight %}

如果从源代码编译的 apache，主配置文件就是 httpd.conf，使用 apache2.conf  作为主配置文件是 ubuntu 包维护者做的更人性化的改动，httpd.conf 以防有的应用试图修改默认的 apache 配置文件。

默认情况下 httpd.conf 文件的内容是空的。

[1]: /programming/2015/11/14/apache-configuration-files-on-ubuntu/
