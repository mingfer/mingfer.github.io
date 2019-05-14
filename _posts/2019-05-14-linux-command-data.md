---
layout: post
title: "Linux 数据转换命令"
subtitle: "如何实现 Base64，DER，JSON 等数据格式的解析"
date: 2019-05-14 21:03:00
author: "mingfer"
header-img: "img/post/bg-1.jpg"
catalog: true
tags: 
    - "Linux"
---

> 主要说明 Linux 下一些用于数据转换的常用命令

## Base64 数据转换

一般来说 Linux 提供了 `base64` 命令来编解码数据，在不同的 Linux 版本中该指令提供的参数和功能可能有所区别。

在 macOS 下，`base64` 命令包括下面的选项：

```shell
  -D, --decode   解码输入的数据
  -b, --break    多少个字符进行换行
  -i, --input    input file (default: "-" for stdin)
  -o, --output   output file (default: "-" for stdout)
```

下面是一些使用的示例：

```shell
# 将数据编码成 base64
$ echo "test data"| base64

# 将数据从 Base64 解码，archlinux 下是 `-d`
$ echo "dGVzdCBkYXRhCg=="|base64 -D

# 将文件内容编码成 Base64，常用于编码存放二进制数据的文件，不加 `-o` 直接打印到终端，加 `-b 64` 输出数据按 64 个字符换行
$ base64 -i test.bin -o test.b64

# 解码 Base64 文件内容，`-o -` 可以直接打印到终端
$ base64 -i test.b64 -D -o test.bin
```

## JSON 数据解析

[jq](https://stedolan.github.io/jq) 是一款 Linux 命令行下用于处理 JSON 数据工具，可以对json数据进行分片、过滤、映射和转换，[这里](<https://stedolan.github.io/jq/tutorial/>) 是它的官方教程。

### 基本用法

jq 最简单的表达式是 `.` ，接收输入并将其转化成优美的输出。

```shell
$ echo '[{"Config":"b81355b10fa34f9bc195fe70c579b373337c4dd0b3a9bb24369376bc7ff64eb1.json","RepoTags":null,"Layers":["4bfac6fe3333e456db496465d112632e00986bf42400faee457a87c8db947b5f/layer.tar","24c116370bf6caa7fdd2e335254bddfd06611cc9410e9c6290680a500da1a768/layer.tar","8a2fb2e56ebf02e6f605fe874409a9c5ca4ae99cb713a8ff19ad1c9a0f4fcab9/layer.tar"]}]' | jq '.'
```

输出结果为：

```json
[
  {
    "Config": "b81355b10fa34f9bc195fe70c579b373337c4dd0b3a9bb24369376bc7ff64eb1.json",
    "RepoTags": null,
    "Layers": [
      "4bfac6fe3333e456db496465d112632e00986bf42400faee457a87c8db947b5f/layer.tar",
      "24c116370bf6caa7fdd2e335254bddfd06611cc9410e9c6290680a500da1a768/layer.tar",
      "8a2fb2e56ebf02e6f605fe874409a9c5ca4ae99cb713a8ff19ad1c9a0f4fcab9/layer.tar"
    ]
  }
]
```

### RAW String 和 JSON Text 转换

jq 支持将一个 JSON 对象转换成一个字符串，在我们需要填写一个 JSON 字符串的时候可以借助这个功能快速的将 JSON 对象转换成 JSON 字符串：

```shell
echo '[{"Config":"b81355b10fa34f9bc195fe70c579b373337c4dd0b3a9bb24369376bc7ff64eb1.json","RepoTags":null,"Layers":["4bfac6fe3333e456db496465d112632e00986bf42400faee457a87c8db947b5f/layer.tar","24c116370bf6caa7fdd2e335254bddfd06611cc9410e9c6290680a500da1a768/layer.tar","8a2fb2e56ebf02e6f605fe874409a9c5ca4ae99cb713a8ff19ad1c9a0f4fcab9/layer.tar"]}]' | jq '.' -R
```

输出结果为：

```shell
"[{\"Config\":\"b81355b10fa34f9bc195fe70c579b373337c4dd0b3a9bb24369376bc7ff64eb1.json\",\"RepoTags\":null,\"Layers\":[\"4bfac6fe3333e456db496465d112632e00986bf42400faee457a87c8db947b5f/layer.tar\",\"24c116370bf6caa7fdd2e335254bddfd06611cc9410e9c6290680a500da1a768/layer.tar\",\"8a2fb2e56ebf02e6f605fe874409a9c5ca4ae99cb713a8ff19ad1c9a0f4fcab9/layer.tar\"]}]"
```

同样的 jq 支持读取一个字符串，并转换成 JSON 对象：

```shell
$ echo "[{\"Config\":\"b81355b10fa34f9bc195fe70c579b373337c4dd0b3a9bb24369376bc7ff64eb1.json\",\"RepoTags\":null,\"Layers\":[\"4bfac6fe3333e456db496465d112632e00986bf42400faee457a87c8db947b5f/layer.tar\",\"24c116370bf6caa7fdd2e335254bddfd06611cc9410e9c6290680a500da1a768/layer.tar\",\"8a2fb2e56ebf02e6f605fe874409a9c5ca4ae99cb713a8ff19ad1c9a0f4fcab9/layer.tar\"]}]" |jq '.' -r
```

### 提取具体的对象

在 jq 中：

- `.` 代表 JSON 对称本身
- `[]` 表示一个数组，`[index]` 表示具体的数组元素
- `|` 将左边过滤的内容作为一个新的 JSON 对象管道到右边，这个在访问对象的子对象的时候非常有用。

```json
[
  {
    "Config": "b81355b10fa34f9bc195fe70c579b373337c4dd0b3a9bb24369376bc7ff64eb1.json",
    "RepoTags": null,
    "Layers": [
      "4bfac6fe3333e456db496465d112632e00986bf42400faee457a87c8db947b5f/layer.tar",
      "24c116370bf6caa7fdd2e335254bddfd06611cc9410e9c6290680a500da1a768/layer.tar",
      "8a2fb2e56ebf02e6f605fe874409a9c5ca4ae99cb713a8ff19ad1c9a0f4fcab9/layer.tar"
    ]
  }
]
```
如果我们要获取上面 JSON 数据中 Layers 字段的第二个元素，用 jq 可以这样表示：
```shell
$ echo "[{\"Config\":\"b81355b10fa34f9bc195fe70c579b373337c4dd0b3a9bb24369376bc7ff64eb1.json\",\"RepoTags\":null,\"Layers\":[\"4bfac6fe3333e456db496465d112632e00986bf42400faee457a87c8db947b5f/layer.tar\",\"24c116370bf6caa7fdd2e335254bddfd06611cc9410e9c6290680a500da1a768/layer.tar\",\"8a2fb2e56ebf02e6f605fe874409a9c5ca4ae99cb713a8ff19ad1c9a0f4fcab9/layer.tar\"]}]" |jq '.[0] | .Layers[1]'
```

如果我们需要将 `Layers` 重命名为 `layers` :

```shell
echo "[{\"Config\":\"b81355b10fa34f9bc195fe70c579b373337c4dd0b3a9bb24369376bc7ff64eb1.json\",\"RepoTags\":null,\"Layers\":[\"4bfac6fe3333e456db496465d112632e00986bf42400faee457a87c8db947b5f/layer.tar\",\"24c116370bf6caa7fdd2e335254bddfd06611cc9410e9c6290680a500da1a768/layer.tar\",\"8a2fb2e56ebf02e6f605fe874409a9c5ca4ae99cb713a8ff19ad1c9a0f4fcab9/layer.tar\"]}]" |jq '[.[0] | {Config: .Config, RepoTags: .RepoTags, layers: .Layers}]'
```