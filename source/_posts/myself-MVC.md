---
title: 自己动手写mvc框架
date: 2018-11-26 10:45:42
categories: spring
tags: mvc
---

### 自己动手写mvc框架
   最近在看自己spring源码和mvc相关的课程,于是就把相关的笔记记在了这里....

#### 一. spring源码的核心组件: IOC  AOP
  1.IOC容器: 负责实例化, 定位, 配置应用程序中的对象及建立这些对象之间的依赖.比如对orderService的生命周期的管理.
    当服务器(如tomcat)启动时--->进行加载项目--->spring会把所有的Bean全部创建出来,放进IOC容器.
    IOC容器就是一个很大的map对象. 源码为证:
    [图片]
    IOC: Map: map.put(key,val);  key就是声明的beanName, value就是new Obj();
    当使用bean时, 就是从容器中get("beanName"), 返回保存的bean实例.
  2. AOP面向切面编程,通过预编译的方式和运行期动态代理实现程序功能的统一维护的一种技术
#### 二. MVC核心类: DispatherServlet
    它继承了httpServlet, 它就是一个servlet

