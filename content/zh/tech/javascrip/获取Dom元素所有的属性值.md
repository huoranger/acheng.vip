---
title: "获取DOM元素的属性值"
date: "2022-05-01T00:00:00+08:00"
slug: "get-dom-properties-value"
tags: ["JavaScript", "JQuery"]
---

在魔改 Hexo Theme 时，需要用到 JavaScript 获取 CSS 属性，但对于 Web 前端的知识我一点也不熟，Google 了一下发现有以下几种方式。<!--more-->

* 原生 JavaScript 方式

```javascript
const container = document.querySelector('.container');
const containerWidth = container.style.width;
const containerLineHeight = container.style.lineHeight;
console.log(containerWidth,containerWidth);
```

该方式只能获取写在标签上的属性，而不能获取写在 `<style></style>` 上的 CSS 属性，推荐以下方式：

```javascript
val = window.getComputedStyle(document.querySelector('.container'),null)["lineHeight"];  
```

* JQuery 方式

```javascript
const containerWidth = $('.container').css('width');
const containerLineHeight = $('.container').css('line-height');
console.log(jq_width,jq_lineHeight);
```



