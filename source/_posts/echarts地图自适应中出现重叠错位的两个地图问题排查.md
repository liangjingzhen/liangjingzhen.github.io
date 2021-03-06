---
title: echarts地图自适应中出现重叠错位的两个地图问题排查
date: 2019-04-27 21:10:41
categories:
- 可视化
tags:
- echarts
---

项目的可视化模块中使用了`echarts`作为图表库。当对散点类地图作自适应开发时，发现在对地图进行偏移或缩放的时候，画布上出现了重叠并且错位的两个地图。

## 问题现象

![offset](/images/echarts/offset.png)

![zoom](/images/echarts/zoom.png)

上图是对地图设置偏移出现的情况，下图是对地图放大出现的情况。为突出对比效果，图中用蓝、黑两种颜色对地图边框进行了着色。

下面分别给出项目中对地图设置偏移和缩放的关键代码，其中的（配置项）主要字段是`geo`和`series`：
```javascript
// 偏移
option = {
    geo: {
       left: 450 // 设置偏移的字段
    },
    series: [
        // ...
    ]
}

// 放大
option = {
    geo: {
        zoom: 1.1 // 设置缩放的字段
    },
    series: [
        // ...
    ]
}
```
正如以上代码所示，根据以往经验，对地图进行偏移或缩放只需在`option`的`geo`属性中设置对应字段即可，但显然在当前环境中如此配置并没有达到预期。那么上述现象是由于代码配置项不恰当引起的问题，还是echarts库本身的问题呢？

## 问题排查

因为项目中的代码配置项非常多，直接从代码里排查问题比较困难，所以先将其与echarts社区中的示例来作对比，确定是否是echart库的问题。

首先对社区示例代码[点此查看](https://gallery.echartsjs.com/editor.html?c=xSkUQiHdBz)分别进行偏移和缩放设置，发现显示完全正常，并没有出现重叠或错位现象。

既然官方示例没有问题，那是不是自己代码中的`option`选项哪里设置不对了？当把项目中的`option`放到示例代码中时，现象出现了，现在可以确定是设置的问题，而不是`echarts`的问题了。

实际上，项目的代码里已经有人给出了避免对地图画布进行定位时引起地图重影的方法，即同时给`geo`和`series`设置相同的偏移量。但是原因是什么呢？看来需要深入`echarts`项目文档查找答案了。

在`echarts`的[github]()项目issues中查找关于地图重叠的问题时，发现了`map`类型的一个参数：`geoIndex`，只要将其值设置为`0`即可。虽然不是最终想要的答案，但却得到了`geoIndex`这条线索。

打开`echarts`的配置项手册，查看`series-map`的`geoIndex`选项，发现如下描述：
> 默认情况下，map series 会自己生成内部专用的`geo`组件。但是也可以用这个`geoIndex`指定一个`geo`组件。
> ...
> 当设定了`geoIndex`后，series-map.map 属性，以及 series-map.itemStyle 等样式配置不再起作用，而是采用`geo`中的相应属性。

原来`geo`和`series-map`是两个相互独立的组件，所以最初代码只对`geo`进行偏移或缩放设置时，会出现重叠的地图。因此当二者同时使用时，要么（一）同时设置缩放或者偏移量，要么（二）给`series-map.map`指定关联的`geo`组件，才能让画布上的地图正常显示。考虑到项目允许对可视化组件进行自定义配置，所以最后采用了第一种方案。

修改后的代码：
```javascript
option = {
    geo: {
        zoom: 1.1
    },
    series: [
        // ...
        {
            zoom: 1.1
        }
    ]
}
```

![正常缩放的地图](/images/echarts/normal.png)

## 总结

成熟的工具库或者框架可以极大地方便项目开发，但是使用过程中难免会遇到各种各样的问题，导致开发中断甚至延误项目进度。回过头来看排查过程，这个问题还是由于对使用文档不熟悉，不了解API的使用造成的。所以使用三方库之前一定要理解其原理，并熟练掌握各种API，而且一定要仔细读官方文档，因为很多问题的答案都在其中。
