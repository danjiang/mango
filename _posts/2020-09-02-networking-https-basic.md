---
title: HTTPS 简介
author: 但江
avatar: danjiang
location: 成都
category: programming
tag: objective-c
---

本文是对 HTTPS 运行机制的整理，便于记忆。

![Networking](/images/networking.jpg)

## HTTP 的缺点

* 通信使用明文（不加密），内容可能会被窃听。
* 不验证通信方的身份，因此有可能遭遇伪装。
* 无法证明报文的完整性，所以有可能已遭篡改。

## HTTPS 的运行机制

### HTTPS 是身披 SSL 外壳的 HTTP

HTTP 直接和 TCP 通信。当使用 SSL 时，则演变成先和 SSL 通信，再由 SSL 和 TCP 通信了。

![HTTP SSL](/images/http-ssl.png)

### 加密方法

近代的加密方法中加密算法是公开的，而密钥却是保密的。

* 对称加密、共享密钥加密：加密和解密同用一个密钥的方式称为共享密钥加密。
* 非对称加密、公开密钥加密：公开密钥加密使用一对非对称的密钥，一把叫做私有密钥，另一把叫做公开密钥，使用公开密钥加密方式，发送密文的一方使用对方的公开密钥进行加密处理，对方收到被加密的信息后，再使用自己的私有密钥进行解密。

### 数字证书

用于证明公开密钥正确性的证书。

![Certificate](/images/certificate.jpg)

### SSL 协议的握手过程

![SSL Handshake](/images/ssl-handshake.png)

假定客户端叫做爱丽丝，服务器叫做鲍勃：

1. 爱丽丝给出协议版本号、一个客户端生成的随机数 Client Random，以及客户端支持的加密方法。
2. 鲍勃确认双方使用的加密方法，并给出数字证书、以及一个服务器生成的随机数 Server Random。
3. 爱丽丝确认数字证书有效，然后生成一个新的随机数 Premaster Secret，并使用数字证书中的公钥，加密这个随机数，发给鲍勃。
4. 鲍勃使用自己的私钥，获取爱丽丝发来的随机数，即 Premaster Secret。
5. 爱丽丝和鲍勃根据约定的加密方法，使用前面的三个随机数，生成 "对话密钥" Session Key，用来加密接下来的整个对话过程。

注意点：

* 生成对话密钥一共需要三个随机数。
* 握手之后的对话使用 "对话密钥" 加密，是对称加密，服务器的公钥和私钥只用于加密和解密 "对话密钥"，是非对称加密，无其他作用。
* 服务器公钥放在服务器的数字证书之中。

## 参考

* [图解 HTTP - 第 7 章　确保 Web 安全的 HTTPS](https://book.douban.com/subject/25863515/)
* [SSL/TLS 协议运行机制的概述](http://www.ruanyifeng.com/blog/2014/02/ssl_tls.html)
* [图解 SSL/TLS 协议](http://www.ruanyifeng.com/blog/2014/09/illustration-ssl.html)