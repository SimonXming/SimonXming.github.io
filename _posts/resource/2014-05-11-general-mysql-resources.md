---
layout: post
title: MySQL常用资源
category: 资源
tags: MySQL
keywords: MySQL
description:
---

## 常用命令

### 登录数据库

    mysql -h localhost -uroot -p

### 导出数据库

    mysqldump -uroot -p db > db.sql

### 导入数据库

    mysql -uroot -p db < db.sql
    // or
    mysql -uroot -p db -e "source /path/to/db.sql"

### 开启远程登录

    grant all privileges on ss.* to 'root'@'%' indentified by 'passoword' with grant option;
    // or
    update user set Host="%" and User="root"
    // 注意%是不包含localhost的
    flush privileges;

### 创建用户

    CREATE USER 'test'@'localhost' IDENTIFIED BY 'password';
    grant all privileges on *.* to test@'localhost' identified by 'test';

### 创建表

    CREATE SCHEMA testdb DEFAULT CHARACTER SET utf8 COLLATE utf8_unicode_ci;

### 修改表字符集
    ALTER DATABASE testdb CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;

### 修改mysql默认配置
    cat /etc/my.cnf
    cat /etc/mysql/my.cnf

### 查看character_set和collation相关设置
    ```shell
    mysql> show variables like 'character_set_%';
    +--------------------------+----------------------------+
    | Variable_name            | Value                      |
    +--------------------------+----------------------------+
    | character_set_client     | utf8                       |
    | character_set_connection | utf8                       |
    | character_set_database   | utf8mb4                    |
    | character_set_filesystem | binary                     |
    | character_set_results    | utf8                       |
    | character_set_server     | latin1                     |
    | character_set_system     | utf8                       |
    | character_sets_dir       | /usr/share/mysql/charsets/ |
    +--------------------------+----------------------------+
    8 rows in set (0.00 sec)

    mysql> show variables like 'collation_%';
    +----------------------+--------------------+
    | Variable_name        | Value              |
    +----------------------+--------------------+
    | collation_connection | utf8_general_ci    |
    | collation_database   | utf8mb4_unicode_ci |
    | collation_server     | latin1_swedish_ci  |
    +----------------------+--------------------+
    3 rows in set (0.01 sec)

    mysql> SHOW CHARACTER SET;          # 查看所有支持的字符集
    ```

[mysql字符集相关资料](https://dev.mysql.com/doc/refman/5.7/en/charset.html)


### 赋予数据库权限

    GRANT ALL ON testdb.* TO 'test'@'localhost';
