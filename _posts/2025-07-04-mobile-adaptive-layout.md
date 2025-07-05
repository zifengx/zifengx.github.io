---
title: 移动端自适应布局方案总结：rem、vw、rpx 的实践与比较
description: 随着移动设备种类的增多，如何实现页面在不同屏幕尺寸上的自适应展示，成为前端开发中不可回避的问题。本文将基于 rem 自适应布局的常用实践，结合 vw 和 rpx 等单位的方案，对比优劣，帮助你根据项目场景选择合适的方案。
date: 2025-07-03 22:00:00 -0400
categories: [前端, 移动端]
tags: [css, adaptive layout, 自适应, 移动端]
---

随着移动设备种类的增多，如何实现页面在不同屏幕尺寸上的自适应展示，成为前端开发中不可回避的问题。本文将基于 rem 自适应布局的常用实践，结合 vw 和 rpx 等单位的方案，对比优劣，帮助你根据项目场景选择合适的方案。

## 📐 一、rem 方案原理与实现

### rem 是什么？

`rem`（root em）是相对于根元素 `<html>` 的字体大小 `font-size` 来计算的单位。通过动态设置根元素的 `font-size`，可以让页面元素根据屏幕宽度等比缩放，达到响应式效果。

### 实现方式一：JavaScript 动态设置

```js
(function () {
  function setRemUnit() {
    const docEl = document.documentElement;
    const width = docEl.clientWidth;
    const rem = width / 10; // 设计稿为 375px 时推荐除以 37.5
    docEl.style.fontSize = rem + 'px';
  }

  setRemUnit();
  window.addEventListener('resize', setRemUnit);
})();
```

### 实现方式二：配合 PostCSS 自动转换 px→rem

使用 `postcss-pxtorem` 插件，在构建时自动将样式中的 `px` 转换为 `rem`，无需手写转换。

#### 安装：

```bash
npm install postcss-pxtorem --save-dev
```

#### 配置（postcss.config.js）：

```js
module.exports = {
  plugins: {
    'postcss-pxtorem': {
      rootValue: 37.5,
      propList: ['*'],
    },
  },
};
```

---

## 🔧 二、配套库：amfe-flexible（或 lib-flexible）

`amfe-flexible` 是阿里出品的适配库，会根据设备宽度自动设置 `<html>` 的 `font-size`，常与 `postcss-pxtorem` 配套使用。

#### 安装：

```bash
npm install amfe-flexible --save
```

#### 引入方式：

```js
import 'amfe-flexible';
```

---

## 📊 三、其他单位方案比较

| 单位  | 优点                 | 缺点                      | 场景推荐       |
| ----- | -------------------- | ------------------------- | -------------- |
| rem   | 自适应灵活、兼容性好 | 需额外设置 html font-size | Vue/React 项目 |
| vw/vh | 不依赖 JS，计算直观  | 字体缩放受限，兼容性差异  | H5 页面        |
| rpx   | 微信生态原生支持     | 仅适用于 uniapp、小程序   | 小程序、uniapp |

---

## 🖼️ 四、高分辨率图片处理方案

### 使用 CSS 媒体查询加载多倍图

```css
.bg {
  background-image: url(bg@1x.png);
}

@media only screen and (-webkit-min-device-pixel-ratio: 2) {
  .bg {
    background-image: url(bg@2x.png);
  }
}

@media only screen and (-webkit-min-device-pixel-ratio: 3) {
  .bg {
    background-image: url(bg@3x.png);
  }
}
```

### 使用 srcset 响应式图片

```html
<img src="logo@1x.png" srcset="logo@2x.png 2x, logo@3x.png 3x" alt="logo" />
```

---

## 🧩 五、uniapp 中的 rpx 实现

在 uniapp 中，`rpx` 是官方推荐单位，1rpx = 屏幕宽度的 1/750。

#### 使用方式：

```html
<view style="width: 200rpx; height: 100rpx; background-color: red;"></view>
```

#### 自动 px → rpx 转换：

```bash
npm install postcss-px2rpx --save-dev
```

配置 `postcss.config.js`：

```js
module.exports = {
  plugins: {
    'postcss-px2rpx': {
      baseDpr: 2,
      rpxUnit: 0.5,
    },
  },
};
```

---

## 🧠 总结

| 方案           | 优点              | 缺点               | 适用场景       |
| -------------- | ----------------- | ------------------ | -------------- |
| rem + flexible | 灵活、兼容性好    | 依赖 JS 和构建工具 | Vue/React 项目 |
| vw/vh          | 无需 JS，单位直观 | 字体缩放问题       | 静态页面       |
| rpx            | 微信生态支持好    | 局限在小程序       | uniapp、小程序 |
