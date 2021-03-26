---
layout: cv
hidden: false
title: 简历
year: 27岁 5年工作经验
phone: 187 1588 7094
email:
  url: mailto:mingfer.cn@gmail.com
  text: mingfer.cn@gmail.com
homepage:
  url: https://www.mingfer.cn
  text: mingfer.cn
---

# 张明丰

<!--
include contact information from the front matter
Supported arguments:
    - homepage: url, text
    - phone
    - email
-->

{% include cv-contact.html %}

## 个人信息

### **技术栈** 
- 编程语言：熟悉 Java，主要使用 JDK1.8；熟悉面向对象设计；对 Shell，JavaScript，C/C++，Objective-C 等编程语言也有一定的了解
- 开发框架：熟悉 SpringBoot，能够熟练使用 WebMVC + HazelCast + JPA 和 WebFlux + HazelCast + JPA 搭建和开发应用
- 通讯协议：熟悉 TCP，HTTP1.1，HTTP2.0 和 GRPC 协议；熟悉 SSL/TLS 和国密 SSL 的工作原理和实现；能够熟练使用 wireshark，tcpdump，openssl，curl 等工具排查通讯问题
- 测试与调优：熟悉 Junit4 和 Junit5 框架，熟悉 Postman 和 supertest 等 HTTP 接口测试工具；能够熟练使用 jvisualvm，jstack 等工具分析性能瓶颈和进行性能优化
- 熟悉 PKI 体系和常用的密码算法，熟悉数字证书的生成及使用，熟悉 KMIP 规范
- 熟悉安全编码相关的规范；有参与和建设敏捷团队 (Scrum) 的经验
- 文档与版本管理：熟悉 Markdown 语法和 OpenAPI 3 规范；能够熟练使用 git 和 GitLab，Gitea 等代码托管平台
- 能够熟练使用相关 Linux 命令，对 Docker 的使用有一定了解；能够使用 gitea + jenkins + nexus 搭建持续测试持续集成平台

### **求职意向**
<!-- - 期望薪资：25k - 30k 元/月 -->
- 工作地点：成都，重庆，广州，深圳
- 工作方向：数据安全，区块链，CA

## 工作经历

### **广州江南科友股份有限公司** `2015.12 - now`

*Java 工程师*<br><br>

#### **CaaS: Crypto as a Service 密码服务平台** `2018.5 - now`

CaaS 是为万事达卡（MasterCard）提供的一套综合密码服务平台。平台采用集群+双数据中心的部署方式，使用 hazelcast 缓存框架并集群缓存数据的一致性，所有实例使用统一的 PG 数据库。管理端使用 WebMVC + JPA，主要完成对密钥，接入应用和工单的管理。业务端使用 WebFlux + GRPC，提供 HTTP2 和 GRPC 两种协议的密码服务，使用 gRPC 协议同硬件加密设备进行通讯。
- **项目职责**
    - 参与系统的概要设计和详细设计，负责完成系统框架的搭建和技术难点攻关
    - 负责设计应用端的 GRPC 接口和 HSM 端的调用接口，并搭建 GRPC 通讯框架
    - 负责设计和实现密钥与应用管理，应用接入认证和应用密钥授权
    - 负责接口测试框架的搭建和各种工具脚本的编写（gradle 脚本，自签发证书生成脚本等）
    - 负责 GRPC 负载均衡和健康检查机制的实现，负责接口性能调优
    - 负责团队代码评审，任务进度跟踪和 PCI 过审，带领团队参加 Sprint 迭代
<br><br>

#### **Java SDK** `2016.6 - now`

科友各个产品基本提供基于 TCP 和 TLV 报文的服务，因此需要 Java SDK 提供给用户的 Java 应用快速集成使用。该 SDK 内置了独立的连接池，支持长短链接；支持轮询和加权等多种负载均衡方式；支持健康检查和异常服务器隔离；支持 SSL/TLS 和国密 SSL 密文链路；支持日志脱敏；支持配置文件配置，参数化配置和远程配置。
- **项目职责**
    + 负责完成客户提出的功能需求和 SDK 的版本维护
    + 负责性能调优，解决包括农信银，浦发银行，万事达等客户提出的性能需求
    + 负责线上生产问题的排查
<br><br>

#### **软算法 SDK** `2018.5 - now`

软算法 SDK 是基于 BouncyCastle 库封装的一套便于应用开发人员使用的软算法接口。SDK 提供了常用的 RSA，EC，SM2，SM4，AES，3DES，SM3，SHA1，SHA2 等算法的封装。同时 SDK 支持从各个 KMS 密钥源获取密钥，包括 SDK 自己生成；从密码服务平台（CSSP）获取；从隐私数据治理平台（PDGP）平台获取；从 KMIP 服务端获取。
- **项目职责**
    + 负责 Java 和 JavaScript 两个版本的接口封装和维护
    + 根据 KMIP 规范编写 KMIP 客户端获取 KMIP 服务端的密钥
    + 基于 HttpClient 封装基于 AKSK 认证的 CSSP 和 PDGP 客户端
<br><br>

#### **国密 SSL SDK** `2020.5 - now`

国密 SSL SDK 是基于国标 SSL VPN 规范在 BouncyCastle tls 库上实现的国密 SSL 套件。SDK 实现了 ECC-SM4-SM3 和 ECDHE-SM4-SM3 两个国密套件，同时支持启动客户端和服务端，支持单双向 SSL，支持多种证书和私钥格式的解析。
- **项目职责**
    + 根据规范实现密码套件
    + 支持公司的 Java 产品集成国密 SSL
    + 支持浦发银行，万事达等客户联调使用 SDK
<br><br>

#### **JCE Provider** `2018.8 - now`

JCE Provider 是基于密码服务平台或 HSM 提供的一套 JCE 接口，由于密钥和计算都发生在硬件环境，所以该 Provider 有着极高的安全性。同时因为 JCE 接口是标准的 Java 接口，这意味着应用可以方便的在不同的 Provider 中切换，而无需担心兼容性问题。
- **项目职责**
    + 负责实现密钥生成，加解密，签名验签，密钥库等一系列的 JCE 接口
    + 负责客户集成的技术支持和版本日常维护
    + 负责项目管理，完成需求分析，功能设计和编码
<br><br>

#### **secure-jdbc** `2020.12 - now`

secure-jdbc 是对 JDBC 的封装，提供了数据入库加密和数据出库解密的功能。secure-jdbc 通过对 DataSource 或 Driver 进行包装，在数据入库前和出库后进行 SQL 解析和敏感字段加密解密。
- **项目职责**
    + 设计接口支持软算法加解密，远程服务加解密等不同的加解密方式
    + 负责实现 SQL 的解析，敏感字段识别和字段值的提取
    + 负责实现支持 SpringBoot 自动配置功能 
<br><br>

#### **安全控件** `2016.9 - now`

安全控件是用于移动端和 H5 小程序的一套安全组件。安全控件提供了数据加密，安全键盘，密钥安全存储等功能。
- **项目职责**
    + 负责 iOS 和 JavaScript 版本的底层算法实现和封装
    + 负责接口设计，使用手册的编写和国密局过检材料的编写
    + 负责项目管理，完成需求分析，功能设计和功能评审

<br><br>

#### **配置管理系统** `2019.12 - now`

配置管理系统用于快速完成科友各个平台后台管理功能的配置。该系统分为配置端和执行端，配置端用于完成页面配置并产生配置文件；执行端用于执行配置文件并提供 WEB 服务。通过该系统，公司内部快速完成了 KMS，TKMS，RA，CMS 等系统的管理端配置。
- **项目职责**
    + 负责后端底层框架的搭建和技术难点的解决
    + 负责需求管理，功能设计和功能评审  
<br><br>

#### **动态令牌 APP** `2018.5 - now`

动态令牌 APP 是配合 OTPS 系统使用的一个一次性口令提供终端。
- **项目职责**
    + 负责口令生成算法的编写
    + 负责项目管理，完成需求分析，功能设计和功能评审 
<br><br>

#### **监控分析系统** `2018.5 - 2019.6`

监控分析系统主要用于采集公司各个产品的日志信息和运行状况并完成数据汇总，分析和展示。
- **项目职责**
    + 负责 Scrum Master 角色，组织站会，计划评审回顾会议
    + 负责需求管理，功能设计和功能评审  
    + 负责项目管理并通过 CMMI 认证


## 最高学历
### **重庆交通大学** `2012.9 - 2016.6`
本科 | 电子信息工程


## 致谢

感谢您在百忙之中看到这份简历，如果您对简历感兴趣，可以通过上面的邮件联系我。非常期待加入到您的团队，共创美好的未来。


