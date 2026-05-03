---
title: 使用cloudflare的Origin Rules解决web https和xray服务443端口冲突的问题
date: 2026-05-03 16:18:10
tags:
    - cloudflare
categories:
    - 技术分享
    - 运维
---

最近新开了一台AWS的LightSail实例（新加坡，512M，每月5美刀），由于xray服务使用了443端口，我又有需求需要在同一个示例上面运行两个web服务，而且均需要提供https访问，一个是帮朋友制作的一个小型门户网站（基本上是静态的），另外一个是tvbox的订阅js文件，也是静态的，使用nginx的try_files功能实现。
之前的想法是使用nginx的stream模块的ssl_preread功能，nginx侦听443端口，然后xray跑在1443，然后两个web服务都跑在2443端口，nginx根据域名（server_name）来区分访问哪个web服务。由于Lightsail的示例配置很低，这样做无疑不是一个好选择。
<!-- more -->
后面我有了两个想法：

**方法1：**
两个web服务都不用https，只使用http，然后开启cloudflare的Flexible SSL功能，cloudflare和用户之间是https，cloudflare和服务器之间是http，这样就可以避免443端口冲突的问题了。
**方法2：**
xray侦听443端口，然后fallback到web服务器的80和81端口，81是h2c的端口，80是http的端口。

但是这两个方法都存在缺陷，第一个方法80端口可以被直接http访问，url有泄露风险，虽然可以通过修改cloudflare的规则来实现http强制跳转https，但是http那个请求还是会泄露url。第二个方法则更加麻烦，由于门户网站使用的域名和代理使用的不一样，用xray fallback直接导致证书错误，而且由于ALPN的缘故，每个域名还要配置两个fallback规则，也不是很理想的做法。

经过一番搜索，我发现了另外一个不错的方法：
1. xray侦听443端口，web服务侦听8443端口，启用SSL。
2. cloudflare的SSL/TLS设置为Full。
3. 对于需要走CDN代理的域名，打开橙色的云朵，开启代理。
4. 配置cloudflare的Origin Rules，设置当访问某个域名时，cloudflare会将请求转发到服务器的8443端口。
这样就可以解决443端口冲突的问题，同时也能保证web服务的安全性和性能。

具体配置的方法如下：
1. 导航到规则tab，选择Origin Rules，点击创建规则：
![](/images/2026-05-03_17-08.png)
![](/images/2026-05-03_17-09.png)

2. 对于要从443回源到8443的域名进行类似如下图的配置：
![](/images/2026-05-03_17-03.png)
要是界面不好点，直接修改表达式方便点，大致就是如果HOST匹配，然后又是SSL协议。
```shell
(http.host eq "tvbox.xxxxx.top" and ssl) or (http.host eq "zj.xxxxx.top" and ssl)
```

3. 转发到8443端口：
![](/images/2026-05-03_17-04.png)

保存后退出，检查规则已经生效：
![](/images/2026-05-03_17-05.png)

此时访问这个域名的https（默认443端口）就会被cloudflare转发到服务器的8443端口，而xray服务仍然可以正常使用443端口提供服务了。
