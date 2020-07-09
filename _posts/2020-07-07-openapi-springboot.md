---
layout: post
title: "使用 OpenAPI3 快速生成 REST API 文档"
date: 2020-07-07  21:29:00
author: "mingfer"
header-img: "img/post/bg-1.jpg"
catalog: true
tags: 
    - "OpenAPI"
    - "Spring Boot"
    - "Web 后端"
---

[springdoc-openapi](https://github.com/springdoc/springdoc-openapi) 是一个在 Spring Boot 项目中快速生成 API 文档的 Java 库。[springdoc-openapi](https://github.com/springdoc/springdoc-openapi) 通过在运行时检查检查程序中的 Spring 配置，类的定义和各种注解来推导 API 的语义。

## 集成

默认的情况下，我们需要先将 [springdoc-openapi](https://github.com/springdoc/springdoc-openapi) 添加到项目依赖列表中：
```xml
<dependency>
  <groupId>org.springdoc</groupId>
  <artifactId>springdoc-openapi-ui</artifactId>
  <version>1.4.3</version>
</dependency>
```
然后启动应用，打开地址 [http://server:port/swagger-ui.html](http://server:port/context-path/swagger-ui.html) 就可以访问到 API 接口的 UI 页面了。

## 配置

[springdoc-openapi](https://github.com/springdoc/springdoc-openapi) 支持通过 Spring Boot 的 Bean 对文档的一些元数据进行配置。下面是两个常用的配置，这些 Bean 使用 `@Bean` 注解声明在任何一个 `@Configuration` 注解的配置类中即可：

使用 `GroupedOpenApi` 对 API 进行分组，防止所有 API 生成在一个页面中，导致阅读体验很差。

```java
@Bean
public GroupedOpenApi actuatorApi() {
    return GroupedOpenApi.builder()
            .group("Actuator")
            .pathsToMatch("/actuator/**")
            .pathsToExclude("/actuator/health/*")
            .build();
}

```

使用 `OpenAPI` 配置 API 的元数据信息：

```java
@Bean
public OpenAPI customOpenAPI(BuildProperties buildProperties) {
    return new OpenAPI()
            .components(new Components()
                .addSecuritySchemes("basicScheme", new SecurityScheme()
                    .type(SecurityScheme.Type.HTTP).scheme("basic")))
            .info(new Info()
                .title("Petstore API")
                .version(buildProperties.getVersion())
                .description(
                    "This is a sample server Petstore server. You can find out more about Swagger at [http://swagger.io](http://swagger.io) or on [irc.freenode.net, #swagger](http://swagger.io/irc/). For this sample, you can use the api key `special-key` to test the authorization filters.")
                    .termsOfService("http://swagger.io/terms/")
                    .license(new License().name("Apache 2.0").url("http://springdoc.org")));
}

```

注意 `BuildProperties` 进行下面的配置才能生成对于的配置文件 `build-info.properties`，在 Spring Boot 中该文件的默认地址是 `classpath:META-INF/build-info.properties`：

Maven:

```xml
<plugin>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-maven-plugin</artifactId>
    <executions>
        <execution>
            <id>build-info</id>
            <goals>
                <goal>build-info</goal>
            </goals>
        </execution>
    </executions>
</plugin>
```

Gradle:

```groovy
springBoot {
    buildInfo()
}
```

## 使用注解声明 API

这里通过一个 Controller 类，一个 UserEntity 来演示如何通过注解完成文档的编写。代码的逻辑很诡异，但这个不是重点，重点是如何使用 Swagger3 的相关注解声明 API 接口。顺带提一下，这些注解均是遵循规范 [OpenAPI-Specification](https://github.com/OAI/OpenAPI-Specification/blob/master/versions/3.0.3.md) 实现的，所以我们可以通过阅读该规范来了解如何使用这些注解。


UserController.java

```java
package cn.mingfer.demo;

import io.swagger.v3.oas.annotations.Operation;
import io.swagger.v3.oas.annotations.Parameter;
import io.swagger.v3.oas.annotations.Parameters;
import io.swagger.v3.oas.annotations.enums.ParameterIn;
import io.swagger.v3.oas.annotations.media.Content;
import io.swagger.v3.oas.annotations.media.Schema;
import io.swagger.v3.oas.annotations.responses.ApiResponse;
import io.swagger.v3.oas.annotations.responses.ApiResponses;
import io.swagger.v3.oas.annotations.tags.Tag;
import org.springframework.web.bind.annotation.*;

import java.util.ArrayList;
import java.util.List;

@RestController
@RequestMapping("user")
@Tag(name = "user", description = "用户信息接口")
public class UserController {


    @PostMapping("test")
    @Operation(description = "用户信息测试")
    @Parameters({
            @Parameter(name = "uid", description = "用户 ID", required = true, in = ParameterIn.PATH),
            @Parameter(name = "name", description = "用户名称", required = true)
    })
    @ApiResponses({
            @ApiResponse(responseCode = "200", description = "操作成功", content = {
                    @Content(mediaType = "application/json", schema = @Schema(implementation = UserEntity.class, type = "array"))
            }),
            @ApiResponse(responseCode = "400", description = "参数错误", content = {
                    @Content(mediaType = "application/json", schema = @Schema())
            })
    })
    public List<UserEntity> test(@PathVariable("uid") String uid,
                       @RequestParam("name") String name,
                       @RequestBody UserEntity entity) {
        List<UserEntity> entities = new ArrayList<>();
        entities.add(new UserEntity(uid, name));
        entities.add(entity);
        return entities;
    }

}
```

UserEntity.java

```java
package cn.mingfer.demo;

import com.fasterxml.jackson.annotation.JsonProperty;
import io.swagger.v3.oas.annotations.media.Schema;

@Schema(description = "用户信息实体")
public class UserEntity {

    @Schema(description = "用户 ID", type = "string", required = true, minLength = 4, maxLength = 12, example = "10086")
    private final String uid;

    @Schema(description = "用户名称", required = true, example = "mingfer.cn")
    private final String name;

    public UserEntity(@JsonProperty("uid") String uid,
                      @JsonProperty("name") String name) {
        this.uid = uid;
        this.name = name;
    }

    public String getUid() {
        return uid;
    }

    public String getName() {
        return name;
    }
}

```

