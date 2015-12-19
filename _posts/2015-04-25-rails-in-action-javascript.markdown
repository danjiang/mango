---
title: Rails 实战 - JavaScript
author: 但江
location: 成都
category: programming
tag: rails
---

从 Rails 3.1 开始已经将 jQuery 作为默认的 Javascript 库，本文将会介绍如何结合 jQuery 和 Rails 3.1 来编写 AJAX 应用，结合 jQuery 和 Rails 3.1 关键在于 [jquery_ujs.js][1] 文件，ujs 是 Unobtrusive JavaScript 的缩写，意味着将行为 Javascript 和表现 HTML 分离开，不要写如 `<button onClick="alert('Hello')">Hello</button>` 这样的代码。

#### [jquery_ujs.js][1] 实现 AJAX 的原理

既然不能在 HTML 中直接为 HTML 标签绑定事件，那么将采用如下方式

ERB 文件中

{% highlight ruby %}
<%= link_to "收藏", favoriate_idea_path(idea), :remote => true %>
{% endhighlight %}

HTML 的结果

{% highlight html %}
<a href="/ideas/2/favoriate" data-remote="true">收藏</a>
{% endhighlight %}

在 [jquery_ujs.js][1] 中会通过如下选择器获取需要使用 AJAX 来发送请求的表单或链接

{% highlight javascript %}
$('form[data-remote]')
$('a[data-remote], input[data-remote]')
{% endhighlight %}

处理 AJAX 的响应，通过对表单和链接的回调方法来实现，如下是回调事件

	ajax:beforeSend //发送请求前
	ajax:success //请求成功
	ajax:complete //请求完成
	ajax:error //请求失败

对表单和链接的绑定相应的事件和方法

{% highlight javascript %}
$('#create_comment_form')
.bind("ajax:beforeSend", function(evt, xhr, settings){
    // do something...
})
.bind("ajax:success", function(evt, data, status, xhr){
    // do something...
})
.bind('ajax:complete', function(evt, xhr, status){
    // do something...
})
.bind("ajax:error", function(evt, xhr, status, error){
    // do something...
});
{% endhighlight %}

#### 处理不同响应类型

**纯文本**

Rails Controller 中

{% highlight ruby %}
def handle
    render :text => "OK"
end
{% endhighlight %}

Javascript 文件中

{% highlight javascript %}
bind('ajax:success', function(evt, data, status, xhr){
    alert(xhr.responseText);
});
{% endhighlight %}

**HTML**

Rails Controller 中

{% highlight ruby %}
def handle
 #render handle.html.erb
end
{% endhighlight %}

Javascript 文件中

{% highlight javascript %}
bind('ajax:success', function(evt, data, status, xhr){
    $('#tab-box').html(xhr.responseText);
});
{% endhighlight %}

**JSON**

Rails Controller 中

{% highlight ruby %}
def handle
    render :json => idea.to_json
end
{% endhighlight %}

Javascript 文件中

{% highlight javascript %}
bind('ajax:success', function(evt, data, status, xhr){
 var idea = $.parseJSON(xhr.responseText);
});
{% endhighlight %}

**Javascript**

Rails Controller 中

{% highlight ruby %}
def handle
 #render handle.js.erb
end
{% endhighlight %}

Javascript 文件中

不用做什么，浏览器会执行返回的 Javascript 的代码

#### 用jQuery 直接发送 AJAX 请求

使用 jQuery AJAX 发送正确类型（text, json, javascript...）的请求，Rails 返回正确类型的值即可

#### 参考资料

* [Rails 3 Remote Links and Forms: A Definitive Guide][2]
* [Rails 3 Remote Links and Forms Part 2: Data-type (with jQuery)][3]
* [jQuery API AJAX][4]
* [Layouts and Rendering in Rails][5]
* [Working with JavaScript in Rails][6]

[1]: https://github.com/rails/jquery-ujs
[2]: http://www.alfajango.com/blog/rails-3-remote-links-and-forms/
[3]: http://www.alfajango.com/blog/rails-3-remote-links-and-forms-data-type-with-jquery/
[4]: http://api.jquery.com/jQuery.ajax/
[5]: http://guides.rubyonrails.org/layouts_and_rendering.html
[6]: http://guides.rubyonrails.org/working_with_javascript_in_rails.html
