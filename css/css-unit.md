---
title: h5屏幕适配
date: 2021-9-31
tags:
  - css
---

# 背景

UI给出的设计稿，一般是以iphone6屏幕大小为准，也就是宽高为375 * 667，激进一些的还会使用414 * 736的设计稿。在其他不同尺寸的屏幕上适配的问题也就由此诞生，一般新搭建的项目都要处理这个问题。



# 常见方案

目前市面上比较常见的有两种方案：rem方案、viewport方案

1. rem方案

   通过计算屏幕宽度比，对html标签设置font-size为px，子元素都使用rem标识

   ```html
   <script type="text/javascript">
     !(function e () {
       var t = document.documentElement,
           n = Math.min(t.clientWidth, 500) / 4.14;
       t.style.fontSize = n + 'px';
       var i = parseFloat(window.getComputedStyle(t).fontSize);
       n !== i && ((n = (n * n) / i), (t.style.fontSize = n + 'px')), (window.__REAL_FONT_SIZE__ = n), window.addEventListener('resize', e);
     })();
   </script>
   ```

   缺点：rem单位会可能会出现精度的兼容性问题，可能出现1px未对齐的问题，即半像素问题

2. viewport方案

   通过更改viewport的 initial-scale=1.0, maximum-scale=1.0 属性控制缩放

   ```html
   <meta name="viewport" content="width=device-width,initial-scale=1,user-scalable=no"/>
   <script>
     ;+(function () {
       function resize() {
         var viewport = document.querySelector('meta[name=viewport]');
         viewport.content = viewport.content.replace(/(initial-scale=)[^,]*/i, '$1'.concat(screen.width / 818))
      	}
      	// 动态调整视口宽度至设计稿宽度
      	window.addEventListener('resize', function () { setTimeout(resize, 300) });
      	resize()
     })();
   </script>
   ```

   优点：不会出现像素不对齐的问题 缺点：视窗缩放时，相当于渲染出了一个大尺寸页面进行缩放，会占用webview内存，影响性能

**观察发现：这两种方案，都需要借助JS实现页面的屏幕适配，有点像补丁代码，对于代码洁癖的人，肯定想除之而后快，所以就想着实现一种纯CSS方案来实现布局适配**



# 纯css方案

该方案与第一种方案类似，不过不是通过JS来控制根节点的`fontSize`，而是引入新的单位`vw`来实现。通过设置根节点单位为`vw`，实现根节点在不同尺寸屏幕下的相同占比；子节点使用单位`rem`，实现根据根节点大小变化调整对应大小。

`vw`字面意思是视窗宽度，将整个视窗划分为100vw，所以`vw`类似百分比，该方案正是利用这个特性实现不同屏幕下的布局适配。

根据设计稿可知， `375px = 100vw`，

如果将根节点`fontSize`设置为100px，所以，根节点html

`fontSize = 100px * 100vw/375px = 26.66666667vw`

为了兼容不支持`vw`的机型，可以先设置px，再用vw进行覆盖，具体如下：

```css
html {
  font-size: 100PX; // 大写的PX，不会被postcss-pxtorem转化为rem
  font-size: 26.66666667vw;
}
```

子节点通过插件`postcss-pxtorem`，在webpack构建时，会把实际写样式时用的`px`转换为`rem`，具体配置如下：

```json
'postcss-pxtorem': {
  rootValue: 100,
  propList: ['*'],
  selectorBlackList: ['__vconsole']
}
```



