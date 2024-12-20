---
title: Spring 依赖注入使用Autowired还是Resource
categories: [编程, Java]
tags: [spring]
---

在Spring中依赖注入的方式主要有：构造器注入、setter注入、字段注入。

注入注解常使用`@Autowired`和`@Resource`，那么两者应该使用哪一个？先给结论：无脑用`@Autowired`。

先来简单了解下两者的区别：

