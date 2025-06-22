---
title: 如何在 Visual Studio 2022 上安装 .NET Framework 4.0&4.5
date: 2025-06-18 22:30:00 -0400
description: 为了使用最新的AI技术，把Visual Studio更新到了2022，然后就打不开.NET Framework4.0的旧项目了。因为VS2022不再原生支持.NET Framework 4.0&4.5, 支持的最低.NET Framework版本是4.6.2，所以需要通过一些方法来支持旧版本的项目。
categories: [如何解决, VS问题]
tags: [ide, visual studio, .net framework 4]     # TAG names should always be lowercase
---

## 问题描述

为了使用最新的AI技术，把Visual Studio更新到了2022，然后就打不开.NET Framework4.0的旧项目了。

![VS2022 does not support .NETFramework 4.0](/assets/images/20250618/vs2022-not-support-net4.0.png)

因为VS2022不再原生支持.NET Framework 4.0&4.5, 支持的最低.NET Framework版本是4.6.2，所以需要通过一些方法来支持旧版本的项目。

## 解决方法

### 方法1

安装支持.NET Framework 4.0&4.5的Visual Studio版本，如VS2017

### 方法2

从nuget下载目标框架，然后覆盖系统.NETFramework的v4.0/v4.5  

1. 官网下载[.NET Framework 4.0](https://www.nuget.org/packages/microsoft.netframework.referenceassemblies.net40) 或者 [.NET Framework 4.5](https://www.nuget.org/packages/microsoft.netframework.referenceassemblies.net45)。其它版本可到[nuget](https://www.nuget.org/packages)搜索  
2. 进入框架页面后，点击右侧的Download package链接下载目标框架
   ![.NETFramework 4.0 package](/assets/images/20250618/net4.0.png)
3. 把下载的框架文件后缀名从nupkg改成zip，然后解压
   ![change suffix from nupkg to zip](/assets/images/20250618/suffix-nupkg-to-zip.png)
4. 将解压目录下 `*\build\.NETFramework` 文件夹内的v4.0
   ![unzip net v4.0](/assets/images/20250618/unzip-net-v4.0.png)
   文件夹复制并覆盖到 `C:\Program Files (x86)\Reference Assemblies\Microsoft\Framework\.NETFramework`
   ![system net v4.0](/assets/images/20250618/system-net-v4.0.png)
5. 重启VS2022重新打开项目，发现已经支持.NET Framework 4.0了
   ![VS2022 support .NETFramework 4.0](/assets/images/20250618/vs2022-support-net4.0.png)
