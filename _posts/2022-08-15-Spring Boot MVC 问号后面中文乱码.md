---
title: Spring Boot MVC 问号后面中文乱码
author: nhsoft.lsd
date: 2022-08-15
categories: [Java,故障解决]
tags: [Spring,Tomcat]
pin: false
---
1. 增加一下参数
```
spring.http.encoding.charset=UTF-8
spring.http.encoding.enabled=true
spring.http.encoding.force=true
```
2. Tomcat，在`server.xml`中的`Connector`标签增加`useBodyEncodingForURI="true"`

![weixin.png](/assets/img/nhsoft_lsd/weixin.png)

<div style="text-align: center;">公众号名称：怪味Coding</div>
<div style="text-align: center;">微信扫码关注或搜索公众号名称</div>
