---
title: Solr 简略笔记
author: 但江
avatar: danjiang
location: 成都
category: programming
---

![Solr in Action](/images/solr-in-action.png)

## Solr 是什么？

Solr 是用 Java 写的搜索引擎，为什么需要有搜索引擎呢，举个例子吧，[热量助手][1] 这款应用可以让你在手机上根据食物名称搜索食物热量，用户搜索 **番茄炒鸡蛋** 和 **西红柿炒鸡蛋** 应该是找同一种食物，需要考虑到同义词的问题，还有用户搜索 **番茄炒鸡蛋**，搜索结果里会有 **西红柿蛋花汤**，**番茄牛腩**，怎么把最相关搜索结果排在前面呢？食物库不断地增加已经突破百万级，怎么保证用户搜索响应在毫秒级呢？这些都是搜索引擎需要干的活儿，Solr 就是一种搜索引擎的解决方案。

## Solr 的核心概念

我们可以在 Solr 中搜索的内容被称为 Document，Document 中有很多 Field，是不是很像关系数据库中 Table 和 Filed 的概念，但是有一点要记住，关系数据库从名字上来看就知道是一种关系型的数据结构，有父子关系这些，Solr 中的 Document 是一种扁平化的数据结构，不同类型间的 Document 通常是毫无关系的，我们在建立模型的时候，可以说建立的食物文本信息就是 Document，食物中的名称、所属分类、热量等级这些就是不同的 Filed。

Document 怎么来的呢，需要通过原始内容信息建立索引形成 Document，原始内容信息可以是网页，关系数据库，PDF 文件，但是不同类型需要不同方式进行处理才能得到我们想要的 Document。

Solr 建立索引形成 Document，这里建立的索引是倒排索引，就像大家看过很多专业书籍一样，在书本最后附录部分有 **操作系统** 这个词出现在 56 页和 88 页，我们要想快速找到书本中关于 **操作系统** 的内容，可以翻到附录部分，找到关键词 **操作系统**，就知道要看的内容在 56 页和 88 页，附录中的关键词是少数，比起你一页页地找内容，这种方式自然要快很多，搜索引擎也是采用的类似的方式，当然要更复杂些，因为有更多实际问题的考虑。

Solr 可以把相关性最高的搜索结果排在前面，Solr 中 Similarity 类实现了这一算法，举个简单例子说明下，比如，我搜索 **安卓系统** 来查找 **安卓系统** 相关的文章，有两篇文章，A 文章中整篇都是描述的 **安卓系统** ，**安卓系统** 这个词在 A 文章中出现了 10 次，B 文章中前一半描述的 **安卓系统**，后一半描述的 **iOS 系统**，**安卓系统** 这个词在 B 文章中出现了 5 次，所以根据相关性的算法，A 文章应该排在搜索结果的前面。

## 建立索引

只需要在 **schema.xml** 配置文件中声明你的 Document 中不同的 Filed 采用什么数据类型，是否索引，是否存储等，Solr 是一个 Java Web 应用，找一个类似于 Tomcat 的 Java Servlet 容器将 Solr 跑起来，通过 Solr 提供的 API 访问方式，你就可以建立索引生成 Document，下面列出了 **schema.xml** 中的部分内容：

{% highlight xml %}
<schema name="kiwi" version="1.5">	
  <field name="id" type="string" indexed="true" stored="true" required="true" multiValued="false" /> 
  <field name="name" type="text_general" indexed="true" stored="true"/>
  <field name="category" type="text_general" indexed="true" stored="true"/>
  <field name="brand" type="text_general" indexed="true" stored="true"/>
  <field name="food" type="boolean" indexed="true" stored="true" />
  <field name="text" type="text_general" indexed="true" stored="false" multiValued="true"/>
</schema>
{% endhighlight %}

## 文本分析

建立索引和搜索过程都需要用到文本分析，比如要将 **番茄炒鸡蛋** 通过分词器分解成 **番茄**、**炒** 和 **鸡蛋** 这三个独立的词，还需要通过一些定制的词汇过滤器来进行一些处理，比如 **番茄** 就需要用 SynonymFilterFactory 来填充同义词 **西红柿**。

下图中展示了原始文字如何通过分词器和词汇过滤器变成了最终的关键词：

![Solr Text Analyze Graph](/images/solr-text-analyze-graph.png)

![Solr Text Analyze Table](/images/solr-text-analyze-table.png)

Tokenizer: parse the text into a stream of tokens

- StandardTokenizer: splitting text on whitespace and punctuation, but also handles acronyms and contractions with ease.
- UAX29URLEmailTokenizer: splits text on whitespace and punctuation, while preserving URLs and email addresses.

Token Filter

- StopFilterFactory: 停用词，常用的但是不能帮助区分文档的词，如 a, an, the
- LowerCaseFilterFactory: 转换成小写
- EnglishPossessiveFilterFactory: 处理英语加 s 成为复数的形式
- KeywordMarkerFilterFactory: 作为关键字，避免被提取词干，如 Manning Publications
- PorterStemFilterFactory: 根据不同语言的语法特点来提取词干
- KStemFilterFactory: 提取词干，没有 Porter 那么激进
- SynonymFilterFactory: 填充同义词
- PatternReplaceCharFilterFactory: 用正则表达式来替换字符
- WhitespaceTokenizerFactory: 根据空格来分词
- WordDelimiterFilterFactory: 根据标点，大小写变化，特殊字符（＃和 @）来分词
- ASCIIFoldingFilterFactory: 将变音符号转换为等同的 ASCII 字母

## 了解更多

[Solr][2] 已经发展多年，涉及内容挺多的，一篇文章只能了解个大概，推荐大家阅读书籍 [Solr in Action][3]。

[1]: http://danthought.com/calorie
[2]: http://lucene.apache.org/solr/
[3]: http://book.douban.com/subject/23133628/
