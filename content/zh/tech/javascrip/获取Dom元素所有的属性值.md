+++

title = "获取DOM元素的属性值"
date = "2022-05-01T00:00:00+08:00"

slug = "get-dom-properties-value"

tags=["javascript", "jquery"]

+++



在魔改 `hexo theme` 时，需要用到 javascript 获取 css 属性，查阅资料特此记录。<!--more-->

* 原生 Javascript 方式

```javascript
const container = document.querySelector('.container');
const containerWidth = container.style.width;
const containerLineHeight = container.style.lineHeight;
console.log(containerWidth,containerWidth);
```

该方式只能获取写在标签上的属性，而不能获取写在 `<style></style>` 上的 `css` 属性，推荐一下方式：

```javascript
val = window.getComputedStyle(document.querySelector('.container'),null)["lineHeight"];  
```

* Jquery 方式

```javascript
const containerWidth = $('.container').css('width');
const containerLineHeight = $('.container').css('line-height');
console.log(jq_width,jq_lineHeight);
```



