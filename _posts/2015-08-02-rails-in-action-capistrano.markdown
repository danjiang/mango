---
title: Rails 实战 - Capistrano
author: 但江
location: 成都
category: programming
tag: rails
---

[Capistrano][1] 是 Ruby 编写的服务器自动化和部署工具，不是只能部署 Ruby 编写的项目，任何语言编写的项目都可以。

#### 安装

Rails 项目中使用 [Capistrano][1]，在 Gemfile 加入如下配置：

{% highlight ruby %}
gem 'capistrano', '~> 3.4.0'
{% endhighlight %}

安装 Gems：

	$ bundle install

初始化需要的目录和文件：

	$ bundle exec cap install

会生成如下目录和文件：

![Capistrano Rails](/images/capistrano-rails.png)

创建不同 stage 的配置文件

	$ bundle exec cap install STAGES=local,sandbox,qa,production

#### 结构

在服务器上部署好的项目会遵循一定的结构，假设部署到服务器的目录为：

{% highlight ruby %}
set :deploy_to, '/var/www/my_app_name'
{% endhighlight %}

部署的项目会遵循的结构：

![Capistrano Deploy To](/images/capistrano-deploy-to.png)

* current 链接到最近的一次版本发布
* releases 所有的版本发布
* repo 版本控制相关的配置和内容
* revisions.log 日志记录每一次发布和回滚
* shared 每个发布版本之间需要保留的配置和数据

#### 配置

Capfile 文件中加载 [Capistrano][1] 需要的模块，有些模块根据需要配置：

{% highlight ruby %}
# Load DSL and Setup Up Stages
require 'capistrano/setup'

# Includes default deployment tasks
require 'capistrano/deploy'

# Includes tasks from other gems included in your Gemfile
#
# For documentation on these, see for example:
#
#   https://github.com/capistrano/rvm
#   https://github.com/capistrano/rbenv
#   https://github.com/capistrano/chruby
#   https://github.com/capistrano/bundler
#   https://github.com/capistrano/rails
#
# require 'capistrano/rvm'
# require 'capistrano/rbenv'
# require 'capistrano/chruby'
require 'capistrano/bundler'
require 'capistrano/rails/assets'
require 'capistrano/rails/migrations'

# Loads custom tasks from `lib/capistrano/tasks' if you have any defined.
Dir.glob('lib/capistrano/tasks/*.rake').each { |r| import r }
{% endhighlight %}

config/deploy.rb 中是共享的配置信息：

{% highlight ruby %}
# config valid only for Capistrano 3.1
lock '3.2.1'

set :application, 'kiwi'
set :repo_url, 'git@bitbucket.org:danjiang/kiwi.git'

# Default branch is :master
# ask :branch, proc { `git rev-parse --abbrev-ref HEAD`.chomp }.call

# Default deploy_to directory is /var/www/my_app
set :deploy_to, '/home/danjiang/public/kiwi'

# Default value for :scm is :git
# set :scm, :git

# Default value for :format is :pretty
# set :format, :pretty

# Default value for :log_level is :debug
# set :log_level, :debug

# Default value for :pty is false
# set :pty, true

# Default value for :linked_files is []
set :linked_files, %w{config/database.yml config/settings.yml config/application.yml config/mongoid.yml}

# Default value for linked_dirs is []
set :linked_dirs, %w{log tmp/pids tmp/cache tmp/sockets vendor/bundle public/system}

# Default value for default_env is {}
# set :default_env, { path: "/opt/ruby/bin:$PATH" }

# Default value for keep_releases is 5
# set :keep_releases, 5

namespace :deploy do

  desc 'Restart application'
  task :restart do
    on roles(:app), in: :sequence, wait: 5 do
      # Your restart mechanism here, for example:
      execute :mkdir, '-p', "#{ release_path }/tmp"
      execute :touch, release_path.join('tmp/restart.txt')
    end
  end

  desc "Setup shared directory. Upload config examples`s files"
  task :setup_config do
    on roles(:app) do
      execute :mkdir, "-p #{shared_path}/config"
      execute :mkdir, "-p #{shared_path}/config/initializers"
      # Read Local File and Upload
      upload! "config/examples/database.yml", "#{shared_path}/config/database.yml"
      upload! "config/examples/settings.yml", "#{shared_path}/config/settings.yml"
      upload! "config/examples/application.yml", "#{shared_path}/config/application.yml"
      upload! "config/examples/mongoid.yml", "#{shared_path}/config/mongoid.yml"
    end
  end

  after :publishing, :restart

end
{% endhighlight %}

config/deploy/<stage_name>.rb 中是针对每个 stage 的配置信息：

{% highlight ruby %}
server 'danthought.com', user: 'danjiang', roles: %w{web app db}
set :branch, 'master'
{% endhighlight %}

**role** 用于区分不同机器的角色职责

{% highlight ruby %}
server "servername", :some_role_name, :another_role_name
role :some_role_name, "servername"
{% endhighlight %}

**task** 定义具体执行的脚本任务

* namespace 定义一类任务的前缀
* desc 描述，cap -T 会显示
* roles 在什么角色的机器上执行

**before**, **after** 定义了在部署生命周期过程中，在任务执行前后会执行的内容：

{% highlight ruby %}
# call an existing task
before :starting, :ensure_user

after :finishing, :notify


# or define in block
before :starting, :ensure_user do
  #
end

after :finishing, :notify do
  #
end
{% endhighlight %}

#### 使用

	# 查看所有任务
	$ bundle exec cap -T

	# 部署到 staging 环境 
	$ bundle exec cap staging deploy

	# 部署到 production 环境 
	$ bundle exec cap production deploy

[1]: http://capistranorb.com
