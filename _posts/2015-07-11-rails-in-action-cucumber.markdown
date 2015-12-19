---
title: Rails 实战 - Cucumber
author: 但江
location: 成都
category: programming
tag: rails
---

从上周的文章 [Rails 实战 - RSpec][1] 了解到如何做单元测试，本文将介绍如何利用 [Cucumber][2] 来做行为驱动测试。

#### Rails 和 Cucumber

Rails 项目中使用 [Cucumber][2]，在 Gemfile 加入如下配置：

{% highlight ruby %}
group :test do
  gem 'cucumber-rails', :require => false
  # database_cleaner is not required, but highly recommended
  gem 'database_cleaner'
  gem 'rspec-rails', '~> 3.1.0'
  gem 'factory_girl_rails'
  # Capybara's selenium driver to test javascript
  gem 'selenium-webdriver'
end
{% endhighlight %}

安装 Gems：

	$ bundle install

初始化 features 目录，features 目录下会生成文件 support/env.rb，此文件明确一些配置项：

	$ rails generate cucumber:install

#### [Factory Girl][3]

在文章 [Rails 实战 - RSpec][1] 中编写的 Factories，在这里还是可以使用。

#### 编写 Features

> 创建锻炼，选择分类，填写名称，单位，还可以附加图片

接着我们来编写 features/create_exercise.feature 来保证上面这个需求：

{% highlight ruby %}
Feature: Create exercise
  In order to add exercise
  As an admin
  I want to create it

  Scenario: Create an exercise
    Given there is an exercise category called "Sport"
    And there is a logined admin
    When I click "Exercises"
    And I click "New Exercise"
    And I select "Sport" of "Category" of form
    And I fill in "Name" with "Football" of form
    And I attach file "Image" with "features/resources/bread.png" of form 
    And I fill in "Unit" with "minute" of form
    And I fill in "Calorie" with "10" of form
    And I press "Create Exercise" of form
    Then I should be on the exercise page
    And see a message "Exercise was successfully created."
    And see an exercise "Football"
    And see an unit "minute"
{% endhighlight %}

终端中运行测试，我们还没有编写步骤定义，肯定会报错：

	$ bin/cucumber features/create_exercise.feature

#### 编写 Step Definitions

通过 features/step_definitions 编写步骤定义，不同 Feature 中描述的类似场景可以使用相同的步骤定义：

features/step_definitions/authentication_steps.rb

{% highlight ruby %}
Given(/^I am on the login page$/) do
  visit "/login"
end

Then(/^I should be on the page with "(.*?)"$/) do |name|
  expect(page).to have_content(name)
end

Given(/^there is a logined admin$/) do
  admin = FactoryGirl.create(:admin)
  visit "/login"
  fill_in(:username, :with => admin.username)
  fill_in(:password, :with => admin.password)
  click_button("Login")
end
{% endhighlight %}

features/step_definitions/common_steps.rb

{% highlight ruby %}
When(/^I click "(.*?)"$/) do |link_name|
  click_link(link_name)
end

Then(/^see a message "(.*?)"$/) do |message|
  expect(page).to have_content(message)
end
{% endhighlight %}

features/step_definitions/form_steps.rb

{% highlight ruby %}
When(/^I fill in "(.*?)" with "(.*?)" of form$/) do |name, value|
  fill_in(name, :with => value)
end

When(/^I check "(.*?)" of form$/) do |name|
  check(name)
end

When(/^I uncheck "(.*?)" of form$/) do |name|
  uncheck(name)
end

When(/^I select "(.*?)" of "(.*?)" of form$/) do |value, name|
  select(value, :from => name)
end

When(/^I attach file "(.*?)" with "(.*?)" of form$/) do |name, path|
  attach_file(name, File.expand_path(path))
end

When(/^I press "(.*?)" of form$/) do |name|
  click_button(name)
end
{% endhighlight %}

features/step_definitions/exercise_steps.rb

{% highlight ruby %}
Given(/^there is an exercise called "(.*?)" in this category$/) do |exercise_name|
  category = Category.last
  FactoryGirl.create(:exercise, :name => exercise_name, :category => category)
end

Then(/^I should be on the exercises list page$/) do
  expect(current_path).to eq(admin_exercises_path)
end

Then(/^I should be on the exercise page$/) do
  exercise = Exercise.first
  expect(current_path).to eq(admin_exercise_path(exercise))
end

Then(/^see an exercise "(.*?)"$/) do |exercise_name|
  expect(page).to have_content(exercise_name)
end

Then(/^not see an exercise "(.*?)"$/) do |exercise_name|
  expect(page).not_to have_content(exercise_name)
end
{% endhighlight %}

在 Rails 项目中实现相应的业务逻辑，终端中再次运行测试，就会看到测试通过：

	$ bin/cucumber features/create_exercise.feature

#### [Capybara][4]

在上面编写测试步骤的过程中，可以看到 visit, fill_in, click_button, click_link 和 attach_file 这些方法都是由 [Capybara][4] 提供的，用来执行浏览器中的操作，如访问链接，填写表单，提交表单等。

#### 含有 JavaScript 操作的测试

在 Gemfile 中包含有 selenium-webdriver，在运行包含有 JavaScript 操作的测试时，就会启动浏览器进行相应的测试，你需要做的就是在 Feature 中添加一个标记，如下：

{% highlight ruby %}
@javascript
Feature: Tag foods
...
{% endhighlight %}

[1]: /programming/2015/07/05/rails-in-action-rspec/
[2]: https://cucumber.io
[3]: https://github.com/thoughtbot/factory_girl
[4]: https://github.com/jnicklas/capybara
