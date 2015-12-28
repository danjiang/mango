---
title: Ruby 实战 - RubyGems
author: 但江
avatar: danjiang
location: 成都
category: programming
tag: ruby
---

RubyGems 是 Ruby 标准的打包，分享代码的方式，[RubyGems.org][1] 为 Ruby 社区提供了 Gem 托管服务，国内网络的原因，你可以使用淘宝提供的 [RubyGems 国内镜像][2]，本文会介绍使用 RubyGems 的命令来管理 Gem，还会做一个简单的 Gem 来发布

## RubyGems 命令管理 Gem

只要是安装 Ruby 1.9 及以上都是同时已经安装了 RubyGems

查找 Gems

{% highlight text %}
$ gem search ^rails

*** REMOTE GEMS ***

rails (4.0.0)
rails-3-settings (0.1.1)
rails-action-args (0.1.1)
rails-admin (0.0.0)
rails-ajax (0.2.0.20130731)
[...]
{% endhighlight %}

安装 Gems

{% highlight text %}
$ gem install drip
Fetching: rbtree-0.4.1.gem (100%)
Building native extensions.  This could take a while...
Successfully installed rbtree-0.4.1
Fetching: drip-0.0.2.gem (100%)
Successfully installed drip-0.0.2
Parsing documentation for rbtree-0.4.1
Installing ri documentation for rbtree-0.4.1
Parsing documentation for drip-0.0.2
Installing ri documentation for drip-0.0.2
Done installing documentation for rbtree, drip after 0 seconds
2 gems installed
{% endhighlight %}

如果安装 Gems 的时候不想安装 ri 和 rdoc 文档，在用户 home 目录创建 .gemrc 加入如下内容

{% highlight text %}
gem: --no-ri --no-rdoc
{% endhighlight %}

添加 Gem 到 Ruby 的加载路径，默认的加载路径如下

{% highlight text %}
% irb -rpp
>> pp $LOAD_PATH
[".../lib/ruby/site_ruby/1.9.1",
 ".../lib/ruby/site_ruby",
 ".../lib/ruby/vendor_ruby/1.9.1",
 ".../lib/ruby/vendor_ruby",
 ".../lib/ruby/1.9.1",
 "."]
{% endhighlight %}

通过 require 添加 Gem 到 Ruby 的加载路径

{% highlight text %}
% irb -rpp
>> require 'ap'
=> true
>> pp $LOAD_PATH.first
".../gems/awesome_print-1.0.2/lib"
{% endhighlight %}

列出已经安装的 Gem

{% highlight text %}
$ gem list

*** LOCAL GEMS ***

bigdecimal (1.2.0)
drip (0.0.2)
io-console (0.4.2)
json (1.7.7)
minitest (4.3.2)
psych (2.0.0)
rake (0.9.6)
rbtree (0.4.1)
rdoc (4.0.0)
test-unit (2.0.0.0)
{% endhighlight %}

删除 Gems

{% highlight text %}
$ gem uninstall drip
Successfully uninstalled drip-0.0.2
{% endhighlight %}

## 制作自己的 Gem

基本的目录结构，lib 中放代码，通常会有一个和 Gem 同名的 Ruby 文件，当 Gem 被安装，通过 require 来加载，这个 Ruby 文件就负责设置好你所有的代码

{% highlight text %}
% tree
.
├── hola.gemspec
└── lib
    └── hola.rb
{% endhighlight %}

lib/hola.rb 中的代码

{% highlight text %}
% cat lib/hola.rb
{% endhighlight %}

{% highlight ruby %}
class Hola
  def self.hi
    puts "Hello world!"
  end
end
{% endhighlight %}

gemspec 描述了 Gem 的基础信息

{% highlight text %}
% cat hola.gemspec
{% endhighlight %}

{% highlight ruby %}
Gem::Specification.new do |s|
  s.name        = 'hola'
  s.version     = '0.0.0'
  s.date        = '2010-04-28'
  s.summary     = "Hola!"
  s.description = "A simple hello world gem"
  s.authors     = ["Nick Quaranto"]
  s.email       = 'nick@quaran.to'
  s.files       = ["lib/hola.rb"]
  s.homepage    = 'http://rubygems.org/gems/hola'
  s.license     = 'MIT'
end
{% endhighlight %}

生成 Gem，在本地安装来测试一下

{% highlight text %}
% gem build hola.gemspec
Successfully built RubyGem
Name: hola
Version: 0.0.0
File: hola-0.0.0.gem

% gem install ./hola-0.0.0.gem
Successfully installed hola-0.0.0
1 gem installed
{% endhighlight %}

你可以在 [RubyGems.org][1] 注册一个帐号来发布 Gem，在你电脑上设置你的 RubyGems 帐号

{% highlight text %}
$ curl -u qrush https://rubygems.org/api/v1/api_key.yaml >
~/.gem/credentials; chmod 0600 ~/.gem/credentials

Enter host password for user 'qrush':
{% endhighlight %}

设置好帐号后就可以发布了

{% highlight text %}
% gem push hola-0.0.0.gem
Pushing gem to RubyGems.org...
Successfully registered gem: hola (0.0.0)
{% endhighlight %}

准备包含更多的文件，目录结构如下

{% highlight text %}
% tree
.
├── hola.gemspec
└── lib
    ├── hola
    │   └── translator.rb
    └── hola.rb
{% endhighlight %}

lib/hola/translator.rb 中的内容

{% highlight text %}
% cat lib/hola/translator.rb
{% endhighlight %}

{% highlight ruby %}
class Translator
  def initialize(language)
    @language = language
  end

  def hi
    case @language
    when "spanish"
      "hola mundo"
    else
      "hello world"
    end
  end
end
{% endhighlight %}

lib/hola.rb 中的内容，加载了 Translator 代码 

{% highlight text %}
% cat lib/hola.rb
{% endhighlight %}

{% highlight ruby %}
class Hola
  def self.hi(language = "english")
    translator = Translator.new(language)
    translator.hi
  end
end

require 'hola/translator'
{% endhighlight %}

修改 gemspec 包含这两个 Ruby 文件

{% highlight text %}
% cat hola.gemspec
{% endhighlight %}

{% highlight ruby %}
Gem::Specification.new do |s|
...
s.files = ["lib/hola.rb", "lib/hola/translator.rb"]
...
end
{% endhighlight %}

[1]: https://rubygems.org
[2]: http://ruby.taobao.org
