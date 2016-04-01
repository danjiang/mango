---
title: Apache 虚拟主机配置详解
author: 但江
avatar: danjiang
location: 成都
category: programming
tag: ubuntu-apache
---

我们知道虚拟主机的基本配置方法，本文将会深入了解一些虚拟主机的配置项，有些配置选项在前面的文章有出现过，有的没有出现过，需要注意一点，有些指令在 apache 主配置文件和虚拟主机的配置中都可以设置，如果两个文件有同样的配置，VirtualHost 配置块中的配置会起作用，你应该花时间来通读这些配置的阐述，你才能明白虚拟主机的功能强大。

![Apache HTTP Server](/images/apache-http-server.jpg)

## ServerAdmin

{% highlight apache %}
ServerAdmin webmaster@domain.com
{% endhighlight %}

设置服务器管理员的邮箱地址，如果你设置 ServerSignature 为 Email，这个邮箱会现在服务器的错误页面。

## ServerName 和 ServerAlias

{% highlight apache %}
ServerName www.domain.com
ServerAlias domain.com
{% endhighlight %}

ServerName 设置虚拟主机的域名，还可以通过 ServerAlias 设置很多别名的域名地址，比如，你可以设置 domain.com 和 domain.net 指向相同的内容。

注意这不是 rewrite 规则（通过 rewrite 模块来控制，本文不会涉及到），但是这里设置的域名都会呈现相同的内容（假设你已设置了域名的 DNS 指向你的服务器IP地址）。

## DirectoryIndex

{% highlight apache %}
DirectoryIndex index.html
{% endhighlight %}

定义主页的文件，如果访客没有输入特定访问的文件，服务就会使用这个文件，当你想用一个文件来替代默认文件，或者用一个非标准的主页文件（如 index.php）时，这个配置就很有用，你可以设置多个文件，apache 会用它找的第一个文件：

{% highlight apache %}
DirectoryIndex index.php index.html
{% endhighlight %}

访客直接输入域名的时候，就会检测这个指令的设置，但访客可以直接输入文件名称来访问（如 http://www.example.com/index.html）

## DocumentRoot

{% highlight apache %}
DocumentRoot /home/demo/public_html/domain.com/public
{% endhighlight %}

域名映射的目录地址，使用绝对路径。

如果访问者只是通过域名地址来访问，服务会查看 DocumentRoot 的配置来决定到那里去查找文件，然后，查看 DirectoryIndex 的配置来决定使用那个文件来显示给访问者。

当一个 URL 有指定子目录或文件时，服务查找文件的地址会相对于 DocumentRoot 的配置，比如下面的 URL：

{% highlight http %}
http://www.example.com/main/sub/waffles.html
{% endhighlight %}

服务会查找如下地址：

{% highlight text %}
{虚拟主机中DocumentRoot的配置}/main/sub/waffles.html
{% endhighlight %}

## ErrorLog 和 CustomLog

{% highlight apache %}
LogLevel warn
ErrorLog  /home/demo/public_html/domain.com/log/error.log
CustomLog /home/demo/public_html/domain.com/log/access.log combined
{% endhighlight %}

设置日志的等级和虚拟主机日志文件的位置，对于域名统计分析非常有用，ErrorLog 指明 apache 输出当前虚拟主机错误日志的位置，内容由 LogLevel 指定，CustomLog 指明 apache 输出其他日志的位置。

combined 的日志类型满足大部分的需要，如果查看主配置文件，你会发现其他 custom 的日志类型，不同日志类型会影响 apache 记录访问者日志的方式。

## ErrorDocument

{% highlight apache %}
ErrorDocument 404 /errors/404.html
ErrorDocument 403 /errors/403.html
{% endhighlight %}

用于标准的错误页面。

在这个示例中，虚拟主机的 public 目录下有 errors 目录，在 errors 目录下创建了相应的错误显示文件，配置中设置的是基于 DocumentRoot 的相对路径。

如果没有设置，apache 会使用自己生成的错误页面，当有错误发生时，自定义的错误页面可以提供一直的用户体验，可以展现任何信息给访问者（比如到站点地图的链接，或者返回主页的方式）。

## ServerSignature

{% highlight apache %}
ServerSignature On
{% endhighlight %}

主配置也有此选项，这里的配置项只针对当前虚拟主机有效，你可以查看 [Apache 关键配置][1]，了解这个配置的作用。

## ScriptAlias 和 cgi-bin

{% highlight apache %}
ScriptAlias /cgi-bin/ /home/demo/public_html/domain.com/cgi-bin/
<Location /cgi-bin>
  Options +ExecCGI
</Location>
{% endhighlight %}

激活 cgi-bin 的位置。

## Directory

{% highlight apache %}
<Directory /home/demo/public_html/domain.com/public>
  Options FollowSymLinks
</Directory>
{% endhighlight %}

为特定的目录设置选项，上例中为 domain.com 的 public 目录开启 FollowSymLinks。

在 Directory 块中可以通过 Options 来激活或失效一些功能特性，通过前置 **-** 显示地失效功能，通过前置 **+** 显示地或不加前置修饰来激活功能。

### Indexes

{% highlight apache %}
Options -Indexes
{% endhighlight %}

使用 -Indexes 或 None 关闭目录浏览，使用 +Indexes 激活，目录浏览在没有发现设置的 DirectoryIndex 时，会呈现目录中的文件名列表给用户。

### Includes

{% highlight apache %}
Options -Includes
{% endhighlight %}

失效或激活服务端的 Includes，如果你不知道它是什么，那就不要激活。

### FollowSymLinks 和 SymLinksIfOwnerMatch

{% highlight apache %}
Options -FollowSymLinks
{% endhighlight %}

激活或失效此选型决定当查找服务文件时是否准许 symlink，注意此选项可能导致安全问题，比如不小心连接到了配置目录。

你可以考虑使用 SymLinksIfOwnerMatch 指令来替代 FollowSymLinks，SymLinksIfOwnerMatch 只允许当连接和目标文件的 Linux 文件访问权限一直时才能连接，这个方案可以过减少一些安全隐患。

### AllowOverride

{% highlight apache %}
AllowOverride None
{% endhighlight %}

AllowOverride 设置为 none 可以失效 .htaccess 支持，设置为 All 可以激活它们，.htaccess 文件可以在主配置文件外配置 apache 的控制指令，如果你想让有的用户修改配置而又不想让他修改主配置文件，它通常用于密码保护的目录。

你可以像如下指定需要使用那些 .htaccess 的特性：

{% highlight apache %}
AllowOverride AuthConfig Indexes
{% endhighlight %}

Apache [htaccess][2] 和 [AllowOverride][3] 文档有更多信息。

保护 .htaccess文件，避免通过浏览器直接查看是个好主意，可以如下配置：

{% highlight apache %}
AccessFileName .myobscurefilename
<Files ~ "^\.my">
  Order allow,deny
  Deny from all
  Satisfy All
</Files>
{% endhighlight %}

### None

{% highlight apache %}
Options None
{% endhighlight %}

这会关闭所有可用的选项，包括被父目录默认激活的选择。

## 层级

Options 指令可以根据每一个目录设置：

{% highlight apache %}
<Directory />
  AllowOverride None
  Options None
</Directory>

<Directory /home/demo/public_html/domain.com/public>
  AllowOverride All
</directory>
{% endhighlight %}

第一个 Directory 块中关闭了所有的 Options，对于所有的目录失效了 .htaccess。

然后，第二个 Directory 配置会覆盖第一个的配置，从而激活了 domain.com/public 目录下的 .htaccess 支持。

[1]: /programming/2015/11/28/apache-key-configuration-on-ubuntu/ 
[2]: http://httpd.apache.org/docs/2.2/howto/htaccess.html
[3]: http://httpd.apache.org/docs/2.2/mod/core.html#allowoverride
