---
title: MySQL 报错 Deadlock found when trying to get lock 的原因与解决办法
description: 在 MySQL 使用过程中有时候会遇到一个简单却需要精精校核的报错：ERROR 1213 (40001):Deadlock found when trying to get lock; try restarting transaction通常会出现在多个连接/事务并发执行定向 SQL 操作时，特别是 DELETE/UPDATE 形式的文件表操作。
date: 2025-06-26 13:00:00 -0400
categories: [服务器, 数据库]
tags: [mysql, 死锁, 错误排查, deaklock]
---

在 MySQL 使用过程中有时候会遇到一个简单却需要精精校核的报错：

```sql
ERROR 1213 (40001): Deadlock found when trying to get lock; try restarting transaction
```

通常会出现在多个连接/事务并发执行定向 SQL 操作时，特别是 DELETE/UPDATE 形式的文件表操作。

## 1. 基本原因

当多个事务同时试图锁定多个记录，但锁的顺序不同时，可能导致相互占用（比如 A 锁了记录 1，等待记录 2；B 锁了记录 2，等待记录 1），则系统无法解决这种相互等待，起用「逻辑分析」判定出现死锁，其中一个事务被终止。

## 2. 实际经验：DELETE 操作使用非主键导致死锁的错误样式

```sql
DELETE FROM onlineusers WHERE datetime <= NOW() - INTERVAL 900 SECOND;
```

上面这条 DELETE 语句使用 datetime 字段，如果此列未给主键或非唯一索引，在这种表形上就有可能导致多线程报 deadlock 错误。

## 3. 推荐写法：先查出 id ，再根据id DELETE

```sql
DELETE FROM onlineusers WHERE id IN (
    SELECT id FROM (
        SELECT id FROM onlineusers
        WHERE datetime <= NOW() - INTERVAL 900 SECOND
        ORDER BY id
    ) AS sub
);
```

**说明：**不能直接在 IN (子查询)中重复同表操作，必须内外子查询分隔 (MySQL 限制)

## 4. 预防策略

✅ **保证多个连接的操作顺序相同**

如：

- connection 1: locks id(1), locks id(2)
- connection 2: locks id(2), locks id(1)  ❌ 可能死锁
- connection 2: locks id(1), locks id(2) ✅ 同序无死锁

✅ **操作时使用主键/唯一索引**

避免表的多次全表扫描，更快排序和锁约。

✅ **加入客户端重试逻辑**

MySQL 官方文档建议：

> "If a deadlock occurs, retry the transaction."

建议在端程序加入自动重试逻辑，如：

```csharp
int retryCount = 0;
while (retryCount < 3)
{
    try
    {
        // 执行操作
        break;
    }
    catch (MySqlException ex) when (ex.Message.Contains("Deadlock found"))
    {
        retryCount++;
        Thread.Sleep(100); // 等等线程重新分配
    }
}
```

## 5. 总结

死锁很常见于多事务并发时间的执行顺序不一致，特别是启用 InnoDB 库时。

DELETE/UPDATE 操作应使用主键或唯一索引来减少问题。

建议尽可能接入 "顺序锁" + "回退重试"策略。
