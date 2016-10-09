---
layout: post
title: mysql字符集相关
category: 技术
tags: mysql
keywords: mysql
description:
---

字符集(Character set)和排序方式(Collation)。

对于字符集的支持细化到四个层次：

* 服务器(server)
* 数据库(database)
* 数据表(table)
* 连接(connection)。

#### MySQL默认字符集

MySQL对于字符集的指定可以细化到一个数据库，一张表，一列，应该用什么字符集。

但是，传统的程序在创建数据库和数据表时并没有使用那么复杂的配置，它们用的是默认的配置，那么，默认的配置从何而来呢?  

1. 编译MySQL 时，指定了一个默认的字符集，这个字符集是 latin1；

2. 安装MySQL 时，可以在配置文件 (my.ini) 中指定一个默认的的字符集，如果没指定，这个值继承自编译时指定的；

3. 启动mysqld 时，可以在命令行参数中指定一个默认的的字符集，如果没指定，这个值继承自配置文件中的配置,此时 character_set_server 被设定为这个默认的字符集；

4. 当创建一个新的数据库时，除非明确指定，这个数据库的字符集被缺省设定为character_set_server；

5. 当选定了一个数据库时，character_set_database 被设定为这个数据库默认的字符集；

6. 在这个数据库里创建一张表时，表默认的字符集被设定为 character_set_database，也就是这个数据库默认的字符集；

7. 当在表内设置一栏时，除非明确指定，否则此栏缺省的字符集就是表默认的字符集；
