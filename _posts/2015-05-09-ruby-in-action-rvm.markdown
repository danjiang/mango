---
title: Ruby 实战 - RVM
author: 但江
location: 成都
category: programming
tag: ruby
---

[RVM][1] 的作用是一个命令行工具，帮助你更轻松地管理 Ruby 环境和 Gem 包，我们需要这样的工具是因为 Ruby 核心团队在不断地更新维护 Ruby，一些优秀 Ruby 框架也在不断地更新维护，但我们在开发的过程中，并不希望不断地更新 Ruby 环境和 Gem 包，从而中断开发的进程，我们希望在准备要更新时才更新，一个现实的例子就是，产品环境中的 Ruby 是 1.9 版本，我们准备将项目升级到使用 Ruby 2.0 版本，但这是需要一段时间才能完成的，所以我们需要在本地开发环境中管理 Ruby 1.9 版本和 Ruby 2.0 版本，这样在升级项目使用 Ruby 2.0 版本过程中，如果产品环境中项目发现了问题，我们还可以在本地继续使用 Ruby 1.9 版本来修复问题，互不影响。 

#### 安装 RVM

	$ \curl -sSL https://get.rvm.io | bash -s stable

#### 管理 Ruby 环境

安装指定版本的 Ruby

	$ rvm install ruby-1.9.3-p547
	$ rvm install ruby-2.0.0-p481

查看本地已安装的 Ruby

	$ rvm list

切换使用不同版本的 Ruby

	$ rvm use ruby-1.9.3-p547

#### 管理 Gemsets

[RVM][1] 不但可以使用不同版本的 Ruby，还可以使用不同的 Gemsets。

创建 Gemsets

	$ rvm 2.1.1
	$ rvm gemset create teddy

查看当前 Ruby 版本下的 Gemsets

	$ rvm gemset list

切换 Gemsets

	$ rvm use 2.1.1@teddy

使用 Gemsets 的一个典型例子就是，一个 Web 应用现在使用是 Rails 4.1，为了使用 Rails 一些新特性，我们需要将项目升级到 Rails 4.2，使用的 Ruby 版本是一样的，在此过程中，既要保证维护线上版本，也要保证升级进程，那就是 Gemsets 发挥作用的时候了。

创建名称为 rails4.1 和 rails4.2 的 Gemsets

	$ rvm ruby-2.0.0-p481
	$ rvm gemset create rails4.1
	$ rvm gemset create rails4.2

利用 [RVM][1] 安装不同版本的 Rails

	$ rvm use ruby-2.0.0-p481@rails4.1
	$ gem install rails -v 4.1.0
	$ rvm use ruby-2.0.0-p481@rails4.2
	$ gem install rails -v 4.2.0

升级过程中配合版本管理工具，就可以很容易地切换环境，同时维护线上版本，以及升级项目。

[1]: http://www.rvm.io
