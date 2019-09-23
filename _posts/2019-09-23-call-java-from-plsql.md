---
title: "如何通过 PL/SQL 调用 Java 方法"
date: 2019-09-23 9:00:00
author: "mingfer"
header-img: "img/post/bg-1.jpg"
catalog: true
tags: 
    - "数据库"
    - "PL/SQL"
---

## 前言

当我们加载并发布了使用 Java 实现的存储过程（包括过程和函数）之后，我们可以通过 SQL 语句调用它们。本文将说明如何在各自的上下文中 SQL 如何去调用 Java 实现的存储过程，以及在 SQL 中如何处理 Java 存储过程中产生的异常。

相关文章参见：

- [PL/SQL 初探](http://www.mingfer.cn/2019/09/19/PLSQL/)

## Hello World

我们首先通过一个简单的打印 Hello World 字符串的过程和函数来说明如何使用 Java 存储过程。在下面的 Java 代码中我们实现了两个不同的用于打印 Hello World 的方法。

```java
public class HelloWorld {

    // 过程
    public static void sayHelloWorld() {
        System.out.println("Hello World.");
    }

    // 函数
    public static String sayHelloWorld(String text) {
        System.out.println("Hello World.");
        return text;
    }

}
```

### 重定向输出

在 Oracle 数据库中默认的输出设备是一个 `trace file` 而不是用户屏幕。也就是说 Java 中 `System.out` 和 `System.err` 的输出会输出到当前的 `trace file` 中。我们可以重定向输出到当前屏幕，并通过 `dbms_java.set_output()` 设置缓冲区大小：

```sql
SQL> SET SERVEROUTPUT ON
SQL> CALL dbms_java.set_output(2000);
```

缓冲区的默认大小和最小值均为 2000 字节，最大值为 1,000,000 字节。我们可以通过如下方式更改缓冲区的大小：

```sql
SQL> SET SERVEROUTPUT ON SIZE 5000
SQL> CALL dbms_java.set_output(5000);
```

配置之后当存储过程运行完毕输出将会打印到屏幕上面。

上面是使用 `sqlplus` 命令行工具的示例，如果使用 Oracle SQL Developer 工具，可以在当前连接下输入上面的代码并执行这些代码：

![1569207655476](/img/post/1569207655476.png)

如果需要打印 `System.out` 和 `System.err` 输出的内容，需要将 【查看】--> 【DBMS 输出】连接到当前数据库。

![1569216787385](/img/post/1569216787385.png)

**注意**：所有过程或函数需要在 【SQL 工作表】下执行，才会打印 `System.out` 和 `System.err` 输出的内容到【脚本输出】和【DBMS 输出】。

### 加载 Java 存储过程

可以使用 `loadjava` 工具将 Java 类源码，编译后的 `.class` 文件或资源文件加载到数据库实例中，它们将被存储为 `Java Schema Object`。当我们在命令行或应用中运行 `loadjava` 命令的时候，可以指定解析器在内的一些可选项。

在下面的例子中，`loadjava` 在命令行使用 `JDBC OCI` 驱动连接 Oracle 数据库实例，它需要我们在连接的时候指定用户和密码。默认情况下，导入的 Java 过程函数将存储在登录用户的 schema 下面。

```sql
$ javac HelloWorld.java
$ loadjava -user HR HelloWorld.class
Password: password
```

当我们调用 Java 存储过程中提供的方法时，解析器会去查找支持的 Java 类。默认的解析器会现在当前用户的 schema 中查找，然后在系统的 schema （包含所有 Java 的核心库）中查找。必要情况下，我们可以指定需要的解析器。

如果使用 Oracle SQL Developer ，直接在 Java 目录导入 Java 文件即可：

![1569209720935](/img/post/1569209720935.png)

### 发布 Java 存储过程

一般来说，无返回的 Java 方法发布为 PL/SQL 过程，有返回的 Java 方法发布为 PL/SQL 函数。在下面的例子中，我们通过 `sqlplus` 连接到 Java 存储过程所在的用户，并发布一个 Java 方法。

```sql
SQL> connect HR
Enter password: password

SQL> CREATE FUNCTION say_hello_world RETURN VARCHAR2
2 AS LANGUAGE JAVA
3 NAME 'HelloWorld.sayHelloWorld(java.lang.String) return java.lang.String';
```

如果使用 Oracle SQL Developer ，可以选择在过程和函数目录发布过程或函数，也可以在 Package 目录同时发布过程或函数。

![1569217725527](/img/post/1569217725527.png)

Package 实现：

```plsql
CREATE OR REPLACE 
PACKAGE PACKAGE_HELLO_WORLD AS 

  /* TODO enter package declarations (types, exceptions, methods etc) here */ 
  PROCEDURE say_hello_world;
  
  FUNCTION say_hello_world(TEXT IN VARCHAR2) RETURN VARCHAR2;
  
END PACKAGE_HELLO_WORLD;

-- package body
CREATE OR REPLACE
PACKAGE BODY PACKAGE_HELLO_WORLD AS

  PROCEDURE say_hello_world AS
    LANGUAGE JAVA NAME 'HelloWorld.sayHelloWorld()';

  FUNCTION say_hello_world(TEXT IN VARCHAR2) RETURN VARCHAR2 AS
    LANGUAGE JAVA NAME 'HelloWorld.sayHelloWorld(java.lang.String) return java.lang.String';

END PACKAGE_HELLO_WORLD;
```

过程实现：

```plsql
CREATE OR REPLACE PROCEDURE PROC_SAY_HELLO_WORLD AS 
LANGUAGE JAVA NAME 'HelloWorld.sayHelloWorld()';
```

函数实现：

```plsql
CREATE OR REPLACE FUNCTION FUNC_SAY_HELLO_WORLD 
(
  TEXT IN VARCHAR2 
) RETURN VARCHAR2 AS 
LANGUAGE JAVA NAME 'HelloWorld.sayHelloWorld(java.lang.String) return java.lang.String';
```

## 异常处理

Java 异常是一个包含名称和继承关系的对象，也就是说我们可以把子类异常对象转换为他的父类异常。所有的 Java 异常都实现了 `toString()` 方法，该方法会返回异常类的全限定名称和相关的异常信息，该信息通常在异常的构造函数中填写。

当在 SQL 语句中运行 Java 存储过程的时候，所有该存储过程抛出的异常都应该是 `java.sql.SQLException` 的子类。这个类提供了 `getErrorCode()` 和 `getMessage()` 方法，用于返回 Oracle 数据库的错误码和错误信息。

如果在 SQL 或 PL/SQL 调用的 Java 存储过程抛出了未被抓取的异常，Oracle 将提示下面的错误信息：

```
ORA-29532 Java call terminated by uncaught Java exception
```

这就是提示所有未抓取的异常，包括非 `java.sql.SQLException` 子类异常的方式。

