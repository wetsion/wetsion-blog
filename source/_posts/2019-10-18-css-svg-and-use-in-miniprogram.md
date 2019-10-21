---
title: CSS：SVG学习及在微信小程序中的简单运用
description: 学习SVG并在微信小程序中实践
categories:
 - 前端
tags:
 - css
 - 微信小程序
---



&emsp;&emsp;之前简单学习了svg，但无用武之地，很快就忘了，最近在写小程序项目时，想要实现一个上下的边框粗细渐变的一个效果，类似如下图：

![WechatIMG5.png](https://i.loli.net/2019/10/18/b4ZotNqL8u5JPWy.png)

&emsp;&emsp;就类似这样，在上下各有一条粗细渐变的细线，不知如何实现，于是想到使用svg来试试。

---

&emsp;&emsp;首先先了解学习下SVG。

> （参考阮一峰的博客：https://www.ruanyifeng.com/blog/2018/08/svg.html）

&emsp;&emsp;SVG其实是一种图像格式，基于xml语法，官方名称是可缩放矢量图，相对于其他图像格式是基于像素处理，它则是基于对图像的形状描述，所以特点就是不论放大多少倍都不会失真。

&emsp;&emsp;SVG可以写在网页中，直接在网页上显示；也可以写在`.svg`文件中，通过`<img>`、`<iframe>`、`<object>`等标签引用插入在网页中，或者在css中通过`url()`引入；也可以转为base64编码，作为data uri引入（在小程序中我就是采用这种方式）。

&emsp;&emsp;由于SVG是基于xml语法，所以SVG的语法主要就是svg提供的一些标签，主要有以下标签：

 - ###  `<svg>`

&emsp;&emsp;最外层的当然就是`<svg>`标签，提供了三个属性：

1. `width`
2. `heigth`

&emsp;&emsp;`<svg>`的宽度和高度，表示svg图像在html元素中占据的高度宽度，如果是百分比，则是相对位置，如果是数字，就是单位是像素的绝对位置，如果未指定这两个属性，svg图像默认是宽300px 高150px。

3. `viewBox`

&emsp;&emsp;表示svg图像的视图大小，比如只想展示svg图像的一部分，注意这里是视图大小不是svg的大小，svg图像还是上面俩属性设置的大小，但展示的可能只是svg图像的一部分，viewBox设置四个数字，分别是左上角的横纵坐标、视图的宽高，比如svg宽高是100，viewBox设置 0 0 50 50，那么整个svg只显示原来50 x 50的图像，也就是左上角四分之一被放大了四倍。


 - ### `<polyline>`

&emsp;&emsp;用于绘制一根折线，有以下属性：

1. **`points`** 主要属性，指定折线中的各个点的坐标，通过依次连接这些点构成一条折线，坐标横纵坐标用英文逗号分隔，点之间用空格分隔

2. `fill` svg中很多标签都有的属性，指定填充的颜色，支持颜色名、16进制颜色、rgb颜色，这里<polyline>就是个折线，不存在填充，所以一般设置none，如果不设置为none，那么将会**默认填充黑色**，就和下面`polygon`标签等效了

3. `stroke` 同样svg很多都有的属性，指定描边线条的颜色，也支持颜色名、16进制颜色、rgb颜色，**默认无颜色**

4. `stroke-width` 指定描边线条的宽度


 - ### `<polygon>`

&emsp;&emsp;用于绘制多边形，属性同上面`<polyline>`，通过将属性`points`配置的点连接起来形成一个闭合的多边形，`fill`指定多边形填充的颜色，如果不配置，默认黑色

 - ### `<line>`

&emsp;&emsp;用于绘制直线，属性`x1`和`y1`分别指定起点的横纵坐标，属性`x2`和`y2`分别指定终点的横纵坐标；fill和stroke属性同上述标签

 - ### `<circle>`
 - ### `<ellipse>`
 - ### `<rect>`

&emsp;&emsp;这三个标签有点类似，`<circle>`通过属性 `cx`和`cy`确定圆心的横纵坐标，再通过属性`r`确定圆的半径，进行绘制圆形，fill、stroke、stroke-width属性同上述标签；`<ellipse>`通过属性`cx`和`cy`确定圆心的横纵坐标，再通过属性`rx`和`ry`指定椭圆横向轴和纵向轴的半径，完成绘制椭圆形，fill、stroke、stroke-width属性同上述标签；`<rect>`通过属性`x`和`y`确定矩形左上角点的横纵坐标，再通过`width`和`height`确定矩形宽和高，完成绘制矩形，fill、stroke、stroke-width属性同上述标签。


 - ### `<path>`

&emsp;&emsp;用于制路径

 - ### `<text>`

&emsp;&emsp;用于绘制文本，属性`x`和`y`代表文本区块基线起点的横纵坐标

 - ### `<use>`

&emsp;&emsp;用于复制一个形状，属性`href`指定被复制的形状节点，属性`x`和`y`设置左上角的坐标

 - ### `<g>`

&emsp;&emsp;用于将多个形状组成一个组，方便复用

 - ### `<defs>`

&emsp;&emsp;用于自定义形状，内部代码不会显示，仅供引用

 - ### `<animate>`

&emsp;&emsp;用于制作动画效果，用在形状标签内，让该形状产生动画效果，属性`attributeName` 发生动画效果的属性名，属性`from`单次动画的初始值，属性`to`单次动画的结束值，属性`dur`单次动画的持续时间，属性`repeatCount`动画的循环模式（indefinite无限循环）

 - ### `<animateTransform>`

&emsp;&emsp;上面`<animate>`对css的transform不起作用，如果要变形、旋转，就使用该标签。


---

&emsp;&emsp;学习了svg，那么怎么实现开头说的那个效果呢，首先写好一段svg，再将svg转成base64编码，作为data uri在小程序的样式中，被background-image引用，代码如下：


```html
<view class="id">
    <view class="border-line"></view>
    <view>
        <text>会员ID:{{userId}}</text>
        <text>邀请码:{{userInviteCode}}</text>
    </view>
    <view class="border-line"></view>
</view>
```

&emsp;&emsp;border-line样式的view就是我打算显示渐变边框的地方，样式如下：

```css
.avatar-name-div .name-div .id .border-line{
  height: 2px;
  width: 100%;
  background-image: url("data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg' height='100%25' width='100%25'%3E %3Cpolygon points='0,1 125,0 250,1 125,2 0,1' fill='%23e9ded8'/%3E %3C/svg%3E");
}
```

最终实现效果如下：

![WX20191018-173241@2x.png](https://i.loli.net/2019/10/18/4J9hmdZVjal7bq1.png)

&emsp;&emsp;好了，关于svg的学习和实践就先记录到此，svg来实现画一些图形确实还是挺不错的，但也让我延伸想到canvas，后面有时间再了解了解canvas。



