---
title: 解决 WCF 大数据量传输
description: 开发中需要通过 WCF 进行大数据量传输，遇到如下报错：System.Net.Sockets.SocketException:远程主机强迫关闭了一个现有的连接
date: 2025-06-30 15:00:00 -0400
categories: [后端, 开发问题]
tags: [wcf, socketexception, 大数据传输, .net]
---

开发中需要通过 WCF 进行大数据量传输，遇到如下报错：System.Net.Sockets.SocketException: 远程主机强迫关闭了一个现有的连接

网上的解决方案大多雷同且不完整，实际操作中很难直接解决。本文记录本人踩坑与最终解决过程，供遇到类似问题的开发者参考。

## 1. 常见建议：增大传输相关配置

### 服务端配置

```xml
<bindings>
  <basicHttpBinding>
    <binding name="GEDemoServiceBinding" closeTimeout="00:05:00" openTimeout="00:05:00" sendTimeout="00:05:00"
      maxBufferPoolSize="2147483647" maxBufferSize="2147483647" maxReceivedMessageSize="2147483647">
      <readerQuotas maxStringContentLength="2147483647" maxArrayLength="2147483647" maxBytesPerRead="2147483647" maxNameTableCharCount="2147483647" maxDepth="32" />
    </binding>
  </basicHttpBinding>
</bindings>
```

### 客户端配置

```xml
<bindings>
  <basicHttpBinding>
    <binding name="BasicHttpBinding_IGEDemoService" closeTimeout="00:01:00" openTimeout="00:01:00" receiveTimeout="00:10:00" sendTimeout="00:01:00"
      allowCookies="false" bypassProxyOnLocal="false" hostNameComparisonMode="StrongWildcard"
      maxBufferSize="2147483647" maxBufferPoolSize="2147483647" maxReceivedMessageSize="2147483647"
      messageEncoding="Text" textEncoding="utf-8" transferMode="Buffered" useDefaultWebProxy="true">
      <readerQuotas maxDepth="32" maxStringContentLength="8192" maxArrayLength="16384" maxBytesPerRead="4096" maxNameTableCharCount="16384" />
      <security mode="None">
        <transport clientCredentialType="None" proxyCredentialType="None" realm="" />
        <message clientCredentialType="UserName" algorithmSuite="Default" />
      </security>
    </binding>
  </basicHttpBinding>
</bindings>
```

> 这些配置仅仅是放宽了传输通道的容量限制，但还不够。

## 2. 关键：配置 dataContractSerializer

WCF 传输自定义对象时，序列化器的对象图大小也有限制，需要配置 `dataContractSerializer`。

### 服务端 behavior 配置

```xml
<behavior name="GE_DemoService.GEDemoServiceBehavior">
  <serviceMetadata httpGetEnabled="true" />
  <serviceDebug includeExceptionDetailInFaults="true" />
  <dataContractSerializer maxItemsInObjectGraph="2147483647"/>
</behavior>
```

### 客户端 behavior 配置

```xml
<behaviors>
  <endpointBehaviors>
    <behavior name="endpoinmaxItemsInObjectGraph">
      <dataContractSerializer maxItemsInObjectGraph="2147483647"/>
    </behavior>
  </endpointBehaviors>
</behaviors>
```

客户端 endpoint 需指定 behavior：

```xml
<endpoint address="http://localhost:10081/GEDemoService/" binding="basicHttpBinding" bindingConfiguration="BasicHttpBinding_IGEDemoService" contract="DemoService.IGEDemoService" name="BasicHttpBinding_IGEDemoService" behaviorConfiguration="endpoinmaxItemsInObjectGraph" />
```

## 3. 补充：IIS 宿主需配置 httpRuntime

如果 WCF 以 IIS 作为宿主，还需在 `system.web` 节点下增加：

```xml
<httpRuntime maxRequestLength="512000" executionTimeout="120" />
```

- `maxRequestLength` 单位为 KB，最大 2097151（约 2GB）。
- 默认仅 4096KB（4MB），大数据量需显式增大。

---

## 4. 总结

- 仅仅放大 binding 的 buffer/size 不够，必须同时配置 dataContractSerializer 的 `maxItemsInObjectGraph`。
- IIS 宿主还需配置 `httpRuntime`。
- 本文方案已实测可传输 2000 条以上数据，原本 1000 条就会报错。

希望本文能帮到遇到同样问题的开发者，少走弯路！
