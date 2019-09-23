---
title: "gRPC 服务 Protocol Buffers 语法教程 • 二"
subtitle: "Protocol Buffers 的字段类型"
date: 2019-09-19 20:00:00
author: "mingfer"
header-img: "img/post/bg-1.jpg"
catalog: true
tags: 
    - "gRPC"
    - "网络通讯"
---

> Protocol Buffers 作为一种独立的编程语言，它有着自己的数据类型。

## 前言

本文主要说明 Protocol Buffers  的数据类型，其他教程参见：

- [如何定义一个 Message](http://www.mingfer.cn/2019/09/18/grpc-protocol-buffers-1/)
- [Protocol Buffers 复合类型和关键字](http://www.mingfer.cn/2019/09/22/grpc-protocol-buffers-3/)

## 正文

### 原生数据类型

一个原生的 Protocol Buffers 字段可能包含下表的数据类型之一。下表主要是 Protocol Buffers 语言和其他语言数据类型之间对应关系，也就是说 .proto 文件编译成相对应的编程语言之后，Protocol Buffers 的字段类型会变成该编程语言对应的字段类型。

| .proto Type | Notes                                                        | C++ Type | Java Type  | Python Type[2] | Go Type | Ruby Type                      | C# Type    | PHP Type          | Dart Type |
| :---------- | :----------------------------------------------------------- | :------- | :--------- | :------------- | :------ | :----------------------------- | :--------- | :---------------- | :-------- |
| double      |                                                              | double   | double     | float          | float64 | Float                          | double     | float             | double    |
| float       |                                                              | float    | float      | float          | float32 | Float                          | float      | float             | double    |
| int32       | Uses variable-length encoding. Inefficient for encoding negative numbers – if your field is likely to have negative values, use sint32 instead. | int32    | int        | int            | int32   | Fixnum or Bignum (as required) | int        | integer           | int       |
| int64       | Uses variable-length encoding. Inefficient for encoding negative numbers – if your field is likely to have negative values, use sint64 instead. | int64    | long       | int/long[3]    | int64   | Bignum                         | long       | integer/string[5] | Int64     |
| uint32      | Uses variable-length encoding.                               | uint32   | int[1]     | int/long[3]    | uint32  | Fixnum or Bignum (as required) | uint       | integer           | int       |
| uint64      | Uses variable-length encoding.                               | uint64   | long[1]    | int/long[3]    | uint64  | Bignum                         | ulong      | integer/string[5] | Int64     |
| sint32      | Uses variable-length encoding. Signed int value. These more efficiently encode negative numbers than regular int32s. | int32    | int        | int            | int32   | Fixnum or Bignum (as required) | int        | integer           | int       |
| sint64      | Uses variable-length encoding. Signed int value. These more efficiently encode negative numbers than regular int64s. | int64    | long       | int/long[3]    | int64   | Bignum                         | long       | integer/string[5] | Int64     |
| fixed32     | Always four bytes. More efficient than uint32 if values are often greater than 228. | uint32   | int[1]     | int/long[3]    | uint32  | Fixnum or Bignum (as required) | uint       | integer           | int       |
| fixed64     | Always eight bytes. More efficient than uint64 if values are often greater than 256. | uint64   | long[1]    | int/long[3]    | uint64  | Bignum                         | ulong      | integer/string[5] | Int64     |
| sfixed32    | Always four bytes.                                           | int32    | int        | int            | int32   | Fixnum or Bignum (as required) | int        | integer           | int       |
| sfixed64    | Always eight bytes.                                          | int64    | long       | int/long[3]    | int64   | Bignum                         | long       | integer/string[5] | Int64     |
| bool        |                                                              | bool     | boolean    | bool           | bool    | TrueClass/FalseClass           | bool       | boolean           | bool      |
| string      | A string must always contain UTF-8 encoded or 7-bit ASCII text, and cannot be longer than 232. | string   | String     | str/unicode[4] | string  | String (UTF-8)                 | string     | string            | String    |
| bytes       | May contain any arbitrary sequence of bytes no longer than 232. | string   | ByteString | str            | []byte  | String (ASCII-8BIT)            | ByteString | string            | List<int> |

- [1] 在 Java 中无符号和有符号的 32 bit 和 64 bit 整型使用同一数据类型标识（int 或 long），Java 中首个二进制位作为符号位。
- [2] 在所有的情况下，向对应类型的字段赋值的时候都会执行类型检查以确保有效性。
- [3] 64 字节的整型和无符号 32 字节的整型在解析时始终为 long 型，但是在对该字段赋值的时候可以赋值为整型。
- [4] Phython 在解析字符串的时候以 Unicode 编码解析，也可以指定为 ASCII，但是这样有可能改变字符串的实际值。
- [5] 64 位的机器上使用 Integer 类型，32 位的机器上使用 String 类型。

### 默认值

在解析 Message 的字段的时候，如果字段没有显式的进行赋值，那么该字段将使用对应类型的默认值：

- 对于 String 类型，默认为空值 `""`
- 对于字节数组，默认值为长度为零的空数组
- 对于布尔类型，默认为 false
- 对于数字相关的类型，默认值为 0
- 对于枚举类型，默认值为该枚举的第一个类型，该类型值必须为 0
- 对于 Message 类型，其中得到字段不会被赋值，它的实际值取决于具体语言的具体实现。

对于复数形式的字段，默认值为空值，一般来说是一个长度为零的 list。

**注意**：对于标准数据类型，一旦 Message 被解析之后就无法确认字段值是被赋值为默认值的还是没有赋值所以采用了默认值。我们在定义消息类型的时候需要注意着两种情况的不同，比如有时候我们希望没有赋值的时候做一些处理，为默认值的时候做另外一些处理。另外一点是，为默认值的字段将不会被序列化发送到对端，因为在没有值的时候对端会自行赋值为默认值。

### 枚举类型

当我们定义某些字段的时候，我们可能希望该字段的值是我们预定义的值列表中的一个。比如说下面的 SearchRequest ，我们希望 corpus 的取值是 `UNIVERSAL`, `WEB`, `IMAGES`, `LOCAL`, `NEWS`, `PRODUCTS` 或 `VIDEO` 中的其中一种。此时我们可以通过 `enum` 关键字来定义一组我们预期的值。下面是一个 `enum` 类型 `Corpus` 和它可能的值：

```protobuf
message SearchRequest {
  string query = 1;
  int32 page_number = 2;
  int32 result_per_page = 3;
  enum Corpus {
    UNIVERSAL = 0;
    WEB = 1;
    IMAGES = 2;
    LOCAL = 3;
    NEWS = 4;
    PRODUCTS = 5;
    VIDEO = 6;
  }
  Corpus corpus = 4;
}
```

如上所示，枚举类型 `Corpus` 的第一个值是 `0`，Protocol Buffers  规定枚举类型必须使用 `0` 作为它第一个值的常量值。原因是：

- 方便我们可以使用 0 作为数字类型的默认值，可以把枚举看成一种特殊的数字类型。
- 便于和 `proto2` 兼容，`proto2` 使用第一个值作为枚举类型的默认值。

当我们将 `allow_alias` 选项置为 `true` 的时候，我们可以为不同名称的枚举值设定相同的常量值。

```protobuf
enum EnumAllowingAlias {
  option allow_alias = true;
  UNKNOWN = 0;
  STARTED = 1;
  RUNNING = 1;
}
enum EnumNotAllowingAlias {
  UNKNOWN = 0;
  STARTED = 1;
  // RUNNING = 1;  // 没有开启 allow_alias 会造成编译错误
}
```

枚举值常量值的必须是一个 32 bit 的整型值，由于枚举值在编码时使用 varint 编码，所以不建议使用负数，以免降低编码效率。我们可以在一个 Message 内部定义枚举类型，也可以定义和 Message 平级的枚举类型，这些枚举类型可以在该 `.proto` 文件的任何 Message 中使用。我们可以通过语法 MessageType.EnumType 在一个 Message 中使用另一个 Message 定义的枚举类型。

#### 保留值

如果我们在更新 proto 文件的时候通过删除不要的字段或注释掉不要的字段来更新，那么未来的用户可能会在同样的编号或字段名称下定义新的内容。这可能导致不同版本的 proto 文件中具有相同编号或字段名但是意义不一样的字段，这可能会带来数据损坏或安全性方面的严重问题。避免这种情况的方法是指定已删除的字段的编号为保留字段编号（也可以指定字段名称为保留名称，但可能引起 JSON 序列化问题），指定之后，未来的用户在这使用这些字段的时候，编译器会进行告警。

```protobuf
enum Foo {
  reserved 2, 15, 9 to 11, 40 to max;
  reserved "FOO", "BAR";
}
```

**注意**：我们可以在同一个 `reserved` 语句中既指定编号，又指定字段名称。我们可以使用 `max` 关键字指定枚举常量值范围内的最大值。

### 使用 Message 类型

我们可以在指定 Message 的字段为其他的 Message 类型。比如说下面的示例中：我们假设每一个 `SearchResponse` Message 可能或包含多个 `Result` Message。我们可以在同一个 `.proto` 文件中定义 `Result` 然后在 `SearchResponse` 中使用它。

```protobuf
message SearchResponse {
  repeated Result results = 1;
}

message Result {
  string url = 1;
  string title = 2;
  repeated string snippets = 3;
}
```

#### 导入其他文件中的类型

在上面的例子中，`SearchResponse` 需要使用的类型 `Result` 和它在同一个 `.proto` 文件中。如果 `Result` 类型位于其他的 `.proto` 文件，我们该怎么使用它呢？

我们可以通过使用 `import` 语句导入其他的 `.proto` 文件来使用它：

```protobuf
import "myproject/other_protos.proto";
```

有时候我们可能会更改 `.proto` 文件的位置或文件名，这种情况下去一个个的改 `import` 语句可能是一件非常繁琐的事情。我们可以通过一个虚拟的 `.proto` 文件将新位置的 `.proto` 文件中的内容代理给所有 `import` 了虚拟文件的 `.proto` 文件。比如：

下面是将 `old.proto` 重命名后的 `new.proto`

```protobuf
// new.proto
// All definitions are moved here
```

建立一个虚拟的 `old.proto`，使用 `import public` 代理 `new.proto` 中的所有内容。

```protobuf
// old.proto
// This is the proto that all clients are importing.
import public "new.proto";
import "other.proto";
```

在引用 `old.proto` 的 `client.proto` 中不用将 `import "old.proto";` 改为 `import "new.proto";`。

```protobuf
// client.proto
import "old.proto";
// You use definitions from old.proto and new.proto, but not other.proto
```

Protocol Buffers 编译器搜索 `import` 的文件是根据 `-I` 或 `--proto_path` 指定的目录去搜索的，如果没有指定目录那么编译器会从编译器目录下寻找。通常 `--proto_path` 应该指定为项目所在的目录，并对所有的导入使用全限定名称。

#### 使用 proto2 版本的定义

proto2 和 proto3 定义的消息类型是可以相互导入使用的。但是不能在 proto3 版本的语法中直接使用 proto2 版本的枚举类型，proto2 本身使用 proto2 的枚举是可以的。

### 类型嵌套

我们可以在一个 Message 中嵌套定义一个新的 Message：

```protobuf
message SearchResponse {
  message Result {
    string url = 1;
    string title = 2;
    repeated string snippets = 3;
  }
  repeated Result results = 1;
}
```

如果我们要在其他的 Message 中使用上面的 `Result` 类型：

```protobuf
message SomeOtherMessage {
  SearchResponse.Result result = 1;
}
```

我们可以根据自己的需要深层嵌套 Message ：

```protobuf
message Outer {                  // Level 0
  message MiddleAA {  // Level 1
    message Inner {   // Level 2
      int64 ival = 1;
      bool  booly = 2;
    }
  }
  message MiddleBB {  // Level 1
    message Inner {   // Level 2
      int32 ival = 1;
      bool  booly = 2;
    }
  }
}
```

### 未知字段

未知字段是指在格式正确的序列化数据中，解析器无法识别的字段。当一个使用一个旧的 Message 去解析新的 Message 中的新字段的时候，这些新的字段会成为旧的 Message 中的未知字段。

在最初的版本中，proto3 在消息解析的过程中直接丢弃未知字段。但是在 3.5 的版本中，保留了未知字段以便于兼容 proto2。在 3.5 及更高的版本中，未知字段会被解析器保留并包含在序列化输出中。

### Any 类型

`Any` 消息类型可以让我们在 Message 中使用其他的 Message 类型，而无需导入它的 `.proto` 文件。一个 `Any` 类型包含了任意的序列化消息字节数组和解析该消息类型的全局唯一的 URL 标识。使用 `Any` 的时候，我们需要导入 `google/protobuf/any.proto` 文件：

```protobuf
import "google/protobuf/any.proto";

message ErrorStatus {
  string message = 1;
  repeated google.protobuf.Any details = 2;
}
```

 `Any` 类型指定的消息类型的 URL 格式为 `type.googleapis.com/packagename.messagename`。

不同的语言实现了不同的帮助程序以类型安全的形式解包和打包 `Any` 类型。比如在 Java 中 `Any` 类型提供了 `pack()` 和 `unpack()` 方法，而在 C++ 中提供了 `PackFrom()` 和 `UnpackTo()` 函数：

```protobuf
// Storing an arbitrary message type in Any.
NetworkErrorDetails details = ...;
ErrorStatus status;
status.add_details()->PackFrom(details);

// Reading an arbitrary message from Any.
ErrorStatus status = ...;
for (const Any& detail : status.details()) {
  if (detail.Is<NetworkErrorDetails>()) {
    NetworkErrorDetails network_error;
    detail.UnpackTo(&network_error);
    ... processing network_error ...
  }
}
```

**目前，该帮助库还在开发阶段。**