---
layout: post
title: spring mvc拦截器工作原理
date: 2019-03-31 15:25:49
tags:
categories: spring mvc
---

### 1.spring mvc拦截器简介
   Spring Web MVC的处理器拦截器（如无特殊说明，下文所说的拦截器即处理器拦截器）类似于Servlet开发中的过滤器Filter，用于对处理器进行预处理和后处理。

### 2.应用场景
<!--more-->
   1、日志记录：记录请求信息的日志，以便进行信息监控、信息统计、计算PV（Page View）等。
   2、权限检查：如登录检测，进入处理器检测检测是否登录，如果没有直接返回到登录页面；
   3、性能监控：有时候系统在某段时间莫名其妙的慢，可以通过拦截器在进入处理器之前记录开始时间，在处理完后记录结束时间，从而得到该请求的处理时间（如果有反向代理，如apache可以自动记录）；
   4、通用行为：读取cookie得到用户信息并将用户对象放入请求，从而方便后续流程使用，还有如提取Locale、Theme信息等，只要是多个处理器都需要的即可使用拦截器实现。
   5、OpenSessionInView：如Hibernate，在进入处理器打开Session，在完成后关闭Session。
   …………本质也是AOP（面向切面编程），也就是说符合横切关注点的所有功能都可以放入拦截器实现。
   
### 拦截器工作方式
 ![图片](./01.jpg)
    
### 3.拦截器实现

**3.1 springMVC拦截器的实现一般有两种方式：**
    
   - 第一种方式是要定义的Interceptor类要实现了Spring的HandlerInterceptor 接口
   - 第二种方式是继承实现了HandlerInterceptor接口的类，比如Spring已经提供的实现了HandlerInterceptor接口的抽象类HandlerInterceptorAdapter
 
**3.2拦截器接口：**
 
   ```
   package org.springframework.web.servlet;  
   public interface HandlerInterceptor {  
       boolean preHandle(HttpServletRequest request, HttpServletResponse response,Object handler)   
               throws Exception;  
     
       void postHandle(HttpServletRequest request, HttpServletResponse response,Object handler, ModelAndView modelAndView)   
               throws Exception;  
     
       void afterCompletion(HttpServletRequest request, HttpServletResponse response,   
               Object handler, Exception ex)  
               throws Exception;  
   }
   ```
   
**方法说明：**
   HandlerInterceptor 接口中定义了三个方法，我们就是通过这三个方法来对用户的请求进行拦截处理的。
   - **preHandle()：** 这个方法在业务处理器处理请求之前被调用，SpringMVC 中的Interceptor 是链式的调用的，在一个应用中或者说是在一个请求中可以同时存在多个Interceptor 。每个Interceptor 的调用会依据它的声明顺序依次执行，而且最先执行的都是Interceptor 中的preHandle 方法，所以可以在这个方法中进行一些前置初始化操作或者是对当前请求的一个预处理，也可以在这个方法中进行一些判断来决定请求是否要继续进行下去。该方法的返回值是布尔值Boolean 类型的，当它返回为false 时，表示请求结束，后续的Interceptor 和Controller 都不会再执行；当返回值为true 时就会继续调用下一个Interceptor 的preHandle 方法，如果已经是最后一个Interceptor 的时候就会是调用当前请求的Controller 方法。
   - **postHandle()：** 这个方法在当前请求进行处理之后，也就是Controller 方法调用之后执行，但是它会在DispatcherServlet 进行视图返回渲染之前被调用，所以我们可以在这个方法中对Controller 处理之后的ModelAndView 对象进行操作。postHandle 方法被调用的方向跟preHandle 是相反的，也就是说先声明的Interceptor 的postHandle 方法反而会后执行。
   - **afterCompletion()：** 整个请求处理完毕回调方法，即在视图渲染完毕时回调，该方法也是需要当前对应的Interceptor 的preHandle 方法的返回值为true 时才会执行。顾名思义，该方法将在整个请求结束之后，也就是在DispatcherServlet 渲染了对应的视图之后执行。这个方法的主要作用是用于进行资源清理工作的。
   
 
### 4.看一下DispatcherServlet内部到底是如何工作的
**注：** 以下是流程的简化代码，中间省略了部分代码，不完整。
```
//doDispatch方法  
//1、处理器拦截器的预处理（正序执行）  
HandlerInterceptor[] interceptors = mappedHandler.getInterceptors();  
if (interceptors != null) {  
    for (int i = 0; i < interceptors.length; i++) {  
    HandlerInterceptor interceptor = interceptors[i];  
        if (!interceptor.preHandle(processedRequest, response, mappedHandler.getHandler())) {  
            //1.1、失败时触发afterCompletion的调用  
            triggerAfterCompletion(mappedHandler, interceptorIndex, processedRequest, response, null);  
            return;  
        }  
        interceptorIndex = i;//1.2、记录当前预处理成功的索引  
}  
}  
//2、处理器适配器调用我们的处理器  
mv = ha.handle(processedRequest, response, mappedHandler.getHandler());  
//当我们返回null或没有返回逻辑视图名时的默认视图名翻译（详解4.15.5 RequestToViewNameTranslator）  
if (mv != null && !mv.hasView()) {  
    mv.setViewName(getDefaultViewName(request));  
}  
//3、处理器拦截器的后处理（逆序）  
if (interceptors != null) {  
for (int i = interceptors.length - 1; i >= 0; i--) {  
      HandlerInterceptor interceptor = interceptors[i];  
      interceptor.postHandle(processedRequest, response, mappedHandler.getHandler(), mv);  
}  
}  
//4、视图的渲染  
if (mv != null && !mv.wasCleared()) {  
render(mv, processedRequest, response);  
    if (errorView) {  
        WebUtils.clearErrorRequestAttributes(request);  
}  
//5、触发整个请求处理完毕回调方法afterCompletion  
triggerAfterCompletion(mappedHandler, interceptorIndex, processedRequest, response, null);

```

triggerAfterCompletion方法

```
// triggerAfterCompletion方法  
private void triggerAfterCompletion(HandlerExecutionChain mappedHandler, int interceptorIndex,  
            HttpServletRequest request, HttpServletResponse response, Exception ex) throws Exception {  
        // 5、触发整个请求处理完毕回调方法afterCompletion （逆序从1.2中的预处理成功的索引处的拦截器执行）  
        if (mappedHandler != null) {  
            HandlerInterceptor[] interceptors = mappedHandler.getInterceptors();  
            if (interceptors != null) {  
                for (int i = interceptorIndex; i >= 0; i--) {  
                    HandlerInterceptor interceptor = interceptors[i];  
                    try {  
                        interceptor.afterCompletion(request, response, mappedHandler.getHandler(), ex);  
                    }  
                    catch (Throwable ex2) {  
                        logger.error("HandlerInterceptor.afterCompletion threw exception", ex2);  
                    }  
                }  
            }  
        }  
    }  
```
   
### 5.拦截方式
可以利用mvc:interceptors标签声明一系列的拦截器，然后它们就可以形成一个拦截器链，拦截器的执行顺序是按声明的先后顺序执行的，先声明的拦截器中的preHandle方法会先执行，然而它的postHandle方法和afterCompletion方法却会后执行。

#### 5.1 方式一：总拦截器，拦截所有url
定义一个Interceptor实现类的bean对象。使用这种方式声明的Interceptor拦截器将会对所有的请求进行拦截。

```
<mvc:interceptors>
    <bean class="com.app.mvc.MyInteceptor" />
</mvc:interceptors>
```

#### 5.2 方式二：总拦截器， 拦截匹配的URL

```
<mvc:interceptors >  
  <mvc:interceptor>  
         <!--  
           /** 的意思是所有文件夹及里面的子文件夹 
           /* 是所有文件夹，不含子文件夹 
           / 是web项目的根目录
         --> 
        <mvc:mapping path="/user/*" />
        <!-- 需排除拦截的地址 -->  
　　　　 <mvc:exclude-mapping path="/user/b" />

        <bean class="com.mvc.MyInteceptor"></bean>  
    </mvc:interceptor>  
    
     <!-- 当设置多个拦截器时，先按顺序调用preHandle方法，然后逆序调用每个拦截器的postHandle和afterCompletion方法  -->
</mvc:interceptors> 
 ```
 
 使用mvc:interceptor标签进行声明。使用这种方式进行声明的Interceptor可以通过mvc:mapping子标签来定义需要进行拦截的请求路径。
 
 mapping只能映射某些需要拦截的请求，而exclude-mapping用来排除某些特定的请求映射。当我们需要拦截的请求映射是比较通用的，但是其中又包含了某个特殊的请求是不需要使用该拦截器的时候我们就可以把它定义为exclude-mapping了。比如像下面示例这样，我们定义的拦截器将拦截所有匹配/user/**模式的请求，但是不能拦截请求“/user/b”，因此它定义为了exclude-mapping。当定义了exclude-mapping时，Spring MVC将优先判断一个请求是否在execlude-mapping定义的范围内，如果在则不进行拦截。 实际上一个interceptor下面定义的mapping和exclude-mapping都是可以有多个的。
 
 另外，exclude-mapping的定义规则和mapping的定义规则是一样的，我们也可以使用一个星号表示任意字符，使用两个星号表示任意层次的任意字符。
 
 **注：**
 如果使用了<mvc:annotation-driven />， 它会自动注册DefaultAnnotationHandlerMapping 与AnnotationMethodHandlerAdapter 这两个bean,所以就没有机会再给它注入interceptors属性，就无法指定拦截器。
 
 当然我们可以通过人工配置上面的两个Bean，不使用 <mvc:annotation-driven />，就可以 给interceptors属性 注入拦截器了
 
#### 6. 对静态资源不拦截的配置
 ```
 <mvc:interceptors>
    <mvc:interceptor>
        <mvc:mapping path="/**" />
        <mvc:exclude-mapping path="/js/**" />
         <mvc:exclude-mapping path="/css/**" />
         <mvc:exclude-mapping path="/image/**" />
         <bean id="commonInterceptor" class="org.shop.interceptor.CommonInterceptor"></bean> <!--这个类就是我们自定义的Interceptor -->
    </mvc:interceptor>
 </mvc:interceptors>

 ```
 或者在web.xml中配置
 ```
    <!-- 不拦截静态文件 -->
    <servlet-mapping>
        <servlet-name>default</servlet-name>
        <url-pattern>/js/*</url-pattern>
        <url-pattern>/css/*</url-pattern>
        <url-pattern>/images/*</url-pattern>
        <url-pattern>/fonts/*</url-pattern>
    </servlet-mapping>
```
 
 