---
layout: post
title: Oracle 使用小记
category: 数据库
tags: oracle
keywords: oracle
description:
---

### 语法

```sql
DECLARE
  c_log LONG;
BEGIN
  c_log := 'This is console log.';
  UPDATE BUILD SET console_log = c_log WHERE ID = '18c52e86a043443';
END;
```

### 数据类型

### 编码问题

### SQL 语句与4000个字符

### 查看与删除锁

```sql
FROM gv$locked_object l, dba_objects o, gv$session s WHERE l.object_id　= o.object_id AND l.session_id = s.sid;

ALTER system kill session '23, 1647'; 
```
