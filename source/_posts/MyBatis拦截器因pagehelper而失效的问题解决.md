---
title: MyBatis拦截器因pagehelper而失效的问题解决
date: 2018-08-01 13:29:17
tags: [CAT, 监控, Mybatis拦截器, Mybatis, 分页]
---


# 

`pagehelper`是github开源的一款MyBatis分页插件，该插件是通过MyBatis拦截器实现的。然而使用`pagehelper`有可能会导致其他拦截器失效，今天就遇到了这个问题。

## 事件经过

### 1.起因
今天一到公司就有业务系统反映CAT无法记录SQL的埋点，经过mentor的提示，可能是cat客户端中的mybatis拦截器`catMybatisPlus`失效而没有上报SQL埋点，大概率和分页插件有关。

### 2.过程

于是大致瞟了一眼业务系统使用的mybatis分页插件是`pagehelper`，然后就开始搜索MyBatis多个拦截器冲突问题。直觉告诉我，这个问题应该和拦截器的执行顺序有关。

MyBatis的拦截器采用责任链设计模式，多个拦截器之间的责任链是通过动态代理组织的。我们一般都会在拦截器中的`intercept`方法中往往会有` invocation.proceed()`语句，其作用是将拦截器责任链向后传递，本质上便是动态代理的invoke。

回到`pagehelper`源码查看，可以看到其`inercept`方法直接获取了`excutor`然后开始分页查询，当查询到结果时，便返回了。在此，我们发现了关键点，那就是`pagehelper`的`intercept`方法中没有`invocation.proceed()`，这意味着什么？这意味着`pagehelper`没有继续向后传递责任链，而是自行处理直接返回。由此，我们可以猜出该问题大概率与拦截器的执行顺序有关。通过断点调试，验证了该猜想，当遇到分页查询时，执行到`pagehelper`就结束了，没有进入我们的`catMybatisPlugin`。

`pagehelper`的拦截器是通过配置类`PageHelperAutoConfiguration`注册的，而非常规的通过xml文件，接着我们在`pagehelper`的配置类上注意到了一个注解`@AutoConfigureAfter(MybatisAutoConfiguration.class)`，这意味着pagehelper是最后注册的，这意味着该拦截器是在动态代理的最外层，当MyBatis开始执行SQL时，首先进入的就是pagehlper拦截器，处理返回；之后拦截器链上的拦截器不再处理。

### 3.解决

到此，问题的答案已经越来越接近，只需让`catMybatisPlus`在`pagehelper`之后注册即可。解决办法便是，仿照`pagehelper`写一个`catMybatisPlus`的配置类，在该类上使用注解`@AutoConfigureAfter(PageHelperAutoConfiguration.class)`即可。编码实现，至此，问题得以解决。

然而最后一步还有一个小插曲，一开始`@AutoConfigureAfter`并没有令`catMybatisPlus`前置注册。经过简单搜索发现`@AutoConfigureAfter`注解只可应用于autoconfigue类型的bean。而autoconfigue类型的bean一般用于spring-boot-starter导入文件，需要在`src/main/resources/META-INF`目录下的`spring.factories`声明。不过这里对我们来说，虽然不是一个spring-boot-starter，但用一下也无妨。创建`src/main/resources/META-INF/spring.factories`，声明该配置类即可。

## 总结

该问题的解决过程不是极其艰难，不过遇到了一些比较经典的关键点，比如责任链设计模式，比如动态代理，比如spring-boot-starter的规范。在过程中，为了实现配置的先后执行，甚至又扒了一边spring bean的加载过程，企图通过改变bean的加载顺序达到目的。整个过程比较有意思，故撰文记录。