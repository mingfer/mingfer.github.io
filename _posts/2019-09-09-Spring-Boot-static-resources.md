---
layout: post
title: "Spring-Boot 静态资源配置"
subtitle: "Spring-Boot 是如何加载静态资源的"
date: 2019-09-09 19:39:00
author: "mingfer"
header-img: "img/post/bg-1.jpg"
catalog: true
tags: 
    - "Spring Boot"
    - "Web 后端"
---

## 前言

当我们在发布一个非前后端分离的，带 Web 页面的应用的时候，Spring Boot 是如何识别这些静态资源的呢？我们可以做哪些客户化的定制实现我们额外的需求呢？下面我们一起探讨一下这一部分的内容。

## 推荐配置

在官方文档 [29.1.5 Static Content](https://docs.spring.io/spring-boot/docs/2.1.8.RELEASE/reference/htmlsingle/#boot-features-spring-mvc-static-content) 中详细介绍了 Spring Boot 是如何进行静态资源配置的。

在默认的情况下，Spring Boot 会从 `ServletContext` 跟目录下的四个位置去寻找静态资源信息，他们分别是：`/static`，`/public`，`/resources`，`/META-INF/resources`。在 Spring Boot 工程中，这四个目录一般建在 `src/main/resources` 下面。 因为 Spring Boot 使用 Spring MVC 中的 `ResourceHttpRequestHandler` 进行静态资源的映射，所以也可以通过继承并覆盖 `WebMvcConfigurer#addResourceHandlers` 方法实现通过代码配置静态资源的位置。

在独立的 Web 应用程序中（不包含静态资源），静态资源加载相关的 Servlet 也是默认启动的，除非我们人为的修改了了 Spring MVC 的相关配置。

默认情况下，静态资源映射到的 URI 路径是 `/**`，这个路径可以通过配置 `spring.mvc.static-path-pattern` 进行更改，比如下面的例子中所有的静态资源都需要 `/resources` 前缀才能访问：

```properties
# 对于资源目录下的 index.html，默认的情况下通过 localhost:8080/index.html 访问
# 在下面的配置下必须通过 localhost:8080/resources/index.html 访问
spring.mvc.static-path-pattern=/resources/**
```

同时，我们也可以通过配置 `spring.resources.static-locations` 来指定静态资源的位置，该配置项下面可以配置相对于 `ServletContext` 跟目录的目录列表。**注意**：一般来说， `ServletContext` 跟目录其实就是 classpath 加载的目录，参见下面的例子：

- 指定 jar 包外的静态资源目录

```bash
# 如果启动的时候指定了一个 jar 外的资源路径
$ java -classpath /home/web/static/ -jar web-application.jar

# /home/web/static/ 的目录结构如下：
$ ls /home/web/static
index.html js/ css/

# 那么这里 ServletContext 根目录 / 相当于 /home/web/static/ ，如果我们不使用 -classpath 进行目录的指定，就算配置 spring.resources.static-locations=/home/web/static/ 也是无法访问资源的。
```

- 指定 jar 包内部的静态资源目录

```bash
# 如果我们的静态文件打包在了 jar 里面，这里的 jar 必须是使用 Spring Boot 的插件进行打包的
$ java -jar web-application.jar
# 此时  ServletContext 根目录 / 相当于 web-application.jar!/BOOT-INF/classes
```

## 代码配置

上面有提到，我们实际上可以通过代码进行资源 URI 路径映射和资源位置的配置。参考代码如下：

```java
package cn.mingfer.demo.resources;

import org.springframework.context.annotation.Configuration;
import org.springframework.web.servlet.config.annotation.ResourceHandlerRegistry;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurer;

@Configuration
public class ResourcesConfiguration implements WebMvcConfigurer {

    /**
     * Add handlers to serve static resources such as images, js, and, css
     * files from specific locations under web application root, the classpath,
     * and others.
     *
     * @param registry
     */
    @Override
    public void addResourceHandlers(ResourceHandlerRegistry registry) {
        registry.addResourceHandler("/**")
                .addResourceLocations("classpath:/static", "classpath:/public");
    }
}

```

在上面的代码中，将静态资源的 URI 路径映射配置为了 `/**`，将资源位置配置为了在 classpath 下了 `/static` 和 `/public` 两个目录。

