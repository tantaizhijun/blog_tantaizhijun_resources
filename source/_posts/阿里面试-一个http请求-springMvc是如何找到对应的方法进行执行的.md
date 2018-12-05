---
layout: post
title: '阿里面试: 一个http请求,springMvc是如何找到对应的方法进行执行的?'
date: 2018-12-05 11:18:42
categories: spring,java
tags: springMvc
---

最近在看spring mvc的相关源码,写了一个mvc框架小demo, 正好今天看到这个面试题,就把相关的只是点记录一下:
还不够详细...回头看完了源码,再补充....

### 1.启动时方法的注册判断

读过spring源码的同学应该都知道, spring在项目启动时, 会扫描所有的类, 为他们创建对应的bean并放进容器,
其实, spring mvc原理也一样,
在项目启动时, 扫描所有的类的时候,会判断每个类上是否有相应的注解(如@controller,@service,@repository等),
会判断每个类上是否有controller和requestMapping注解,然后把他们的方法拿到注册中心(detectHandlerMethods())进行注册..

**详细源码看图:**
![图片](./1.png)

<!--more-->

图1中 isHandler()方法的判断原则:
![图片](./2.png)
把他们的方法拿到注册中心(detectHandlerMethods())进行注册,源码:
对应图1中的detectHandlerMethods()
![图片](./3.png)
图3中的 registerHandlerMethod()方法
![图片](./4.png)

### 2.怎么注册?
遍历类中所有的方法 放到一个map,然后遍历map,然后判断哪些带有requestMapping注解,
然后注册到注册中心 (registerHandlerMethod() => mappingRegistry.registry(mapping,handler,method))
MappingRegistry这个注册中心定义了一堆final类型的map,,,比如urlLookup这个map,就是记录这个url地址应该由哪个方法进行处理的.
![图片](./5.png)
![图片](./6.png)

### 3.调用的时候呢.
通过getHandlerInternal(request) 获取地址url,(getUrlPathHelper.getLookupPathForRequest)
然后到注册中心去找对应的方法去执行
![图片](./6.1.png)
![图片](./7.png)

### 更多源码说明看完补充...