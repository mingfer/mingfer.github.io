---
layout: post
title: "gRPC 服务 Protocol Buffers 语法教程 • 三"
subtitle: "Protocol Buffers 复合类型和关键字"
date: 2019-09-22 20:00:00
author: "mingfer"
header-img: "img/post/bg-1.jpg"
catalog: true
tags: 
    - "gRPC"
    - "网络通讯"
---

## 前言

本教程用于说明 Protocol Buffers 中一些关键字的作用和用法，其他教程参见：

- [如何定义一个 Message](http://www.mingfer.cn/2019/09/18/grpc-protocol-buffers-1/)
- [Protocol Buffers 的字段类型](http://www.mingfer.cn/2019/09/19/grpc-protocol-buffers-2/)

## 正文

### Oneof

当我们的 Message 中包含多个字段但是只允许其中一个字段生效的时候可以使用 `oneof` 关键字，这样在编码得到时候会节省一部分内存。

`oneof` 内声明字段和常规声明字段得到方式类似，但是需要注意 `oneof` 里面的字段使用的编号适合外部的字段编号同级的，这里需要注意不要让编号冲突。`oneof` 中的多个字段只允许我们生效其中一个，这意味着当我们设置其中一个字段的时候，都会清除掉其他所有的成员。我们可以使用 `case()` 或 `WhichOneof` 方法去获取被赋值的字段，具体使用哪个方法取决于具体的编程语言。

#### 如何使用

在 `.proto` 中定义一个 `oneof` 类型，只需要在 `oneof` 关键字后边跟上字段名称即可。示例如下：

```protobuf
message SampleMessage {
  oneof test_oneof {
    string name = 4;
    SubMessage sub_message = 9;
  }
}
```

我们可以将我们需要的字段都添加到 `oneof` 的定义里面，这些字段可以是任意类型，但不能是 `repeated` 修饰的字段。

#### Oneof 的特性

- 赋值 `oneof` 中定义的字段时会自动清除其他所有字段值，所以在我们设置了多个 `oneof` 中的字段的时候，仅最后最后设置的那个字段有值。

    ```protobuf
    SampleMessage message;
    message.set_name("name");
    CHECK(message.has_name());
    message.mutable_sub_message();   // name 字段的值会被清除
    CHECK(!message.has_name());
    ```

- `oneof` 中不能使用 `repeated` 修饰的字段。

- 当解析器遇到同一个 `oneof` 中的多个成员字段时，只有最后一个字段会被解析。

- 可以对 `oneof` 类型的字段使用反射。

- 如果将 `oneof` 中的字段设置为默认值，该字段在传输过程中会被序列化（非 `oneof` 字段的默认值是不会被序列化的）。

- 在 C++ 中需要注意内存越界的问题，下面的代码会产生内存越界，因为在调用 `set_name()` 的时候删除了 `sub_message`。

    ```protobuf
    SampleMessage message;
    SubMessage* sub_message = message.mutable_sub_message();
    message.set_name("name");      // Will delete sub_message
    sub_message->set_...            // Crashes here
    ```

- 在 C++ 中，如果使用 `Swap()` 方法交换两个包含 `oneof` 的 Message 的值，该 Message 或去获取到另一个 Message 中 Oneof 持有的值。示例如下，交换值之后 `msg1` 中的 `oneof` 字段的值为交换前 `msg2` 中的值 `sub_message`：

  ```protobuf
  SampleMessage msg1;
  msg1.set_name("name");
  SampleMessage msg2;
  msg2.mutable_sub_message();
  msg1.swap(&msg2);
  CHECK(msg1.has_sub_message());
  CHECK(msg2.has_name());
  ```

#### 兼容性问题

再添加或删除 `oneof` 字段的时候需要小心：如果 `oneof` 字段返回的值为 `None` 或 `NO_SET` 可能意味着两种情况，该字段没有被赋值或者赋值的字段为其他版本中的 `oneof` 字段。解析器无法知道字节序列中的未知字段是否是 `oneof` 中的成员，所以无法区别上边两种情况。

**标识重用的问题**：

- **向 `oneof` 中加入字段或将 `oneof` 中的字段移出**：这种情况下在消息序列化或反序列化之后我们可能会丢失一些字段信息。但是我们可以安全的将单个字段加入到一个新的 `oneof` 字段中；而且如果能够确认 `oneof` 字段只包含了一个成员，那么我们可以移入多个成员。
- **删除 `oneof` 字段后又添加回来**：在消息序列化或反序列化后可能导致当前的 `oneof` 字段被清除。
- **拆分或合并**：这种情况和第一种情况是一致的。

### Maps

我们可以通过下面的语法创建一个 Map 类型的字段：

```protobuf
map<key_type, value_type> map_field = N;
```

这里 `key_type` 可以是任意的整型和字符串类型，不允许是浮点数类型或字节数组。`value_type`可以是除 `map` 之外的任意类型。

下面的例子中我们创建了一个 `Project` 消息类型的 Map 字段，它的键值类型为 `string`：

```protobuf
map<string, Project> projects = 3;
```

- Map 类型的字段不允许使用 `repeated` 修饰。
- Map 中元素添加的顺序和序列化的顺序是不一致的，我们不应该依赖特定的添加顺序。
- 当生成 `.proto` 文本格式的 Map 的时候，Map 中的元素按照 Key 进行排序，数字类型的 Key 按数值排序。
- 当解析链路上的 Map 序列的时候，如果有两个 Key 相同的值，那么最后一个值会被使用。如果文本格式的 Key 包含两个 Key 相同的元素，将会导致序列化失败。
- 如果我们设置了 Key 却没有指定 Value 的值，序列化的时候采取的行为是依据编程语言而定的。在 C++，Java，Python 中会使用 Value 对应类型的默认值，其他语言下将不序列化该元素。

#### 兼容性

Map 类型类似于下面的 Message 类型，因此在不支持 Map 的 Protocol Buffers 版本中可以使用下面的类型代替。

```protobuf
message MapFieldEntry {
  key_type key = 1;
  value_type value = 2;
}

repeated MapFieldEntry map_field = N;
```

任何支持 Map 类型的 Protocol Buffers 实现都应该能够构造和接受上面的数据类型。

### Packages

为了防止协议间的类型冲突，我们可以在 `.proto` 中添加可选的 `package` 声明。

```protobuf
package foo.bar;
message Open { ... }
```

当我们使用 `Open` 类型的时候，可以通过包的全限定名称进行使用：

```protobuf
message Foo {
  ...
  foo.bar.Open open = 1;
  ...
}
```

`package` 的实现根据选择的编程语言的不同而不同：

- 在 C++ 中是通过命名空间实现的，上面例子中的 `Open` 的命名空间为 `foo::bar`。
- 默认情况下，Java 中使用 Java 的 package 关键字实现，除非我们使用 `option java_package` 额外指定了包名。
- 在 Python 中会忽略 `package` 声明，因为 Python 模块是根据其文件在文件系统中的位置组织的。
- 在默认情况下，Go 中使用 Go 的 package 实现，除非我们使用 `option go_package` 额外指定了包名。
- 在 Ruby 中使用 Ruby 的命名空间实现，并且会将名称自动转换为 Ruby 的大写样式（首字母大小写，如果首个字节不是字母，则使用 `PB_` 开头）。上面例子中的 `Open` 的命名空间为 `Foo::Bar`。
- 在 C# 中，除非使用 `option csharp_package` 指定了包名，否则在转换为 PascalCase 之后，package 声明的名称将用作命名空间。 上面例子中的 `Open` 将位于命名空间 `Foo.Bar` 中。

#### 包和类型名称查找

Protocol Buffer 查找类型名称的方式类似于 C++ ，会首先搜索最内层的作用域，然后搜索下一个最内层的作用域。每一个包都可以视为其父包的内层作用域。以 `.` 开始的 package 声明 `.foo.bar.Open` 表示从最外层作用域开始查找。

Protocol Buffer  通过解析导入的 `.proto` 文件来完成对类型名称的查找。每种编程语言的编译规则可能不同，但是每种语言的代码生成器都知道如何去引用该语言的每种类型。

### 定义服务 Service

如果我们希望在 RPC 系统上使用我们定义的 Message，那么我们需要在 `.proto` 文件中定义我们的 RPC Service，并将这些服务编译成对应的服务接口和 stubs。我们可以以下面的方式定义一个接收 `SearchRequest` 请求并响应 `SearchResponse` 的服务：

```protobuf
service SearchService {
  rpc Search (SearchRequest) returns (SearchResponse);
}
```

与 Protocol Buffer 一起使用的 RPC 协议一般是 gRPC 协议：这是 Google 开发的与语言和平台无关的开源 RPC 协议，它支持通过  Protocol Buffer 相关的编译器插件直接生成对应的 RPC 代码。

### Options

在 `.proto` 文件中可以通过 `option` 声明一些特定的描述，这些描述不会改变文件的整体含义，但会影响在特定的语言中处理声明的方式。这些描述的完整定义在 `google/protobuf/descriptor.proto` 中。

其中一些 `option` 是文件级别的，这些选项应该编写在最顶层的范围内，而不应该在任何的 Message，enum 或 service 中定义。有一些是 message 级别的，只能在 Message 中使用；有一些是字段级别的，只能在字段定义中使用；`option` 也可以在枚举类型，枚举值，服务类型和服务方法上使用，但是目前不存在可以在这些地方上使用的选项。

下面是一些通用的 `option`：

- `java_package` ：文件级别的 `option`，用于指定生成的 Java 类属于那一个 package 下面。为指定的情况下，会使用 `.proto` 的 package 名称作为 Java 类的包名，但是 protobuf 文件的包名通常不适合用于 Java 的包名，它们通常不以反向域名开头。如果不是生成 Java 代码，该选项不会生效。

    ```protobuf
    option java_package = "com.example.foo";
    ```

- `java_multiple_files` ：文件级别的 `option`，将 message ，enum 和 service 定义为 package 级别的类，而不是 `.proto` 文件所在 Java 类的内部类。如果不是生成 Java 代码，该选项不会生效。

    ```protobuf
    option java_multiple_files = true;
    ```

- `java_outer_classname`：文件级别的 `option`，用于指定 `.proto` 文件生成的 Java 类的名称。如果没有使用该选项进行设置，那么将会使用 `.proto` 文件的文件名进行驼峰转换之后作为类名，如 `foo_bar.proto` 会转换为 `FooBar.java`。如果不是生成 Java 代码，该选项不会生效。

    ```protobuf
    option java_outer_classname = "Ponycopter";
    ```

- `optimize_for`：文件级别的 `option`，可选值包括 `SPEED`，`CODE_SIZE` 和 `LITE_RUNTIME`。

    - `SPEED`：默认使用该值，编译器会生成 Message 类型的序列化，解析和其他常见操作的代码，改代码已经过高度的优化。
    - `CODE_SIZE`：编译器会生成最少的类，并将基于反射实现序列化，解析和其他常见操作，因此生成的代码会比 `SPEED` 更少，但执行起来会更慢。类中提供的 API 和 `SPEED` 模式下的一致，该模式适用于存在大量的 `.proto` 文件且不需要所有文件都快速运行的情况。
    - `LITE_RUNTIME`：编译器仅生成依赖于 `libprotobuf-lite` 的运行时库，精简版的库远小于完整的库，但是省略了一部分的功能。该选项适用于平台首先的运行环境中，如移动端。

    ```protobuf
    option optimize_for = CODE_SIZE;
    ```

- `cc_enable_arenas`：文件级别 `option`，为 C++ 生成的代码启用 [arena allocation](https://developers.google.com/protocol-buffers/docs/reference/arenas)

- `objc_class_prefix`：文件级别 `option`，设置生成 Objective-C 类的前缀，根据 Apple 的规范，我们应该指定 3 到 5 个大写的字符作为前缀，2 个字符的前缀是 Apple  的保留前缀。

- `deprecated`：字段级别 `option`，如果设置为 `true` 说明该字段已经过期，在新的代码中不应该使用该字段。在大多数的语言中，该选项并不生效。在 Java 中，该字段会被 `@Deprecated` 注解。

    ```protobuf
    int32 old_field = 6 [deprecated=true];
    ```



​    

