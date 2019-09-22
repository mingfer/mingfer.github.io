---
title: "gRPC 服务 Protocol Buffers 语法教程 • 一"
subtitle: "如何定义一个 Message "
date: 2019-09-18 20:00:00
author: "mingfer"
header-img: "img/post/bg-1.jpg"
catalog: true
tags: 
    - "gRPC"
    - "网络通讯"
---

> 默认情况下，gRPC 使用 Protocol Buffers 服务定义语言（IDL）来描述服务接口和请求响应消息的结构。

## 前言

Google 在其开发者网站提供了关于 [Protocol Buffers](https://developers.google.com/protocol-buffers/docs/proto3) 的详细介绍，当然由于国内的原因我们往往无法访问。下面是根据官方文档整理的 proto3 版本的 Protocol Buffers 使用指南。本指南主要介绍如何通过 Protocol Buffers 语法去描述我们的服务接口和请求响应对象，以及 .proto 文件的语法和如何通过 .proto 文件生成对应语言的代码。

这里主要以 java 语言作为开发语言作为示例语言，带来不便，还请见谅。

其他教程参见：

- [Protocol Buffers 的字段类型](http://www.mingfer.cn/2019/09/19/grpc-protocol-buffers-2/)
- [Protocol Buffers 复合类型和关键字](http://www.mingfer.cn/2019/09/22/grpc-protocol-buffers-3/)

## 正文

我们先来看一个简单的 Message 定义。这里，我们假设要定义一个用于搜索的请求报文 `SearchRequest`，该报文包含一个查询字符串 `query`，指定的查询页 `page_number`，和每一页的条目数 `result_per_page`。下面是对应的 `.proto` 文件：

```protobuf
syntax = "proto3";

message SearchRequest {
  string query = 1;
  int32 page_number = 2;
  int32 result_per_page = 3;
}
```

- 第一行我们声明了使用哪个版本的 Protocol Buffers 语法，默认的情况下是使用 `proto2` 的语法。该声明必须位于文件非空行和非注释行的第一行。
- `SearchRequest` 消息定义三个字段（键值对）。每个字段对应要包含在此类消息中的数据； 每个字段都有一个名称，类型和唯一的序号。

### 指定消息字段类型

在上面的例子中我们指定了两个 Integer 字段 `page_number` 和 `result_per_page` 和一个 String 类型的字段 `query`。Protocol Buffers 允许消息为任意的[标准字段类型](#标准字段类型)，或者是枚举类型或其他自定义的数据类型。

### 分配消息字段编号

从例子中可以看到，每一个字段都指定了一个唯一的编号。这个序号用于指定在二进制编码后消息的类型（可以理解为将 5 个字节的 `query` 替换成了一个字节的 `1`），并且在使用了消息类型后不应该更改该编号。**注意**：编号 [1, 15] 在编码过程中仅使用 1 个字节编码，编号 [16, 2047] 会占用两个字节。所以，我们应该将频繁使用的字段的编号放到 [1, 15] 区间内，并且注意为以后会新增的频繁使用的字段预留空间。

编号的范围在 [1,  536870911] 区间，其中 [19000, 19999] 区间的编号作为协议的保留编号不建议使用。如果我们使用了该部分的编号，在编译 proto 文件的时候编译器会发出警告。

### 指定消息字段规则

消息字段有下面两种类型：

- 单数形式：proto3 默认的字段规则为单数形式，格式正确的消息中可以包含一个或领个该字段。
- 复数形式：通过 `repeated` 关键字可以将自定为复数形式的字段，格式正确的消息中可以包含零个或无数个该字段，并且这些会保留这些字段的添加顺序。

在 proto3 中，标准类型的复数形式字段默认使用压缩编码。

### 添加更多的消息

我们可以将多个 Message 的定义写到同一个 proto 文件中，比如说我们需要定义一些互相关联的消息的时候。下面的例子中我们在文件中添加了一个关于 `SearchRequest` 的响应消息 `SearchResponse`：

```protobuf
message SearchRequest {
  string query = 1;
  int32 page_number = 2;
  int32 result_per_page = 3;
}

message SearchResponse {
 ...
}
```

### 添加注释

Protocol Buffers 使用 C/C++ 风格的注释：`//` 或 `/* .... */`。

```protobuf
/* SearchRequest represents a search query, with pagination options to
 * indicate which results to include in the response. */

message SearchRequest {
  string query = 1;
  int32 page_number = 2;  // Which page number do we want?
  int32 result_per_page = 3;  // Number of results to return per page.
}
```

### 保留字段

如果我们在更新 proto 文件的时候通过删除不要的字段或注释掉不要的字段来更新，那么未来的用户可能会在同样的编号或字段名称下定义新的内容。这可能导致不同版本的 proto 文件中具有相同编号或字段名但是意义不一样的字段，这可能会带来数据损坏或安全性方面的严重问题。避免这种情况的方法是指定已删除的字段的编号为保留字段编号（也可以指定字段名称为保留名称，但可能引起 JSON 序列化问题），指定之后，未来的用户在这使用这些字段的时候，编译器会进行告警。

```protobuf
message Foo {
  reserved 2, 15, 9 to 11;
  reserved "foo", "bar";
}
```

**注意**：我们可以在同一个 `reserved` 语句中既指定编号，又指定字段名称。

### .proto 文件可以生成什么代码？

当我们编译 .proto 文件的时候，编译器会根据我们选择的语言生成对应的源代码。当我们需要使用我们定义的消息的时候，我们可以通过调用源代码中的 getter/setter 方法进行取值和赋值。并且编译出的代码会负责将消息对象序列化为 `OutputStream` 或从 `InputStream` 中解析出对应的消息对象。

- 对于 C++ 语言，编译器将 `.proto` 文件编译成一个 `.cc` 文件和一个 `.h` 文件，`.proto` 文件中的每一个消息都使用一个 class 进行描述。
- 对于 Java 语言，编译器将每一个消息编译成一个 `.java` 文件，并且通过构建器模式 `Builder` 对消息进行赋值和构造。
- Python 有点不同 -  Python 编译器生成一个模块，其中包含 `.proto` 中每种消息类型的静态描述符，然后与元类一起使用，以在运行时创建必要的 Python 数据访问类。
- 对于 Go ，编译器会生成一个 `.pb.go` 文件，其中包含文件中每种消息类型的类型。
- 对于 Ruby，编译器生成一个带有包含消息类型的 Ruby 模块的 .rb 文件。
- 对于 Objective-C，编译器从每个 `.proto` 生成一个 `pbobjc.h` 和 `pbobjc.m` 文件，其中包含文件中描述的每种消息类型的类。
- 对于 C＃，编译器从每个 `.proto` 生成一个 `.cs` 文件，其中包含文件中描述的每种消息类型的类。
- 对于 Dart，编译器会生成一个 `.pb.dart` 文件，其中包含文件中每种消息类型的类。

官方的 [API reference](https://developers.google.com/protocol-buffers/docs/reference/overview) 包含了各种语言 API 的更详细的介绍。

