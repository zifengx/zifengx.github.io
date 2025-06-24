---
title: 如何使用Batch在Windows Server获取当前系统标准化日期时间（适配多语言/多地区）
date: 2025-06-21 13:50:00 -0400
description: 在 Windows 批处理脚本中，获取当前系统日期和时间看似简单，实际上却暗藏“陷阱”。不同语言、地区的操作系统，%date% 和 date 命令的输出格式都可能不同，导致直接字符串截取的方法不具备通用性。
categories: [服务器, 运维]
tags: [windows server, batch] # TAG names should always be lowercase
---

在 Windows 批处理脚本中，获取当前系统日期和时间看似简单，实际上却暗藏“陷阱”。不同语言、地区的操作系统，`%date%` 和 `date` 命令的输出格式都可能不同，导致直接字符串截取的方法不具备通用性。

## 常见的日期格式差异

- `星期二 2008-07-29`
- `2008-07-29 星期二`
- `07/29/2008 Tue`
- `Tue 07/29/2008`

## 为什么不能直接用 `%date%` 或 `date` 命令？

Windows 的日期格式由注册表 `HKEY_CURRENT_USER\Control Panel\International` 下的 `sShortDate` 决定。不同用户、不同地区、不同语言环境下，`%date%` 的内容都可能不同，直接用字符串截取很容易出错。

## 推荐做法：使用 WMIC 获取标准化日期时间

WMIC（Windows Management Instrumentation Command-line）可以直接获取标准化的本地日期时间，格式为 `yyyymmddHHMMSS.ffffff±UUU`，不受系统语言和地区影响。

### 批处理脚本示例

```bat
:: 获取本地标准化日期时间
for /f "tokens=2 delims==" %%a in ('wmic OS Get localdatetime /value') do set "dt=%%a"

:: 分别提取年月日时分秒
set "YY=%dt:~2,2%"
set "YYYY=%dt:~0,4%"
set "MM=%dt:~4,2%"
set "DD=%dt:~6,2%"
set "HH=%dt:~8,2%"
set "Min=%dt:~10,2%"
set "Sec=%dt:~12,2%"

:: 拼接成自定义格式（如 20250621_101530）
set "curFullstamp=%YYYY%%MM%%DD%_%HH%%Min%%Sec%"

echo 当前标准化时间戳：%curFullstamp%
```

### 输出示例

```plaintext
当前标准化时间戳：20250621_101530
```

## 适用范围

- 适用于所有 Windows 语言和地区设置
- 不依赖注册表、系统变量的具体格式
- 适合日志、备份、自动化脚本等场景

---

如需进一步定制或有其它脚本需求，欢迎留言交流！
