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
