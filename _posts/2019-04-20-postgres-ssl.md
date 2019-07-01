---
layout: post
title: "使用 SSL/TLS 和 PostgreSQL 通讯"
subtitle: "Spring Boot 下如何和 PostgreSQL 建立 SSL/TLS 连接"
date: 2019-04-20 16:03:00
author: "mingfer"
header-img: "img/post/bg-2.jpg"
catalog: true
tags: 
    - "Spring Boot"
    - "数据库"
	- "Web 后端"
---

> 出于安全方面的考虑，有时候可能我们会希望应用和数据库之间的数据传输是通过 SSL/TLS 协议进行的。本文的主要内容在于介绍一个基于 Spring Boot 的应用如何和 [PostgreSQL](https://www.postgresql.org) 数据库进行 SSL/TLS 通讯。



在 PostgreSQL 数据库的 JDBC 官方文档中，对如何配置 SSL 进行了说明。但是根据官方文档的解释，PostgreSQL 的 JDBC 只是实现一个加密的 SSL 链路，但是并不会验证证书的有效性。这不一定能够满足某些情况下要求的安全强度，所以下文会介绍如何通过扩展 JDBC 实现和 PostgreSQL 建立双向验证的 SSL/TLS 链路。

参考文档：

- [Chapter 4. Using SSL](https://jdbc.postgresql.org/documentation/94/ssl.html)
- [18.9. Secure TCP/IP Connections with SSL](https://www.postgresql.org/docs/current/ssl-tcp.html)

## 环境准备

为了快速的进行 PostgreSQL 的使用，这里使用 Docker 启动 PostgreSQL，关于 PostgreSQL 在 docker 下的使用参见[这里](https://hub.docker.com/_/postgres/)。

启动一个 PostgreSQL 实例：

```shell
$ docker run --name postgres -d -v ~/tmp/config:/config -e POSTGRES_PASSWORD=123456 postgres
```

- `-d` 是指后台启动
- `-v ~/tmp/config:/config` 是将宿主机的 `~/tmp/config` 目录映射到容器的 `/config` 目录，方面之后配置证书
- `-e POSTGRES_PASSWORD=123456` 指定 PostgreSQL 的登录密码，默认的用户是 `postgres`，可以通过 ``POSTGRES_USER`` 变量更改用户名。

重新启动一个实例，登录到上一个实例中进行数据库操作：

```shell
# 启动一个临时的 posetgres 实例当做客户端
$ docker run --rm -ti postgres /bin/bash

# 登录到 posetgres 数据库实例中，密码是 123456
$ psql -U postgres -W

# 创建一个 test 数据库
postgres=# create database test;

# 切换到 test 数据库下面, 用户密码是 123456
postgres=# \c test
```

使用 [demo-posetgresql](https://github.com/mingfer/demo/tree/master/demo-postgresql)  中的 `cn.mingfer.demo.postgresql.UserRepositoryTest#insert` 方法插入一条数据到 user 数据表（这里使用 JPA 自动执行了数据表的创建，也可以自己手动创建数据表）。

## PostgreSQL 单向 SSL

PosetgreSQL 提供的 JDBC 中支持的 SSL 是纯 SSL 链路，也就是不对双方的身份证书进行校验，而只是建立一个加密信道。下面是配置 PostgreSQL SSL 链路的过程：

启动一个 PostgreSQL 容器：

```shell
$ docker run --name postgres -d -v ~/tmp/config:/config -e POSTGRES_PASSWORD=123456 postgres
```

**注意：**这里需要将配置目录 `/config` 和宿主机的一个目录进行映射，方便我们后面进行证书文件的复制和修改配置文件。PostgreSQL 提供的镜像里面没有包含 vi 等编辑工具，我们需要借助宿主机编辑之后复制到对应的目录。

签发出标识 PostgreSQL 的证书文件 `postgres.crt` 和私钥文件 `postgres.key`，具体的证书文件生成过程参见[生成数字证书](http://www.mingfer.cn/2019/04/20/cert-gen/)。将这两个文件放到宿主机的 `~/tmp/config` 目录下，将权限配置为 `600` (`chmod 600 postgres.*`) ，也就是当前用户可读写。

通过下面的命令进入到 PostgreSQL 容器中：

```shell
$ docker exec -ti postgres bash
```

将 `/var/lib/postgresql/data/` 目录下面的 `pg_hba.conf` 和 `postgresql.conf` 这两个配置文件复制到 `/config` 目录下面：

```shell
$ cd /var/lib/postgresql/data/
$ cp pg_hba.conf postgresql.conf /config
```

在宿主机的 `~/tmp/config` 目录下编辑 `postgresql.conf` 文件，将 `ssl` 配置为 `on`，`ssl_cert_file` 是 PostgreSQL 的身份证书，`ssl_key_file` 是 PostgreSQL 的私钥文件。`ssl_ca_file` 是 PostgreSQL 信任的证书，PostgreSQL 通过这些证书去认证客户端，这里我们只是配置单纯的 SSL 用不着验证客户端身份，所以略过此项：

```shell
# - SSL -

ssl = on
#ssl_ca_file = '/config/root.crt'
ssl_cert_file = '/config/postgres.crt'
#ssl_crl_file = ''
ssl_key_file = '/config/postgres.key'
#ssl_ciphers = 'HIGH:MEDIUM:+3DES:!aNULL' # allowed SSL ciphers
#ssl_prefer_server_ciphers = on
#ssl_ecdh_curve = 'prime256v1'
#ssl_dh_params_file = ''
#ssl_passphrase_command = ''
#ssl_passphrase_command_supports_reload = off
```

在 `pg_hba.conf` 文件中配置一条 `hostssl` 规则：

```shell
# IPv4 local connections:
hostssl    all             all             0.0.0.0/0            md5
```

使用配置好的文件覆盖 `/var/lib/postgresql/data/` 下面的同名文件，**请注意备份原文件**。

```shell
$ cd /var/lib/postgresql/data/
$ cp /config/postgresql.conf /config/pg_hba.conf ./
```

退出容器，重启容器：

```shell
$ exit
$ docker restart postgres
```

启动一个临时的容器对 PostgreSQL 进行访问：

```shell
$ docker run --rm -ti 30bf4f039abe /bin/bash
root@6460748d933b:/# psql -U postgres -h 192.168.10.102 
Password for user postgres: 
psql (11.2 (Debian 11.2-1.pgdg90+1))
SSL connection (protocol: TLSv1.2, cipher: ECDHE-RSA-AES256-GCM-SHA384, bits: 256, compression: off)
Type "help" for help.

postgres=# 
```

可以发现连接到数据库的连接是 `TLSv1.2` 的 SSL 连接。

需要注意，如果是从 PostgreSQL 数据库容器本地用户去访问，这里是不会使用 SSL 的：

```shell
$ docker exec -ti postgres bash
root@7554074b9f97:/# su postgres
postgres@7554074b9f97:/$ psql
psql (11.2 (Debian 11.2-1.pgdg90+1))
Type "help" for help.

postgres=# 
```

因为此时的连接是使用 `pg_hba.conf` 中的 local 规则建立连接的：

```shell
# "local" is for Unix domain socket connections only
local   all             all                                     trust
```



