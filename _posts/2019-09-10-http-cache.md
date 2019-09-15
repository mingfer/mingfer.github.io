---
layout: post
title: "HTTP Cache"
subtitle: "Spring Boot 如何配置 HTTP Cache"
date: 2019-09-10 20:00:00
author: "mingfer"
header-img: "img/post/bg-2.jpg"
catalog: true
tags: 
    - "Spring Boot"
    - "Web 后端"
---

## 前言

HTTP Cache 是一种提高 Web 页面资源加载效率的技术手段。具体实现为：Web 资源经过的各级系统通过存储 Web 文档，视频，网页，图片等资源的副本。在指定的条件下，前端（如浏览器）直接加载存储的副本，而无需到服务器上请求，以此减少资源请求的滞后性。

相关规范：[RFC 7234](https://tools.ietf.org/html/rfc7234)

## HTTP Cache

HTTP Cache 的实现主要和 HTTP Header 中的几个字段有关，我们一起了解一下。

### Cache-Control

用于指定缓存在整个请求/响应链中必须服从的指令。这些指令指定用于指导缓存对于请求或响应内容的具体行为。缓存指令是单向的，在请求中存在一个指令并不意味着响应中也会存在同一个指令，请求中的指令一般是客户端设置的，响应中的指令一般是服务端设置的。

一般来说常用的 Cache-Control 指令包括：

| 指令                                 | 效果                                                         |
| ------------------------------------ | ------------------------------------------------------------ |
| no-store                             | 不对资源做任何的缓存                                         |
| no-cache                             | 必须先与服务器确认返回的响应是否被更改，然后才能使用该响应来满足后续对同一个网址的请求。no-cache 会发起通信通过 E-tag 字段向服务器验证缓存的响应，如果资源未被更改，那么加载副本中的资源。 |
| max-age=30                           | 缓存的内容将在指定的秒数（例子是 30 秒）后失效, 这个选项只在 HTTP 1.1 及以上可用。当和 Last-Modified 一起使用时, max-age 的优先级更高。 |
| public                               | 所有内容都将被缓存（客户端和代理服务器都可缓存），并且该缓存是共享的。也就是如果新打开一个一模一样的网页窗口，会直接使用前一个窗口的缓存。 |
| private                              | 内容只缓存到私有缓存中（仅客户端可以缓存，代理服务器不可缓存）。如果新打开一个一模一样的网页窗口，不允许使用前一个窗口的缓存，而是向服务器请求资源。 |
| must-revalidation/proxy-revalidation | 如果缓存的内容失效，请求必须发送到服务器/代理以进行重新验证  |

下面是在不同的情况下，浏览器是将请求发往服务器还是直接使用缓存中的内容：

| 指令       | 打开一个新的浏览器窗口           | 在原窗口敲击 Enter | 刷新       | 单击 Back 按钮 |
| ---------- | -------------------------------- | ------------------ | ---------- | -------------- |
| no-store   | 发送请求到 Server，不会比较 Etag | 和前面一致           | 和前面一致 | 和前面一致     |
| no-cache   | 发送请求到 Server，会比较 Etag   | 和前面一致         | 和前面一致 | 和前面一致     |
| max-age=30 | 在 30 秒后，浏览器重新发送请求到服务器 | 在 30 秒后，浏览器重新发送请求到服务器 | 浏览器重新发送请求到服务器 | 在 30 秒后，浏览器重新发送请求到服务器 |
| public | 浏览器呈现来自缓存的页面 | 浏览器呈现来自缓存的页面 | 浏览器重新发送请求到服务器 | 浏览器呈现来自缓存的页面 |
| public | 浏览器呈现来自缓存的页面 | 浏览器呈现来自缓存的页面 | 浏览器重新发送请求到服务器 | 浏览器呈现来自缓存的页面 |
| must-revalidation/proxy-revalidation | 浏览器重新发送请求到服务器 | 第一次，浏览器重新发送请求到服务器；此后，浏览器呈现来自缓存的页面 | 浏览器重新发送请求到服务器 | 浏览器呈现来自缓存的页面 |

### Expires

Expires 字段存储了一个 GMT 格式的时间戳，在改时间戳之后的响应会被认为失效。失效的缓存条目通常不会被缓存（无论是代理缓存还是用户代理缓存）返回，除非通过了原始服务器（或者拥有该实体的最新副本的中介缓存）验证。（注意：Cache-Control 中的 max-age 和 s-maxage 指令将覆盖 Expires。）

Expires 字段中时间戳格式为 `Expires: Sun, 08 Nov 2009 03:37:26 GMT`，如果请求资源时的日期在给定的日期之前，则认为该资源没有失效直接缓存中提取出来使用，反之则认为该资源失效。

### E-Tag

ETag 响应头部字段值是一个实体标记，它提供一个 “不透明” 的缓存验证器。这可能在以下几种情况下提供更可靠的验证：不方便存储修改日期；HTTP 日期值的 one-second （主要是可能会有一秒内多次请求的情况）解决方案不够用；或者原始服务器希望避免由于使用修改日期而导致的某些冲突。

## 配置 HTTP Cache

在 Spring Boot 的 [ResourceProperties](https://github.com/spring-projects/spring-boot/blob/v2.1.8.RELEASE/spring-boot-project/spring-boot-autoconfigure/src/main/java/org/springframework/boot/autoconfigure/web/ResourceProperties.java) 类中提供了关于 HTTP Cache 的配置。使用外部配置：

```properties
# 开启 chain cache
spring.resources.chain.cache=true

# http 缓存过期时间
spring.resources.cachePeriod=60 
```

此外还可以通过实现 `WebMvcConfigurer` 接口的 `addResourceHandlers` 方法通过代码进行 HTTP 缓存的配置：

```java
@Override
public void addResourceHandlers(ResourceHandlerRegistry registry) {
   registry.addResourceHandler("/static/**")
           .addResourceLocations("classpath:/static/")
           .setCacheControl(CacheControl
                   .maxAge(10, TimeUnit.MINUTES)
                   .cachePrivate());
}
```

我们也可以定义自己的 HTTP Cache 配置类，实现个性化的配置：

配置类：`ResourcesConfiguration`

```java
package cn.keyou.web.common.resources;

import org.springframework.boot.autoconfigure.web.ResourceProperties;
import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.servlet.config.annotation.ResourceHandlerRegistration;
import org.springframework.web.servlet.config.annotation.ResourceHandlerRegistry;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurer;

/**
 * 前端页面资源配置
 */
@Configuration
@ConfigurationProperties("service.resources")
public class ResourcesConfiguration implements WebMvcConfigurer {

    private String[] handler = new String[]{"/**"};

    private String[] locations = new String[]{"classpath:/resources/", "classpath:/META-INF/resources/", "classpath:/static/", "classpath:/public/"};

    private ResourceProperties.Cache.Cachecontrol cache;

    @Override
    public void addResourceHandlers(ResourceHandlerRegistry registry) {
        ResourceHandlerRegistration registration = registry.addResourceHandler(handler).addResourceLocations(locations);
        if (cache == null) {
            cache = new ResourceProperties.Cache.Cachecontrol();
            cache.setNoCache(true);
            cache.setCachePublic(true);
        }
        registration.setCacheControl(cache.toHttpCacheControl());
    }

    public String[] getHandler() {
        return handler;
    }

    public void setHandler(String[] handler) {
        this.handler = handler;
    }

    public String[] getLocations() {
        return locations;
    }

    public void setLocations(String[] locations) {
        this.locations = locations;
    }

    public ResourceProperties.Cache.Cachecontrol getCache() {
        return cache;
    }

    public void setCache(ResourceProperties.Cache.Cachecontrol cache) {
        this.cache = cache;
    }
}

```

外部配置：

```yaml
service:
  resources:
    handler: /static/**
    locations: classpath:/demo/
    cache:
      max-age: 15
      cache-public: true
```

