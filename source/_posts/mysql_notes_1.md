---
title: mysql笔记 (一)
date: 2018-02-01 22:59:05
tags: mysql
---

- [层级结构](#%E5%B1%82%E7%BA%A7%E7%BB%93%E6%9E%84)
- [存储引擎](#%E5%AD%98%E5%82%A8%E5%BC%95%E6%93%8E)
    - [MyISAM](#myisam)
        - [索引支持](#%E7%B4%A2%E5%BC%95%E6%94%AF%E6%8C%81)
    - [InnoDB](#innodb)
        - [主要特性](#%E4%B8%BB%E8%A6%81%E7%89%B9%E6%80%A7)
- [权限级别](#%E6%9D%83%E9%99%90%E7%BA%A7%E5%88%AB)
    - [Global Level](#global-level)
    - [Database Level](#database-level)
    - [Tabel Level](#tabel-level)
    - [Column Level](#column-level)
    - [Routine Level](#routine-level)
    - [GRANT](#grant)

<!-- more -->

# 层级结构

__Sql Layer__ 负责查询相关, __Storage Engine Layer__ 底层数据存储相关

将存储与计算分离

![mysql_layer.png](/images/mysql_layer.png)

# 存储引擎

## MyISAM

### 索引支持

1. B-Tree
    - 按照 _B-Tree_ 结构存储数据
2. R-Tree
    - 设计用于为存储空间和多维数据的字段做索引 (暂时不是很明白这段话的意思)
3. Full-Text
    - 全文检索, 提高 _LIKE_ 查询的效率

## InnoDB

### 主要特性

1. 支持事务安全
2. 数据多版本读取
3. 锁定机制改进
    - 改变了MyISAM的锁机制实现了行锁
4. 实现外建

# 权限级别

## Global Level

全局级别, __拥有最高优先级__

`*.*` 表示全局作用域

示例:

```sql
GRANT SELECT,UPDATE,DELETE,INSERT ON *.* TO 'def'@'localhost';
```

## Database Level

`database.*` 表示数据库级别作用域

示例:

```sql
GRANT ALTER ON test.* TO 'def'@'localhost';
```

也可以先选定数据库, 然后通过 `*` 来限定作用域为当前数据库, 如:

```sql
USE test;
GRANT DROP ON * TO 'def'@'localhost';
```

多个用户可以用逗号隔开:

```sql
GRANT CREATE ON perf.* TO 'abc'@'localhost','def'@'localhost';
```

## Tabel Level

`database.table` 表级别作用域

示例:

其中 _%_ 表示通配符

```sql
GRANT INDEX ON test.t1 TO 'abc'@'%.jianzhaoyang.com';
```

## Column Level

列级别, 语法与 _Table Level_ 相似, 只是需要在括号中指定列

示例:

```sql
GRANT SELECT(id,value) ON test.t2 TO 'abc'@'%.jianzhaoyang.com';
```

## Routine Level

主要针对 _procedure_ 和 _function_

## GRANT

特殊权限, 拥有该权限的用户可以将 __自身__ 拥有的权限赋予其他用户

示例:

```sql
GRANT ALL ON test.t5 TO 'abc';
```
