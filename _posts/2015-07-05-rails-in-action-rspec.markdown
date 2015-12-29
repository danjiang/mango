---
title: Rails 实战 - RSpec
author: 但江
avatar: danjiang
location: 成都
category: programming
tag: rails
---

Ruby 社区一直都喜欢测试驱动开发，Rails 框架内置就有测试模块，本文将介绍如何利用 [RSpec][1] 来做单元测试，当然 [RSpec][1] 也可以用来做集成测试。

## Rails 和 RSpec

Rails 项目中使用 RSpec，在 Gemfile 加入如下配置：

{% highlight ruby %}
group :test do
 gem 'rspec-rails', '~> 3.1.0'
 gem 'factory_girl_rails'
end 
{% endhighlight %}

安装 Gems：

{% highlight text %}
$ bundle install
{% endhighlight %}

初始化 spec 目录，spec 目录下会生成文件 spec_helper.rb，此文件将 RSpec 和 Rails 结合起来，运行 RSpec 测试的时候加载 Rails 项目等等这些：

{% highlight text %}
$ rails generate rspec:install
{% endhighlight %}

## Factory Girl

上一步配置 Gemfile 文件的时候，还声明了 factory_girl_rails，[Factory Girl][2] 作用是生成测试需要的数据，在测试很多问题的时候，都需要预先在数据库中填充一些数据，测试完一个单元，还应该清除数据库中的数据，保证下一个单元的测试不被这些遗留数据所影响，我们应该为每一个测试中要用到的 Model，在 spec/factories 编写一个 Factory：

{% highlight ruby %}
# spec/factories/category.rb
FactoryGirl.define do
  factory :category do
    language "en"
    name "A Category"

    factory :food_category do
      is_food true
    end

    factory :exercise_category do
      is_food false
    end
  end
end

# spec/factories/exercise.rb
FactoryGirl.define do
  factory :exercise do
    language "en"
    name "A Exercise"
    association :category, :factory => :exercise_category
  end
end
{% endhighlight %}

## 测试 Models

在 spec/models 中编写针对 Models 的业务逻辑测试。

> 每一个锻炼都有一个分类，当分类下面的锻炼不为零的时候，分类是不能够被删除的。

接着我们来编写测试 spec/models/category_spec.rb 来保证上面这个需求：

{% highlight ruby %}
require "spec_helper"

describe Category, :type => :model do
  it "can not remove category have exercises" do
    category = FactoryGirl.create(:exercise).category
    expect(category.destroy).to be false
    expect(category.errors.full_messages.first).to eql("Cannot destroy category when its exercises not zero")
  end
end
{% endhighlight %}

终端中运行测试，我们还没有实现这样的业务逻辑，肯定会报错：

{% highlight text %}
$ bin/rspec spec/models/category_spec.rb
{% endhighlight %}

通过 before_destory 实现相应的业务逻辑，再次运行上面的测试，就会看到测试通过：

{% highlight ruby %}
class Category < ActiveRecord::Base
  has_many :foods
  has_many :exercises

  validates :name, :language, :presence => true
  validates :name, :length => { :maximum => 23 }

  before_destroy do |category|
    if category.is_food
      unless category.foods.count == 0
        errors[:base] << I18n.t("errors.not_zero", :scope => :admin, :parent => I18n.t("models.category", :scope => :activerecord).downcase, :children => I18n.t("foods", :scope => :admin).downcase)
        false
      end
    else
      unless category.exercises.count == 0
        errors[:base] << I18n.t("errors.not_zero", :scope => :admin, :parent => I18n.t("models.category", :scope => :activerecord).downcase, :children => I18n.t("exercises", :scope => :admin).downcase)
        false
      end
    end
  end
end
{% endhighlight %}

## 测试 Controllers

在 spec/controllers 中编写针对 Controllers 的业务逻辑测试。

> 输入正确的用户名和密码就可以登录到管理页面，否则就不可以

接着我们来编写测试 spec/controllers/sessions_controller_spec.rb 来保证这个上面这个需求：

{% highlight ruby %}
require "spec_helper"

describe SessionsController, :type => :controller do
  context "post create" do
    it "login as admin" do
      admin = FactoryGirl.create(:admin)
      post :create, { :username => admin.username, :password => admin.password }
      expect(response).to redirect_to(admin_root_path)
    end

    it "cannot login as normal user" do
      user = FactoryGirl.create(:user)
      post :create, { :username => user.username, :password => user.password }
      expect(response).to be_success
      expect(response.status).to eq(200)
      expect(flash[:alert]).to eql("Invalid username or password")
    end
  end
end
{% endhighlight %}

终端中运行测试，我们还没有实现这样的业务逻辑，肯定会报错：

{% highlight text %}
$ bin/rspec spec/controllers/sessions_controller_spec.rb
{% endhighlight %}

通过 create 实现相应的业务逻辑，再次运行上面的测试，就会看到测试通过：

{% highlight ruby %}
class SessionsController < ApplicationController
  layout false

  def create
    user = User.find_by_username_and_password_and_admin(params[:username], params[:password], true) 
    if user
      session[:login] = user.id
      redirect_to admin_root_path
    else
      flash.now[:alert] = I18n.t("session.error")
      render :new
    end
  end
end
{% endhighlight %}

[1]: http://rspec.info
[2]: https://github.com/thoughtbot/factory_girl
