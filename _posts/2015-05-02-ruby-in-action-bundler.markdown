---
title: Ruby 实战 - Bundler
author: 但江
avatar: danjiang
location: 成都
category: programming
tag: ruby
---

[Bundler][1] 的作用是为了管理 Ruby 项目中所需要的 Gem 包，确保在开发环境，测试环境和产品环境中 Gem 包的正确安装。

## Gemfile 中声明依赖的 Gem 包

{% highlight ruby %}
source 'https://rubygems.org'

gem 'nokogiri'
gem 'rails', '3.0.0.beta3'
gem 'rack',  '>=1.0'
gem 'thin',  '~>1.1'
{% endhighlight %}

**source** 说明了 Gem 包的安装源。声明依赖的 Gem 包，需要注意版本说明符，**>= 1.0** 的意思很显而易见，通过示例来说明 **~>** 的含义，**~> 2.0.3** 等价于 **>= 2.0.3 和 < 2.1**，**~> 2.1** 等价于 **>= 2.1 和 < 3.0**， **~> 2.2.beta** 会匹配像 **2.2.beta.12** 这样的预发版本。

## 安装 Gem 包

运行 *bundle install* 会看到如下输出：

{% highlight text %}
$ bundle install
Fetching gem metadata from https://rubygems.org/.........
Fetching additional metadata from https://rubygems.org/..
Resolving dependencies...
Using rake 10.3.1
Using json 1.8.1
Installing minitest 5.3.3
Installing i18n 0.6.9
...
Installing rails 4.1.0.rc2
Installing nokogiri 1.6.1
Your bundle is complete!
Use `bundle show [gemname]` to see where a bundled gem is installed.
{% endhighlight %}

Gem 包安装完成后，会在 Gemfile 的同级目录生成一个 Gemfile.lock，其中记录了所有 Gem 包的名称和版本，内容大致如下：

{% highlight ruby %}
GEM
  remote: http://rubygems.org/
  specs:
    actionmailer (4.1.8)
        actionpack (= 4.1.8)
        actionview (= 4.1.8)
        mail (~> 2.5, >= 2.5.4)
    actionpack (4.1.8)
      actionview (= 4.1.8)
      activesupport (= 4.1.8)
      rack (~> 1.5.2)
      rack-test (~> 0.6.2)
    ...

PLATFORMS
  ruby

DEPENDENCIES
  bootstrap-sass (~> 3.3.4)
  cancan
  ...
{% endhighlight %}

Gemfile.lock 文件的内容不需要手动修改，改动 Gemfile 文件，运行 [Bundler][1] 相关命令，Gemfile.lock 文件的内容自然会变化，我们需要将 Gemfile 和 Gemfile.lock 加入到版本库中，其他程序员在获取项目代码，运行 *bundle install* 就会根据 Gemfile.lock 来安装精确版本的 Gem 包，和你的开发环境一模一样。

{% highlight text %}
$ git add Gemfile Gemfile.lock
{% endhighlight %}

## 项目中加载通过 Bundler 安装的 Gem 包

{% highlight ruby %}
require 'rubygems'
require 'bundler/setup'

# require your gems as usual
require 'nokogiri'
{% endhighlight %}

## 运行通过 Bundler 安装的 Gem 包中的执行脚本

{% highlight text %}
$ bundle exec rspec spec/models
{% endhighlight %}

生成执行脚本的快捷方式

{% highlight text %}
$ bundle install --binstubs
$ bin/rspec spec/models
{% endhighlight %}

## Rails 3 中使用 Bundler

Rails 3 已经集成了 [Bundler][1]，你只需要修改 Gemfile，再运行 *bundle install* 就行了，但是有一个小问题需要注意，Rails 项目区分了开发，测试和产品环境，不同的环境中需要的 Gem 包自然是有所不同的，所以需要知道 [Bundler][1] 中 Group 的概念，典型的 Rails 项目中，Gemfile 内容如下：

{% highlight ruby %}
gem 'mysql2'

group :development do
  gem 'capistrano', '~> 3.2.0', :require => false
  gem 'capistrano-rails', '~> 1.1', :require => false
  gem 'capistrano-bundler', '~> 1.1', :require => false
end

group :test do
  gem 'cucumber-rails', :require => false
  gem 'rspec-rails', '~> 3.0.0.beta'
  gem 'factory_girl_rails'
  gem 'selenium-webdriver'
end
{% endhighlight %}

没有写 **group** 的 Gem 包都在 **default group**，在通过 *bundle install* 安装 Gem 包时，我们可以指明不需要那个 **group** 的 Gem 包，命令如下：

{% highlight text %}
$ bundle install --without test development
{% endhighlight %}

项目中加载通过 [Bundler][1] 安装的 Gem 包可以指明需要那个 **group** 的 Gem 包，方法如下：

{% highlight ruby %}
Bundler.require(:default, Rails.env)
{% endhighlight %}

在 Rails 项目中的 application.rb 文件中，你会看到如下内容，你应该明白其中关于 [Bundler][1] 一段代码的含义了吧：

{% highlight ruby %}
require File.expand_path('../boot', __FILE__)

require 'rails/all'

if defined?(Bundler)
  # Require the gems listed in Gemfile, including any gems
  # you've limited to :test, :development, or :production.
  Bundler.require(*Rails.groups)
end

module Kiwi
  class Application < Rails::Application
  ...
  end
end
{% endhighlight %}

[1]: http://bundler.io
