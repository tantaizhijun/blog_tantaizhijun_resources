---
layout: post
title: spring系统架构与源码解读笔记
date: 2018-12-23 21:50:20
tags:
categories: spring
---

### 一. spring系统架构

    spring总共大约有20个模块, 由1300多个文件构成. 这些组件被分别整合在以下6个模块集合中:
        - 核心容器(Core Container)
        - AOP(Aspect Oriented Programming)
        - 设备支持(Instrumentaion)
        - 数据访问与集成(Data Access/Integration)
        - Web
        - 报文发送(Messaging)
        - Test
        具体如下图:
<!--more-->            
   ![图片](./01.png)

    spring家族的依赖关系图如下:
    
   ![图片](./2.png)
   
### 二. spring IOC核心体系结构

    - 1.BeanFactory
        *一切从Bean开始*
        Spring Bean的创建是典型的工厂模式,这一系列的Bean工厂, (即IOC容器), 为开发者管理对象之间的依赖关系提供了很多遍历和基础服务, 
        我们常用的启动spring的类: ClassPathXmlApplication也是一个BeanFactory,
        其继承关系图如下:
        
   ![图片](./3.png)
   

