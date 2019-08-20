---
title: 【日常笔记系列】在PostgreSQL中使用Dapper时不能使用in?
categories: 日常笔记系列
comments: true
date: 2019/08/20 22:09:00
updated: 2019/08/20 22:09:00
tags:
    - 日常笔记系列
    - PostgreSQL
    - Dapper
---

## 错误：syntax error at or near "$1"
使用`Dapper`操作`PostgreSQL`时，发生此错误。
原因：PostgreSQL `IN`关键字不支持把数组作为参数，只有list才可以。你可以使用any关键字进行此项操作，如下：
```csharp
string sql = "select * from table where id= any(@ids)";
var result = DbConnection.Query<T>(sql, new { ids = ids });
```
