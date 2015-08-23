---
title: 浏览器缓存
author: 但江
location: 成都
category: programming
---

![Browser](/images/browser.jpg)

浏览器缓存带来最大价值是减少带宽，搞懂 HTTP 协议中有关缓存的部分很有价值。

#### 浏览器缓存放置位置

浏览器缓存当然放在本地咯，放置的位置：

* IE C:\Documents and Settings\danjiang\Local Settings\Temporary Internet Files
* Firefox 浏览器地址栏输入 about:cache

#### 缓存协商

浏览器通过 HTTP 协议和服务器交流，来处理浏览器缓存，具体相关信息。

**Last-Modified**

1. 前提是基于 GET 类型请求，对于 POST 类型的请求，浏览器一般不启用本地缓存；
2. 浏览器第一次请求资源后，服务器在 HTTP 响应头中包含 Last-Modified 信息，并返回响应体；
3. 浏览器再次请求同一资源，HTTP 请求包含 If-Modified-Since 的信息；
4. 服务器收到此信息进行比较后得出所请求的资源没有变动，服务器在 HTTP 响应头包含 304 Not Modified 的信息，浏览器直接从本地获取响应体。

**Etag**

采用一串编码来标记内容，通过编码是否改变判断内容是否改变，Last-Modified 的替换方法。

**Expires**

暗示浏览器在该内容过期之前不需要再询问服务器，而直接使用本地缓存即可。

**Cache-Control**

指定了缓存过期的相对时间，浏览器会根据这个值在本地计算出过期时间，避免了 Expires 指定的过期时间来自服务器，但本地时间和服务器时间不一致带来的问题。

#### 请求页面的不同方式

* Crtl+F5 强制刷新 —— 不使用缓存协商，获取所有内容的最新版本；
* F5 一般刷新 —— Last-Modified 有效，Expries 无效；
* 转到或超链接 —— Last-Modified 有效，Expries 有效。
