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
# 将数据编码成 base64，下面的 echo 会在 "test data" 后面加上换行符，如果不想带上换行符可以加上 `-n` 选项：echo -n "test data"|base64
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

## DER 编码的数据解析

openssl 命令的 asn1parse 选项提供了用于解析 der 格式数据的功能，关于 DER 格式的更多信息参见[ASN.1 语法・一](<http://www.mingfer.cn/2019/04/01/X690-ASN1%E8%AF%AD%E6%B3%95/>)，命令用法如下：

```shell
$ openssl asn1parse -h
unknown option -h
asn1parse [options] <infile
where options are
 -inform arg   input format - one of DER PEM
 -in arg       input file
 -out arg      output file (output format is always DER
 -noout arg    don't produce any output
 -offset arg   offset into file
 -length arg   length of section in file
 -i            indent entries
 -dump         dump unknown data in hex form
 -dlimit arg   dump the first arg bytes of unknown data in hex form
 -oid file     file of extra oid definitions
 -strparse offset
               a series of these can be used to 'dig' into multiple
               ASN1 blob wrappings
 -genstr str   string to generate ASN1 structure from
 -genconf file file to generate ASN1 structure from
```

`openssl asn1parse` 支持分析 DER 和 PEM 格式的任意文件，需要通过 `-inform` 选项进行文件内容格式的指定，默认的文件格式是 PEM。一般来说，我们拿到的数据更多的是 Base64 的数据，我们可以通过上面的 `base64` 命令将 Base64 字符串还原成二进制字节流，在进行 asn1 序列的拆分。如：

```shell
# 注意不同 linux 版本下，base64 命令的选项不同，下面所用的 base64 运行在 macOS 系统
$ echo "MIIDpDCCAowCCQDVaCKd1RCCajANBgkqhkiG9w0BAQsFADCBkzELMAkGA1UEBhMCQ04xCzAJBgNVBAgMAlNIMQswCQYDVQQHDAJTSDEOMAwGA1UECgwFdmlob28xITAfBgNVBAsMGEludGVybmV0IFdpZGdpdHMgUHR5IEx0ZDEYMBYGA1UEAwwPeW91ci5kb21haW4uY29tMR0wGwYJKoZIhvcNAQkBFg55b3VyQGVtYWlsLmNvbTAeFw0xOTA0MjAxMDA2MzlaFw0yOTA0MTcxMDA2MzlaMIGTMQswCQYDVQQGEwJDTjELMAkGA1UECAwCU0gxCzAJBgNVBAcMAlNIMQ4wDAYDVQQKDAV2aWhvbzEhMB8GA1UECwwYSW50ZXJuZXQgV2lkZ2l0cyBQdHkgTHRkMRgwFgYDVQQDDA9sZWFmLmRvbWFpbi5jb20xHTAbBgkqhkiG9w0BCQEWDnlvdXJAZW1haWwuY29tMIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAurytK07Sg/GQFuG1h2360caxuIdCEsbKm8Q0otx1k84UyaodajfDBNxJohUeZ0P58Xring8sfDKTM3tF61cMiv2N+01aYP09amDERki4ODqc62YHrZGOnAqGDqVjstJZ111yV6uUYSSbvZGPDFq0RLMq0lL/SNRBhdx9xNzFcYdQfoKwTiMsKMM9CB5n4FCBYYhO+793T1nmfvnTcZAxtng5SF71W842RsDujiDC9FUw/qEZ7Nx9w7RtMYmJ+QY+ioVW0jEW0HjeuHwJCjehMttaHLPQg0cVm15v5WH0qwdUJmxnrOv+5/9HceVD5q3FV+wP4bx4+/13rXKK3/6e8wIDAQABMA0GCSqGSIb3DQEBCwUAA4IBAQCMNSMwTf54OHVaE4A5k9ILFKS7imd0VP+aQ1fiXzO8hDoY4o+ygd43GGw51+ASkKX5zmPCqOW6Xeqc0Ie8wEwbYUwSG/ElN/QAnXazu/snrV0VYFjyI/pIG6DroUTgNU2xrKgUfNuIvGiRQJZzqbRY5oPUiMHuRyMobEf9VSRkIX5cXGvk3CWmUVWZEjgMN8ec2LksPI/sOzwHJYpPV2ZMSaagK0M/934cbPIOZztmKTWIBu1VqHESpMXvImQ9ijiY0OhRQGolkYPzZ0VU2ABrbZl90WfQUHHLkoqPG5V4pEq91KiEtRCA6FwxAe7m0rbzOxO1jmLLSwB0NAZt332S" | base64 -D -i -|openssl asn1parse -inform der
```

运行上面的命令我们可以得到类似下面的内容：

```shell
    0:d=0  hl=4 l= 932 cons: SEQUENCE          
    4:d=1  hl=4 l= 652 cons: SEQUENCE          
    8:d=2  hl=2 l=   9 prim: INTEGER           :D568229DD510826A
  . . . . . . . . . .
  660:d=1  hl=2 l=  13 cons: SEQUENCE          
  662:d=2  hl=2 l=   9 prim: OBJECT            :sha256WithRSAEncryption
  673:d=2  hl=2 l=   0 prim: NULL              
  675:d=1  hl=4 l= 257 prim: BIT STRING 
```

其中：

- 第一列的 `0, 4, 8` 代表节点在字节流里面的位置

- `:` 后面的 `d` 表示节点的深度

- `hl` 表示节点头字节的长度，所谓的头字节包含 Tag 域和长度域

- `l` 表示数据的长度

- `cons` 和 `prim` 表示数据是复合结构的数据还是不可拆分的原始数据

- `SEQUENCE` 和 `INTEGER` 等表示 [ASN.1]((<http://www.mingfer.cn/2019/04/01/X690-ASN1%E8%AF%AD%E6%B3%95/>)) 中定义的数据类型

- 最右边的内容为实际的数据内容，如果想要看到 `BIT STRING` 的内容，可以通过 `-dump` 选项打印出来。效果如下：

    ```shell
      675:d=1  hl=4 l= 257 prim: BIT STRING        
          0000 - 00 8c 35 23 30 4d fe 78-38 75 5a 13 80 39 93 d2   ..5#0M.x8uZ..9..
          0010 - 0b 14 a4 bb 8a 67 74 54-ff 9a 43 57 e2 5f 33 bc   .....gtT..CW._3.
          0020 - 84 3a 18 e2 8f b2 81 de-37 18 6c 39 d7 e0 12 90   .:......7.l9....
          0030 - a5 f9 ce 63 c2 a8 e5 ba-5d ea 9c d0 87 bc c0 4c   ...c....]......L
          0040 - 1b 61 4c 12 1b f1 25 37-f4 00 9d 76 b3 bb fb 27   .aL...%7...v...'
          0050 - ad 5d 15 60 58 f2 23 fa-48 1b a0 eb a1 44 e0 35   .].`X.#.H....D.5
          0060 - 4d b1 ac a8 14 7c db 88-bc 68 91 40 96 73 a9 b4   M....|...h.@.s..
          0070 - 58 e6 83 d4 88 c1 ee 47-23 28 6c 47 fd 55 24 64   X......G#(lG.U$d
          0080 - 21 7e 5c 5c 6b e4 dc 25-a6 51 55 99 12 38 0c 37   !~\\k..%.QU..8.7
          0090 - c7 9c d8 b9 2c 3c 8f ec-3b 3c 07 25 8a 4f 57 66   ....,<..;<.%.OWf
          00a0 - 4c 49 a6 a0 2b 43 3f f7-7e 1c 6c f2 0e 67 3b 66   LI..+C?.~.l..g;f
          00b0 - 29 35 88 06 ed 55 a8 71-12 a4 c5 ef 22 64 3d 8a   )5...U.q...."d=.
          00c0 - 38 98 d0 e8 51 40 6a 25-91 83 f3 67 45 54 d8 00   8...Q@j%...gET..
          00d0 - 6b 6d 99 7d d1 67 d0 50-71 cb 92 8a 8f 1b 95 78   km.}.g.Pq......x
          00e0 - a4 4a bd d4 a8 84 b5 10-80 e8 5c 31 01 ee e6 d2   .J........\1....
          00f0 - b6 f3 3b 13 b5 8e 62 cb-4b 00 74 34 06 6d df 7d   ..;...b.K.t4.m.}
          0100 - 92  
    ```

    