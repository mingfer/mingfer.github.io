---
layout: cv
hidden: true
title: 简历
year: 1994/01 5年工作经验
email:
  url: mailto:mingfer.cn@gmail.com
  text: mingfer.cn@gmail.com
homepage:
  url: https://www.mingfer.cn
  text: Mingfer's Blog
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

### **我的技术栈** 

- 熟练掌握 Java 和面向对象设计开发，能够熟练使用 SpringBoot 框架，包括 WebMVC 和 WebFlux。
- 熟悉 TCP/IP，SSL/TLS 工作原理，熟悉 gRPC 协议，能够熟练的使用 Java 工具进行性能调优。
- 熟悉 git 的工作流（GitLab），能够熟练使用 MarkDown，有良好的文档编写风格和习惯。
- 能够熟练使用 Linux 命令和编写 Shell 脚本，对 C/C++, Objective-C，JavaScript 等编程语言有一定的了解。
- 对 Docker 有一定的了解，使用 Docker 搭建过 Jenkins，Nexus 等开发平台。

### **我的业务栈** 

- 熟悉常用的对称和非对称算法的使用，对证书体系和 PKI 体系有深入的了解
- 熟悉 JCE 框架，有在 BouncyCastle 库上扩展国密 SSL 的经验，并通过了国密检测中心的验收
- 能够熟练的通过阅读 RFC 等算法规范资料完成算法流程设计
- 有带领敏捷团队的经验，能够熟练的使用敏捷方法展开 Sprint 迭代

### **我的期望** 
- 期望工作：Java 应用开发；数据保护与信息安全；区块链
- 期望薪资：税前月薪 25K ~ 28K，如果有特别喜欢的工作内容可例外
- 期望城市：广州，深圳，成都，重庆

## 工作经历

### **广州江南科友股份有限公司** `2015.12 - now`

*Java 工程师*<br><br>

- **CaaS 密码服务平台**<br>CaaS 是一个为 MasterCard 业务系统提供包括加解密，数字证书签发吊销，卡密钥和卡数据制作的综合密码服务平台。该系统使采用 2 管理端 + 8 业务端的方式集群部署，采用 hazelcast 作为缓存框架并保证分布式下缓存数据的一致性。管理端采用 WebMVC 框架实现，业务端采用 WebFlux 框架实现，使用 gRPC 协议同硬件加密设备进行通讯，使用 OpenAPI 作为接口文档。主要负责该系统基本框架的搭建，工作流程的设计，性能优化和技术难点的突破。带领团队开展开发工作，实现业务价值，完成 Sprint 的计划，评审和回顾<br><br>
- **Java SDK**<br>负责科友各平台的 Java SDK 客户端的开发，该客户端使用 TCP 协议和平台完成通讯。独立实现了连接池，负载均衡，多种报文格式，长短链接，TLS 及国密 SSL 通讯等模块；对外提供了普通的 Java 调用接口，JCE 标准接口和基于注解的加解密注解接口。除此之外，还负责该 SDK 的性能调优，客户现场问题排查等工作。<br><br>
- **软算法，安全控件与动态令牌**<br>负责 JavaScript，Java（包括 Android）和 Objective-C 版本软算法的开发。安全控件是基于软算法的使用在移动端和 Web 浏览器上的一个控件，主要作用是保证用户在输入登录密码和交易密码时，提供输入安全的保障和数据安全传输支持。动态令牌是一个离线的用于产生一次性口令的安全应用，主要作用是通过一次性口令的方式替换用户密码，免去用户设置密码和记忆密码的麻烦。<br><br>


## 致谢

非常感谢您百忙之中看到我的简历，如果您对我感兴趣，可以邮件联系我。非常期望加入到您的团队，和您一起创造非凡的价值。


