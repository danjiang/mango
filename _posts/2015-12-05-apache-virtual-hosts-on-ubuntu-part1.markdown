---
title: Ubuntu Apache 快速上手指南 - Apache 虚拟主机基础
author: 但江
location: 成都
category: programming
---

通过之前的一系列的文章，你的 apache 服务应该设置完毕而且正常运行，如果你想让一个 Web 服务能够响应多个域名，你就需要为每个域名设置虚拟主机，本文中我们将设置一些示例域名的虚拟主机。

#### 创建你的目录结构

在这个示例中，我们将使用两个域名：domain1.com 和 domain2.com。

在你的用户主目录，创建 public_html 目录：

	cd ~
	mkdir public_html

现在给每一个域名创建一系列标准的子目录：

	mkdir -p public_html/domain1.com/{public,private,log,cgi-bin,backup}

和

	mkdir -p public_html/domain2.com/{public,private,log,cgi-bin,backup}

会给每一个域名（domain1.com 和 domain2.com），创建 public, private, log, cgi-bin 和 backup 目录。

#### index.html

public 目录放置内容，在示例中，我会用一个很简单的 html 页面检测下虚拟主机是否正常工作。

所以给每一个域名创建一个 index.html 文件：

	nano public_html/domain1.com/public/index.html

文件中的内容：

	<html>
	 <head>
	  <title>domain1.com</title>
	 </head>
	 <body>
	  <h1>domain1.com</h1>
	 </body>
	</html>

给 domain2.com 也创建相同的文件，只是把 domain1.com 修改为 domain2.com。

两个域名有了基本的目录结构，现在来看看定义两个虚拟主机。

#### 设置目录权限

想让 apache 能够提供 Web 页面，apache 进程需要有你站点映射目录下所有文件的读取权限，为了保证有我们需要的权限，运行下面两个命令：

sudo chmod -R a+rX ~/public_html
sudo chmod a+rx ~

第一个命令设置 public_html 目录下的所有文件对于所有用户都有读写权限，注意 X 是大写，第二个命令设置你的用户主目录下的所有文件对于所有用户都有读写权限，但是只是第一层，不会深入到更下级目录。

如果你添加了更多的虚拟主机的映射目录，可以再运行第一个命令来保证权限。

#### NameVirtualHost

对于虚拟主机，一个常令人困惑的问题是 NameVirtualHost 的设置。

对于每一个 apache 监听的接口和端口，我们都需要一个 NameVirtualHost 指令，记住每一个端口只需要定义一次。

在 ports.conf 配置文件有默认的 NameVirtualHost 指令配置，运行下面的命令来查看一下：

	cat /etc/apache2/ports.conf

内容如下：

	NameVirtualHost *:80
	Listen 80

	<IfModule mod_ssl.c>
	  # If you add NameVirtualHost *:443 here, you will also have to change
	  # the VirtualHost statement in /etc/apache2/sites-available/default-ssl
	  # to <VirtualHost *:443>
	  # Server Name Indication for SSL named virtual hosts is currently not
	  # supported by MSIE on Windows XP.
	  Listen 443
	</IfModule>

	<IfModule mod_gnutls.c>
	  Listen 443
	</IfModule>

默认的 NameVirtualHost 设置满足我们目前的需求，默认配置会使 apache 执行基于名称的虚拟主机逻辑，使 HTTP 请求在 80 端口上的任何接口都行。

#### 自定义虚拟主机

设置了基础的配置，我们来为 domain1 创建一个虚拟主机的配置文件：

	sudo nano /etc/apache2/sites-available/domain1.com

在你设置自己的域名时候，修改用户名，目录和域名这些的时候，确保你针对这个虚拟主机的配置信息都放在 VirtualHost 块中，否则这些配置信息作用就像放在主配置文件中一样。

一个常见的配置像这样：

	# Place any notes or comments you have here
	# It will make any customization easier to understand in the weeks to come

	# domain: domain1.com
	# public: /home/demo/public_html/domain1.com/

	<VirtualHost *:80>

	  # 管理员邮箱，Server Name 和别名
	  ServerAdmin webmaster@domain1.com
	  ServerName  www.domain1.com
	  ServerAlias domain1.com

	  # 主页文件和映射的根目录，就是我们之前创建的public_html目录
	  DirectoryIndex index.html
	  DocumentRoot /home/demo/public_html/domain1.com/public

	  # 自定义日志文件的输出
	  LogLevel warn
	  ErrorLog  /home/demo/public_html/domain1.com/log/error.log
	  CustomLog /home/demo/public_html/domain1.com/log/access.log combined

	</VirtualHost>

#### a2ensite

现在我们在 site available 有了站点配置信息，运行下面的命令来激活：

	sudo /usr/sbin/a2ensite domain1.com

这个命令的输出是：

	Site domain1.com installed; run /etc/init.d/apache2 reload to enable.

根据提示，重新加载 apache：

	sudo /etc/init.d/apache2 reload

#### 访问测试

为了测试域名的访问而不通过网络上的域名解析，我们可以修改自己电脑上的 host 配置文件来映射 domain1.com 到你服务器的公网 IP，在 Mac 电脑是修改 /etc/hosts 文件，内容如下：

	127.0.0.1    localhost
	...

	# entries related to the demo slice
	123.45.67.890  domain1.com
	123.45.67.890  www.domain1.com
	123.45.67.890  domain2.com
	...

每一个操作系统的 hosts 文件位置有所区别，自己动动手找找吧。

在测试完成后，记得删除掉你自己电脑上配置的域名测试信息。

修改好了后，我们在电脑上通过浏览器来访问下域名：

![Apache Domain1 Com](/images/apache-domain1-com.jpg)

你看到了你自己编写的 index.html 文件的内容，虚拟主机的配置看来没什么问题。

#### ServerAlias

记得我们在虚拟主机的配置中有设置 ServerAlias，你也可以通过这个别名来访问你的站点：

![Apache WWW Domain1 Com](/images/apache-www-domain1-com.jpg)

#### 设置 domain2.com

根据之前的配置来设置 domain2.com：

	sudo nano /etc/apache2/sites-available/domain2.com

激活站点，重启 apache：

	sudo a2ensite domain2.com
	...
	sudo /etc/init.d/apache2 reload

再次在本机修改 hosts，添加 domain2.com 的信息：

	http://domain2.com
	or
	http://www.domain2.com

#### 日志文件

如在虚拟主机中的配置，每一个域名都有自己的日志文件，我们快速看下：

	ls /home/demo/public_html/domain1.com/log/

输出反映了你在虚拟主机中的配置：

	access.log  error.log

保证每一个域名有自己的日志文件，便于以后针对每一个域名站点查找问题。

#### 默认的虚拟主机

记住默认的虚拟主机仍然有效。

现在，如果有人自己输入你服务器的 IP 地址，他会看到默认的虚拟主机呈现的页面，为什么会使用默认的虚拟主机配置呢？

apache 会根据请求的 IP 地址或者域名，来按照字母顺序来查找激活的虚拟主机配置，如果没有找到对应的配置，就会用第一个虚拟主机配置。

如果我们停止或者删除默认的虚拟主机配置，我们就会看到 domain1.com 的内容，因为它在字母顺序上领先于 domain2.com。

在你设置自己的站点的时候，请记住这一点，你就知道如何设置特定的域名为默认的虚拟主机，或者通过 IP 访问的时候，你想呈现什么内容。
