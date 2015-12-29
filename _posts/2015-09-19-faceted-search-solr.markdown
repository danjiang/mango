---
title: Solr 构建分面搜索
author: 但江
avatar: danjiang
location: 成都
category: programming
---

分面搜索（Faceted Search）已经成为关键的特性，可以加强各种搜索应用的查找可能性和用户搜索体验，在这篇文章中会介绍用 [Solr][1] 来构建分面搜索。

## 什么是分面搜索？

分面搜索是动态分类归并条码或者搜索结果为类别，让用户可以通过任何字段的任何值深入搜索结果（甚至整个跳过搜索），每一个显示的分面同样会显示搜索符合此分类的数量，然后用户可以深入特定约束的搜索结果集，分面搜索也叫分面浏览，分面导航，指南导航，或者参数搜索。

通过 [CNET][2] 做示例可以容易理解什么是分面搜索，[CNET][2] 是第一个使用 [Solr][1] 的网站，早在 [Solr][1] 被贡献给 Apache。

![Solr CNET]({{ site.image_base_url }}/solr-cnet.jpg)

这个示例是分面浏览，因为从 digital cameras 类目开始而不是用户搜索，Manufactuter 是一个搜索结果的分面，此分面的值包括 Canon USA，Sony 和 Nikon，在这之前的一个页面还包括 Price 和 Digital camera type 分面，因为我们已经从这些分面中选择了的 Price 为 $400 - $500 和 Digital camera type 为 SLR 的约束，所以不在显示，对应这些约束，结果集数量和相机列表只显示了价格在 $400 - $500 的数码 SLR 相机，任何显示的分面约束都可以被点击，进一步过滤搜索结果，已经应用的约束可以从面包屑中取消。

分面搜索提供了一种允许用户精练搜索结果的有效方式，持续深入直到找到理想的条目，包括以下好处：

- 优越的反馈，用户可以一眼看到搜索结果的统计，在不同条件的下有多少搜索结果。

- 没有死链接，用户在点击前就是知道有多少个结果集，统计结果为零约束已经被移除，从而排除了用户点击会导致零结果的约束。

- 没有强制的选择层级，用户可以按任意顺序随意地添加和删除约束。

## 通过 [Solr][1] 实现分面搜索

从 [Solr][1] 获取分面信息相对简单，这里有一些预备知识，[Solr][1] 提供如下的类型分面搜索，都可以在不需要预先配置情况下请求：

- Field faceting - 获取某字段所有词条的数量或者顶级词条，字段必须是建立索引的。
- Query faceting - 从搜索结果集中统计出符合查询条件的结果数量。
- Date faceting - 从搜索结果集中统计符合日期区间的结果数量。

分面搜索的命令添加在 [Solr][1] 常规搜索请求上，分面结果同样也在搜索响应中，如果你不熟悉 [Solr][1] 搜索请求的细节，[查看指南](http://lucene.apache.org/solr/resources.html#tutorials)。

## 实现 Manufacturer 分面搜索

为实现 Manufacturer 分面搜索 ，我向 [Solr][1] 发送了 field faceting 命令，这个示例假定有 manu 字段在 schema 中存在而且被作为一个 token 来索引，Solr schema 中 string 满足这个需求。

假定用户在搜索框中输入了 camera，对应的 [Solr][1] 查询语句如下：

{% highlight http %}
localhost:8983/solr/select?q=camera
{% endhighlight %}

要获取 manu 字段分面搜索的数量，在查询语句中添加如下参数：

{% highlight http %}
&facet=true&facet.field=manu
{% endhighlight %}

在查询语句可以添加任何数量的分面搜索命令，为了分面 manu 和 camera_type 两个字段，添加如下参数：

{% highlight http %}
&facet=true&facet.field=manu&facet.field=camera_type
{% endhighlight %}

查询结果里包含的分面统计信息

{% highlight xml %}
<lst name="facet_fields">
 <lst name="manu">
  <int name="Canon USA">17</int>
  <int name="Olympus">12</int>
  <int name="Sony">12</int>
  <int name="Panasonic">9</int>
  <int name="Nikon">4</int>
 </lst>
 <lst name="camera_type">
  <int name="Compact">17</int>
  <int name="Ultracompact">11</int>
  <int name="SLR">17</int>
  <int name="Full body">9</int>
 </lst>
</lst>
{% endhighlight %}

分面统计结果都是在当前查询环境下，如，在搜索索引中 Canon 制造的相机有 100 个，但是只有 17 个满足目前的搜索；现在由表现层来决定以包含数量的可点击链接的形式，将这些信息展现给用户。

## 实现 Price 分面搜索

如果在 price 字段上请求 field faceting，获取到的是每个价格的统计数量，我们需要价格区间，而不是每个独立的价格，一种方法是建立如 100_200, 200_300, 300_400 这样的索引字段，然后对这些字段使用 field faceting，更灵活的方式是使用 query facets。

假定我们有一个 price 的索引字段，我们想获取这些价格区间（小于 $100, $100-$200, $200-$300, $300-$400, $400-$500, 大于 $500）的分面搜索数量，我们需要在搜索请求中为想要的价格区间添加 facet.query 命令：

{% highlight http %}
&facet=true&facet.query=price:[* TO 100]
&facet.query=price:[100 TO 200]&facet.query=[price:200 TO 300]
&facet.query=price:[300 TO 400]&facet.query=[price:400 TO 500]
&facet.query=price:[500 TO *]
{% endhighlight %}

查询结果里包含的分面查询的统计信息

{% highlight xml %}
<lst name="facet_queries">
 <int name="price:[* TO 100]">28</int>
 <int name="price:[100 TO 200]">54</int>
 <int name="price:[200 TO 300]">98</int>
 <int name="price:[300 TO 400]">84</int>
 <int name="price:[400 TO 500]">73</int>
 <int name="price:[500 TO *]">56</int>
</lst>
{% endhighlight %}

## 应用约束做为搜索过滤条件

现在，我们知道如何进行分面搜索获取统计信息，我们该如何让用户深入而进一步用约束来缩小搜索结果，答案是利用 [Solr][1] 标准过滤查询，搜索结果可以被任意数量的过滤查询过滤。

假定用户在搜索框中输入了 camera，我们请求搜索返回了最匹配的结果和分面统计数量显示给用户：

{% highlight http %}
localhost:8983/solr/select?q=camera
&facet=on&facet.field=manu&facet.field=camera_type
&facet.query=price:[* TO 100]
&facet.query=price:[100 TO 200]&facet.query=[price:200 TO 300]
&facet.query=price:[300 TO 400]&facet.query=[price:400 TO 500]
&facet.query=price:[500 TO *]
{% endhighlight %}

现在，假定用户想通过 $400 - $500 价格区间的约束进一步深入，用户将会得到只包含此价格区间的搜索结果，因此我们需要使用 fq（过滤查询）参数去过滤搜索，同样也发送相关的分面搜索命令来更新分面统计数量：

{% highlight http %}
localhost:8983/solr/select?q=camera
&facet=on&facet.field=manu&facet.field=camera_type
&fq=price:[400 to 500]
{% endhighlight %}

fq 命令可以出现在搜索请求的任意位置，参数顺序是无关紧要，请注意我们没有在请求关于价格的分面统计数量，因为约束结果在指定一个价格区间，从而已经知道其他价格区间的分面统计数量为零。

现在，我们获取到 [Solr][1] 响应，更新 Web 前端分面信息的显示和结果列表的显示（包括添加 $400 - $500 的面包屑，允许用户删除），假定用户接着点击了 camera_type 分面中的 SLR，进一步过滤搜索，我们只需再添加一个 fq 参数

{% highlight http %}
localhost:8983/solr/select?q=camera
&facet=on&facet.field=manu
&fq=price:[400 to 500]
&fq=camera_type:SLR
{% endhighlight %}

当 Web 前端获取到 [Solr][1] 的响应，我们就可以添加 SLR 的面包屑，同样再一次更新分面信息的显示和结果列表的显示。

[1]: http://lucene.apache.org/solr/
[2]: http://www.cnet.com
