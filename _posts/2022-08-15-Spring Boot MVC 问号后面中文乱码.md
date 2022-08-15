---
title: Spring Boot MVC 问号后面中文乱码
author: nhsoft.lsd
date: 2022-08-15
categories: [故障解决,Spring]
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
