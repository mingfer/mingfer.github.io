---
layout: post
title: "Spring Boot 外部化配置"
subtitle: "如何实现数据库密码密文配置"
date: 2019-03-30 20:00:00
author: "mingfer"
header-img: "img/post/spring-boot-bg.jpg"
catalog: true
tags: 
    - "Spring Boot"
    - "数据库"
	- "Web 后端"
---

> Spring Boot 主推"约定优于配置"的原则，减少了大量的 XML 配置。但是在实际应用的过程中，出现自定义配置参数的需要是无法避免的。



## 前言

关于 Spring Boot 配置的说明在 Spring Boot 官方文档 [24. Externalized Configuration](https://docs.spring.io/spring-boot/docs/2.1.2.RELEASE/reference/htmlsingle/#boot-features-external-config) 已经做了详细的说明。这里主要想讨论的是如何通过自定义的配置来解决一些具体的问题。

关于本文的出现的所有代码均可以在 [demo-configuration](https://github.com/mingfer/demo/tree/master/demo-configuration) 里面找到。

关于 Spring Boot 的自定义属性配置，这里主要讨论：如何实现数据库密码密文配置？

----

## 正文

### Spring Boot 配置概述

一般来说，我们使用 [YAML](https://en.wikipedia.org/wiki/YAML) 格式的配置文件来进行外部参数配置。在没有定制化的前提下，Spring Boot 的配置文件会放在 `src/main/resources` 或 `src/test/resources` 目录下，并且文件名称为 `application.yml` 。

#### 使用 @ConfigurationProperties 访问属性

很多时候我们可能需要拿到我们的配置属性做一些判断或业务上的操作，Spring Boot 提供了非常便捷的操作给我们访问这些属性。

Spring Boot 中，在配置文件 `application.yml` 中配置如下属性：

```yaml
mingfer:
  test:
    name: mingfer
    email: mingfer.cn@gmail.com
```

我们可以通过注解 `@ConfigurationProperties` 将配置信息注入到 Java 类中：

```java
@Data
@Configuration
@ConfigurationProperties("mingfer.test")
public class MingferConfiguration {

    private String name;

    private String email;
}
```

测试一下：

```java
@RunWith(SpringRunner.class)
@SpringBootTest(classes = Application.class)
public class MingferConfigurationTest {

    @Autowired
    private MingferConfiguration configuration;

    @Test
    public void testGetProperties() {
        assertNotNull(configuration);
        assertEquals("mingfer", configuration.getName());
        assertEquals("mingfer.cn@gmail.com", configuration.getEmail());
    }

}
```

#### 使用 @Value 访问属性

相较于注解  `@ConfigurationProperties` 而言，`@Value` 提供了更细腻的操作。它可以为单独的成员属性绑定一个外部配置的属性，并且在外部没有配置属性的时候还可以指定默认值。

通过 `@Value` 注入属性信息，这里沿用上面配置文件中配置的 `name` 和 `email` 值，但是没有配置 `password` 的值，而是使用 `@Value` 指定了一个默认值 `123456`：

```java
@Data
@Configuration
public class MingferConfigurationForValue {

    @Value("${mingfer.test.name}")
    public String name;

    @Value("${mingfer.test.email}")
    public String email;

    @Value("${mingfer.test.password:123456}")
    public String password;

}
```

测试一下：

```java
@RunWith(SpringRunner.class)
@SpringBootTest(classes = Application.class)
public class MingferConfigurationForValueTest {


    @Autowired
    private MingferConfigurationForValue configuration;

    @Test
    public void testGetProperties() {
        assertNotNull(configuration);
        assertEquals("mingfer", configuration.getName());
        assertEquals("mingfer.cn@gmail.com", configuration.getEmail());
        assertEquals("123456", configuration.getPassword());
    }
}
```

#### 如何对注入的配置值进行校验

出于对安全或其他方面的考虑，我们可能需要对外部配置的值进行校验。Spring Boot 允许在使用 `@ConfigurationProperties` 时使用 Spring 的 `@Validated` 验证配置的值。**注意**：`@Value`  和 `@Validated` 无法一起配合使用。

我们配置一个错误的 `phone-number`：

```yaml
mingfer:
  test:
    phone-number: 123456
```

设定 `phone-number` 的长度必须为 11 字节，这里所有类似于 `@Size` 这种用于约束的注解均来自符合 JSR-303 规范的 `javax.validation` 下的注解：

```java 

@Data
@Validated
@Configuration
@ConfigurationProperties("mingfer.test")
public class MingferConfigurationForValidate {

    @Size(min = 11, max = 11)
    private String phoneNumber;
}
```

测试一下，这里会抛出一个 `BindValidationException` 引起的异常，因为是启动时的异常，所以 `@Test` 注解的方法无法抓取到这个异常：

```java

@RunWith(SpringRunner.class)
@SpringBootTest(classes = Application.class)
public class MingferConfigurationForValidateTest {

    @Autowired
    private MingferConfigurationForValidate configuration;

    @Rule
    public ExpectedException expectedException = ExpectedException.none();

    @Test
    public void testGetProperties() {
        expectedException.expect(BindValidationException.class);
        expectedException.expectMessage("Binding validation errors on mingfer.test");
    }

}
```

#### @ConfigurationProperties vs. @Value

关于 `@ConfigurationProperties` 和 `@Value` 的对比，这里援引 Spring Boot 官方的对比。

`@Value` 是容器的核心功能，它不能像 `@ConfigurationProperties` 一样提供类型安全的配置功能，下表是对 `@Configuration` 和 `@Value` 的对比：

| 功能                                                         | @ConfigurationProperties | @Value |
| ------------------------------------------------------------ | ------------------------ | ------ |
| [Relaxed binding](https://docs.spring.io/spring-boot/docs/2.1.2.RELEASE/reference/htmlsingle/#boot-features-external-config-relaxed-binding) | YES                      | NO     |
| [Meta-data support](https://docs.spring.io/spring-boot/docs/2.1.2.RELEASE/reference/htmlsingle/#configuration-metadata) | YES                      | NO     |
| `SpEL` evaluation                                            | NO                       | YES    |

如果我们自定义了一组属性，那么这里推荐使用 `@ConfigurationProperties` 注解去注入这些属性值。必须注意到的是，`@Value` 并不能支持属性名模糊匹配(如：不区分大小写，不区分 `-` 和驼峰)，所以在注入外部的配置值的时候，它不见得是一个很好的选择。

### 如何实现数据库密码密文配置

 Spring Boot 的 `spring.datasource.password` 属性提供了数据库密码的配置，但是遗憾的是这里只能配置明文值。出于安全的考虑，有时候不能把数据库的密码明文暴露出来，那么如何让  `spring.datasource.password` 支持密文属性的配置呢？

要解决这个问题，需要克服下面三个点：

- 需要实现数据库密码的加解密方法
- 如何让  `spring.datasource.password`  属性区分填入的值是明文值还是密文值
- 如何在建立数据库连接的时候让 `DataSource` 使用到数据库明文

#### 简单的密码加解密方法

这里使用 DES 对称算法作为数据库密码的加解密算法，当然在实际生产中尽量不要使用该算法，该算法已经不是一个安全的算法了。

加解密方法：

```java

/**
 * 数据库密码加解密工具
 */
public class DataBaseCipher {

    private final static byte[] key = new byte[]{0x25, 0x38, 0x31, 0x53, 0x67, 0x13, 0x42, 0x04};
    private final static byte[] iv = new byte[]{0x75, 0x08, 0x37, 0x53, 0x67, 0x13, 0x22, 0x44};
    static byte[] doCipher(int opmode, byte[] data) {
        try {
            final SecretKey secretKey = new SecretKeySpec(key, "DES");
            final IvParameterSpec parameterSpec = new IvParameterSpec(iv);
            final Cipher cipher = Cipher.getInstance("DES/CBC/PKCS5Padding");
            cipher.init(opmode, secretKey, parameterSpec);
            return cipher.doFinal(data);
        } catch (Exception e) {
            throw new IllegalStateException("Database cipher failed.", e);
        }
    }

    /**
     * 加密明文数据
     *
     * @param password 明文密码值
     * @return Base64 编码的密文值
     */
    public static String encrypt(String password) {
        return Base64.getEncoder().encodeToString(doCipher(Cipher.ENCRYPT_MODE, password.getBytes()));
    }

    /**
     * 解密密文数据
     *
     * @param cipherText BASE64 编码的密文数据
     * @return 明文密码值
     */
    public static String decrypt(String cipherText) {
        return new String(doCipher(Cipher.DECRYPT_MODE, Base64.getDecoder().decode(cipherText)));
    }
}
```

测试一下：

```java

public class DataBaseCipherTest {

    @Test
    public void encrypt() {
        assertEquals("kzdpDr6Jmqg=", DataBaseCipher.encrypt("123456"));
    }

    @Test
    public void decrypt() {
        assertEquals("123456", DataBaseCipher.decrypt("kzdpDr6Jmqg="));
    }
}
```

#### 区分明文和密文值

这里定义了一套规则来区分 `spring.datasource.password` 字段配置的是明文值还是密文值，我们约定：

- 使用 `ENC(cipherText)` 括号中的是 Base64 编码的密码密文字符串
- 使用 `CLR(plaintext)` 括号中的是明文数据库密码
- 默认直接填写的数据为数据库密码明文，当然这里默认为明文还是密文可以根据具体的实现去选择

在 `application.yml` 配置一个数据源，并将数据源的密码设置为密文值：

```yaml
spring:
  datasource:
    driver-class-name: com.mysql.jdbc.Driver
    type: com.zaxxer.hikari.HikariDataSource
    url: jdbc:mysql://localhost:3306/first?useUnicode=true&characterEncoding=utf-8&autoReconnect=true
    password: ENC(kzdpDr6Jmqg=)
    username: root
```

编写一个属性值的解析器：

```java

/**
 * 配置参数值解析器
 */
@Data
public class PropertyResolver {

    private final static String CLR_PREFIX = "CLR(";
    private final static String ENC_PREFIX = "ENC(";
    private final static String COMMON_SUFFIX = ")";

    private final String value;

    public PropertyResolver(String value) {
        if (value.startsWith(CLR_PREFIX) && value.endsWith(COMMON_SUFFIX)) {
            this.value = value.substring(CLR_PREFIX.length(), value.length() - 1);
        } else if (value.startsWith(ENC_PREFIX) && value.endsWith(COMMON_SUFFIX)) {
            this.value = DataBaseCipher.decrypt(value.substring(ENC_PREFIX.length(), value.length() - 1));
        } else {
            this.value = value;
        }
    }
}
```

测试一下：

```java
public class PropertyResolverTest {

    @Test
    public void getValue() {
        PropertyResolver resolver = new PropertyResolver("ENC(kzdpDr6Jmqg=)");
        assertEquals("123456", resolver.getValue());
        resolver = new PropertyResolver("CLR(123456)");
        assertEquals("123456", resolver.getValue());
        resolver = new PropertyResolver("123456");
        assertEquals("123456", resolver.getValue());
    }
}
```

#### 如何让密文在 `DataSource` 生效

Spring Boot 是通过 `DataSource` 和数据库建立连接的，所以只要在注入 `DataSource` Bean 的时候将解密号的数据库密码设置到 Bean 里面 `DataSource` 就可以和数据库成功建立连接了。

更改解密数据库密码并自定义 `DataSource` Bean：

```java

@Data
@Configuration
@ConfigurationProperties("spring.datasource")
public class DataBaseConfiguration {

    private String driverClassName;

    private String url;

    private String username;

    private String password;

    @Bean
    public DataSource dataSource() {
        final HikariDataSource dataSource = new HikariDataSource();
        dataSource.setJdbcUrl(url);
        dataSource.setUsername(username);
        dataSource.setPassword(new PropertyResolver(password).getValue());
        return dataSource;
    }
}
```

通过运行 `Application.java` 测试数据库是否连接成功。



