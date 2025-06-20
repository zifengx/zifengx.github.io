---
title: Windows 服务器 IIS 日志自动清理方案
date: 2025-06-19 21:50:00 -0400
categories: [服务器, IIS]
tags: [server, windows iis]     # TAG names should always be lowercase
---

## 问题描述

Windows 服务器 IIS 默认每天都会生成日志文件，如果网站的流量比较大，每天的日志文件可能会达到上百兆，这些日志文件如果不定时清理，日积月累便会严重地占用服务器磁盘空间。

## 解决办法

### 1. 编写批处理脚本自动清理日志

我们可以创建一个批处理文件（如命名为 `deleteIISlogs.bat`），自动删除超过设定保留天数的日志文件。内容如下：

```bat
:: 清理IIS日志文件
@echo off
title 清理IIS日志文件

:: IIS日志文件目录（根据实际情况修改）
set log_dir="C:\inetpub\logs\LogFiles"

:: 保留日志天数（此处设置为保留最近 30 天的日志，可根据实际需求修改，例如保留 60 天的日志将其改为 set bak_dat=60）
set bak_dat=30

:: 删除超过指定天数的日志文件
forfiles /p %log_dir% /S /M *.log /D -%bak_dat% /C "cmd /c echo 正在删除@relpath 文件… & echo. & del @file"
```
{: file='deleteIISlogs.bat'}

### 2. 双击运行 `deleteIISlogs.bat` ，即可自动删除 30 天前的日志文件，仅保留最近 30 天的记录

![delete iis logs](/assets/images/20250619/deleteIISlog.webp)

### 3. 配置服务器计划任务实现自动执行

为了避免手动执行，你可以将该脚本加入系统计划任务中，定期自动执行：

- 打开「任务计划程序」（Task Scheduler）；
- 点击「创建基本任务」；
- 设置任务名称，例如：自动清理IIS日志；
- 选择触发器（建议每日执行一次）；
- 操作类型选择「启动程序」，浏览选择 `deleteIISlogs.bat` 文件；
- 完成设置后保存即可。

> 建议在凌晨或服务器负载较低的时间段执行，避免对正常业务造成影响。
{: .prompt-tip }
