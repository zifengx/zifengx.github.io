---
title: 如何使用 Web Serial API 自动连接串口设备
date: 2025-06-21 22:00:00 -0400
description: 介绍如何通过Web Serial API在网页中自动识别并连接串口设备，适用于物联网、硬件调试等场景。
categories: [前端, 硬件]
tags: [frontend, web, serial] # TAG names should always be lowercase
---

[Web Serial API](https://developer.mozilla.org/en-US/docs/Web/API/Web_Serial_API) 让网页可以直接访问本地串口设备，为物联网、硬件调试、设备管理等场景带来了极大便利。本文介绍如何在网页中自动识别并连接串口设备，并给出完整代码示例。

## 基本原理

1. **获取串口对象**
   - 通过 `navigator.serial.requestPort()` 让用户手动选择串口（需用户手势触发，如点击按钮）。
   - 通过 `navigator.serial.getPorts()` 获取用户已授权的网站可用的所有串口。

2. **监听串口设备的连接与断开**
   - 通过 `navigator.serial.addEventListener('connect', ...)` 监听新设备接入。
   - 通过 `navigator.serial.addEventListener('disconnect', ...)` 监听设备断开。

3. **打开串口**
   - 拿到 `SerialPort` 对象后，调用 `port.open({ baudRate: 9600 })` 打开串口。

## 代码示例

### 1. 用户手动选择串口

```js
// 需要用户点击按钮等手势触发
const button = document.querySelector('button');
button.addEventListener('click', async () => {
  // 弹出串口选择框
  const port = await navigator.serial.requestPort();
  // 打开串口
  await port.open({ baudRate: 9600 });
  // 这里可以继续进行读写操作
});
```

### 2. 自动获取已授权串口并连接

```js
// 获取所有已授权的串口
const ports = await navigator.serial.getPorts();
for (const port of ports) {
  if (!port.readable) {
    await port.open({ baudRate: 9600 });
    // 这里可以继续进行读写操作
  }
}
```

### 3. 监听串口设备的连接与断开

```js
// 监听新串口设备接入
navigator.serial.addEventListener('connect', async (event) => {
  const port = event.target;
  // 可自动打开串口或提示用户
  await port.open({ baudRate: 9600 });
  // 这里可以继续进行读写操作
});

// 监听串口设备断开
navigator.serial.addEventListener('disconnect', (event) => {
  const port = event.target;
  // 处理断开逻辑，如提示用户或清理资源
});
```

## 注意事项

- Web Serial API 目前仅在支持的 Chromium 内核浏览器（如 Chrome、Edge）中可用。
- 串口操作需 HTTPS 环境。
- `requestPort()` 必须由用户手势触发，不能自动弹出。
- 自动连接仅限于用户已授权过的串口。

## 参考资料

- [MDN Web Serial API](https://developer.mozilla.org/zh-CN/docs/Web/API/Serial)
- [Google Web Serial API 指南](https://web.dev/serial/)
