---
title: 如何修复主机服务器出现的MySQL server has gone away
date: 2025-06-17 11:30:00 -0400
description: 最近，一个旧的PHP项目总是出现 MySQL server has gone away 的报错，经过调研，发现这是一个非常常见的 MySQL 报错，尤其是在共享主机或类似 GoDaddy Web Hosting 这种托管环境中。这个错误的本质原因是：查询时间太长，导致MySQL连接超时。
categories: [服务器, 数据库]
tags: [数据库，mysql]     # TAG names should always be lowercase
---

最近，一个旧的PHP项目总是出现 MySQL server has gone away 的报错，经过调研，发现这是一个非常常见的 MySQL 报错，尤其是在共享主机或类似 GoDaddy Web Hosting 这种托管环境中。  

这个错误的本质原因是：查询时间太长，导致MySQL连接超时。

## 解决方法

在网上也许你会看到其它教程推荐修改MySQL配置，  
例如：修改my.cnf 或 my.ini中的`wait_timeout`（单位 秒）

```ini
wait_timeout = 600
```

但是，修改`wait_timeout`（空闲连接超时时间）治标不治本，也不适合主机服务器（一般`wait_timeout`为60秒，也没有权限修改）

所以，解决MySQL server has gone away的问题，需要从这两点出发：

1. 优化SQL语句，减少查询时间 （后续补充SQL语句优化教程）
2. 在出现连接错误时，尝试进行一次重连  
   代码示例：（语言PHP，用mysqli面向过程写法，可自行修改为面向对象/PDO）

```php
<?php 
  // ====== 数据库连接配置 ======
  $db_host = 'localhost';
  $db_user = 'your_user';
  $db_pass = 'your_pass';
  $db_name = 'your_db';

  // ====== 封装连接函数 ======
  function get_db_connection() {
    static $db = null;

    if ($db && mysqli_ping($db)) {
      return $db; // 当前连接可用
    }

    // 若连接为空或失效，重新连接
    $db = @mysqli_connect($GLOBALS['db_host'], $GLOBALS['db_user'], $GLOBALS['db_pass']);
    if (!$db) {
      die('Database connection failed: ' . mysqli_connect_error());
    }

    if (!@mysqli_select_db($db, $GLOBALS['db_name'])) {
      die('Select DB failed: ' . mysqli_error($db));
    }

    return $db;
  }

  // ====== 执行查询函数 ======
  function db_query($sql) {
    $db = get_db_connection();
    $result = @mysqli_query($db, $sql);

    // 若查询失败，尝试重连一次 (重点)
    if (!$result && mysqli_errno($db) == 2006) { // 2006 = MySQL server has gone away
      $db = get_db_connection(); // 强制重连
      $result = @mysqli_query($db, $sql);
    }

    if (!$result) {
      error_log("SQL Failed: " . mysqli_error($db));
    }

    return $result;
  }
?>
```

`mysqli_ping`的开销很低，所以不必担心重复使用。  
如果一段程序内多次执行查询函数，觉得`mysqli_ping`有影响的话，可以考虑将`$db = get_db_connection()`从`db_query`内提出来，只在失败时强制重连时使用。
