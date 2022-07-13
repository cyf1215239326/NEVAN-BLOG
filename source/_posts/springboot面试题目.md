---
title: springboot面试题目
top: false
cover: false
toc: true
mathjax: true
date: 2022-07-13 15:15:19
password:
summary:
tags: 
	- java
	- springboot
categories:
	- java
	- springboot
---

# springboot 面试题目

1. 描述一下 springboot 的作用

- 自动依赖管理

  - 在 Springboot-dependency 中定义各种 jar 包的版本，进行导入，省去自己去导入的过程

- 自动配置原理
  - springboot.autoconfigure 包保存了大量的自动配置类，对应每个常用的框架，使用 Java 代码对框架进行配置
  - 每个自动配置类生效的条件是：导入了对应的依赖 @ConditionOnClass({类.class})
  - 在 META-INF/spring.factores 中把所有自动配置类的全名定义出来
  - 在 SpringBoot 类上有@SpringBootApplication 注解
  - 该注解由三个注解组成：
    - SpringbootConfiguration 代表该类作为配置类使用
    - ComponentScan 对包进行扫描
    - EnableAutoConfiguration 启动自动配置
  - 在 EnableAutoConfiguration 注解的 XXSelector 源码中，会读取 spring.factores 文件，通过反射将所有的自动配置类加载到内存中，启动了自动配置

1. springboot 有哪些特性

- 遵循习惯优于配置的原则。使用 springboot 我们只需要很少的配置，大多数使用默认配置即可
- 项目快速搭建。springboot 帮助开发者快速搭建 spring 框架，可无需配置的自动整合第三方框架
- 可以完全不使用 xml 配置，只需要自动配置和 Java config
- 内嵌 servlet 容器，降低了对环境的要求，可用命令直接执行项目
- 提供了 starter POM，能够非常方便的进行包管理
- 对主流框架无配置集成
- 与云计算天然集成
