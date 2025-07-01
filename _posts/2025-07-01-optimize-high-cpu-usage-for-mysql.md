---
title: 优化 MySQL 高 CPU 占用的排查与实用建议
description: 高 CPU 占用通常由慢 SQL、连接管理不善、临时表溢出、表碎片、参数配置不合理等多因素叠加。建议定期巡检、优化 SQL、合理配置参数，并结合慢查询日志持续跟踪。
date: 2025-07-01 01:00:00 -0400
categories: [服务器, 数据库]
tags: [mySQL, 性能优化, cpu]
---

服务器 MySQL 占用 CPU 过高时，应从以下几个方面系统排查：

## 1. 查看进程列表，定位高负载 SQL

使用：

```sql
SHOW PROCESSLIST;
```

重点关注 `Command` 为 `Query` 且 `Time` 较长的进程，分析 `Info` 字段中的 SQL 语句，优先优化慢 SQL（如加索引、重写语句等）。

| 字段    | 描述                       |
| ------- | -------------------------- |
| Id      | 进程标识，KILL 时使用      |
| User    | 当前用户                   |
| Host    | 来源 IP:端口               |
| db      | 连接的数据库               |
| Command | 当前命令（Sleep/Query 等） |
| Time    | 状态持续时间（秒）         |
| State   | SQL 状态                   |
| Info    | SQL 语句内容               |

## 2. 关注休眠（Sleep）线程

大量 Sleep 连接会消耗资源，甚至导致崩溃。建议：

- 检查连接池配置，避免长时间空闲连接未释放。
- 适当调小 `wait_timeout`，防止无效连接堆积。

## 3. 检查临时表与磁盘 IO

如进程 `State` 出现 `Copying to tmp table on disk`，说明临时表过大，已写入磁盘，影响性能。

- 查看/调整临时表大小：

```sql
SHOW VARIABLES LIKE 'tmp_table_size';
SET GLOBAL tmp_table_size=33554432; -- 32MB
```

## 4. 检查最大连接数

```sql
SHOW VARIABLES LIKE 'max_connections';
SET GLOBAL max_connections = 1024;
```

并发连接数过高会导致 `Too many connections` 错误。合理设置连接池和最大连接数。

## 5. 合理设置连接超时

`wait_timeout` 过大易堆积 SLEEP 进程，过小可能导致连接断开。

```sql
SHOW VARIABLES LIKE 'wait_timeout';
SET GLOBAL wait_timeout = 36000; -- 10 小时
```

## 6. 启用并分析慢查询日志

慢 SQL 是高 CPU 的常见元凶。建议：

```sql
SET GLOBAL slow_query_log = 1;
SHOW VARIABLES LIKE 'slow_query_log_file';
SET GLOBAL long_query_time=4; -- 4 秒
SHOW GLOBAL STATUS LIKE 'Slow_queries';
```

分析慢查询日志，重点优化 GROUP BY、ORDER BY、JOIN 等语句。

## 7. 定期维护与优化表/索引

- **分析表**：

```sql
ANALYZE TABLE db.tblname;
```

- **检查表**：

```sql
CHECK TABLE db.tblname;
```

- **优化表**（清理碎片）：

```sql
OPTIMIZE TABLE db.tblname;
```

## 8. 检查锁与参数配置

- 检查是否存在锁等待，合理设计事务与索引。
- 调整服务器参数以提升整体性能，常见参数包括：
  - `innodb_buffer_pool_size`：InnoDB 缓冲池大小，影响数据和索引缓存能力。
  - `key_buffer_size`：MyISAM 索引缓冲区大小。
  - `table_open_cache`：可同时打开的表数量。
  - `innodb_log_file_size`：InnoDB 日志文件大小，影响恢复和写入性能。
  - `max_heap_table_size`：内存临时表的最大大小。

建议根据实际业务负载和服务器内存情况合理设置上述参数。

## 总结

高 CPU 占用通常由慢 SQL、连接管理不善、临时表溢出、表碎片、参数配置不合理等多因素叠加。建议定期巡检、优化 SQL、合理配置参数，并结合慢查询日志持续跟踪。
