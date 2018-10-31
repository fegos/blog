---
title: 那些年忽略的CSS
date: 2018-08-23
tags: css
author: oOjia
---

## 碎碎念
>CSS的学习入门很容易，只要1年甚至半年的时候，我们就能根据设计图迅速切出页面，能熟练使用些CSS hack，这个阶段我们的成长很快，每天都能汲取新知识。这实际上是CSS非常初级的阶段，也是广大页面仔们（包括我本人😳）最为浮躁，最自以为是，最觉得CSS不过如此的阶段。所以我们要增加学习的深度，深入掌握细节和原理，当我们对CSS的底层表现有一定的理解与认识的时候，遇到一些看似奇怪的问题，可以轻松搞定~

# 布局

我们在研究布局之前，首先需要了解几个很重要的概念和属性：

## 文档流
文档流指的是元素排版布局过程中，元素会自动从左往右，从上往下的流式排列。并最终窗体自上而下分成一行行，并在每行中从左至右的顺序排放元素。HTML中全部元素都是盒模型，盒模型占用一定的空间，依次排放在HTML中，形成了文档流。

`float`、`position：absolute|fixed`是脱离文档流的。脱离文档流是指，这个标签脱离了文档流的管理。不受文档流的布局约束了，并且更重要的一点是，当一个元素脱离文档流后，这个标签在原文档流中所占的空间也被清除掉了，依然在文档流中的其他元素将忽略该元素并填补其原先的空间。

#### 块级元素
- 每个块级元素都是独自占一行，其后的元素也只能另起一行，并不能两个元素共用一行。
- 元素的高度、宽度、行高和顶底边距都是可以设置的。
- 元素的宽度如果不设置的话，默认为父元素的宽度
- 常见的块级元素：`<div>、<p>、<h1>...<h6>、<ol>、<ul>、<dl>、<table>、<address>、<blockquote> 、<form>`

#### 行级元素
- 可以和其他元素处于一行，不用必须另起一行。
- 元素的高度、宽度及顶部和底部边距不可设置。
- 元素的宽度就是它包含的文字、图片的宽度，不可改变。
- 常见的行级元素:`<span>、<a>、<strong>、<em>`

#### 行级元素与块级元素的转换

- 可以在css样式中用display:inline将块级元素设为行级元素
- 同样，也可以用display:block将行级元素设为块级元素

#### display
是CSS中最重要的用于控制布局的属性，每个元素都有一个默认的值，这与元素的类型有关。对于大多数元素它们的默认值通常是`block`(块级元素)或`inline`(行内元素)。


`display:none` 和`visibility`属性不一样。`display:none`的元素不会占据它本来应该显示的空间，但是设置成`visibility: hidden;`还会占据空间。

#### 外边距折叠
如果两个相邻元素都在其上设置外边距，并且两个外边距接触，则两个外边距中的较大者保留，较小的一个消失

## 定位（position）

- static：默认值。元素框正常生成。块级元素生成一个矩形框，作为文档流的一部分；行内元素则会创建一个或多个行框，置于其父元素中。

- relative：元素框相对于之前正常文档流中的位置发生偏移，并且原先的位置仍然被占据。发生偏移的时候，可能会覆盖其他元素。

<div style="width: 700px;height:50px;line-height:50px;border:1px solid #ccc;position: relative;">这是一个设置了 position: relative; 的div</div>
<div style="width: 500px;padding:10px;border:1px solid #ccc;position: relative;top: -20px;left: 20px;">这是一个设置了position:relative; top:-20px; left:20px; width: 500px;的div</div>

- absolute：元素框不再占有文档流位置，并且相对于包含块进行偏移(所谓的包含块就是最近一级外层元素position不为static的元素)，找不到符合条件的父（祖先）节点，则相对浏览器窗口进行定位；没有设置TRBL，则默认浮动，默认浮动在父级节点的content-box区。
- fixed：属性的值为fixed的元素会相对于视窗来定位，即便页面滚动，它还是会停留在相同的位置。一个固定定位元素不会保留它原本在页面应有的空隙（即脱离文档流），但是移动设备兼容性不好。
- sticky：(这是css3新增的属性值)粘性定位，官方的介绍比较简单。其实，它就相当于relative和fixed混合。最初会被当作是relative，相对于原来的位置进行偏移；一旦超过一定阈值之后，会被当成fixed定位，相对于视口进行定位。

## 浮动（float）
最初，设计浮动时，其实并不是为了布局的，而是为了实现文字环绕的特效。类似于ps中的图层一样，浮动的元素会在浮动层上面进行排布，而在原先文档流中的元素位置，会被以某种方式进行删除，但是还是会影响布局，所以要记得清除浮动。

浮动为什么会被使用在布局中呢？因为设置浮动后的元素会形成BFC（使得内部的元素不会被外部所干扰），并且元素的宽度也不再自适应父元素宽度，而是适应自身内容。这样就可以，轻松地实现多栏布局的效果。浮动通常用于创建多个列布局，并且由于它有良好的浏览器兼容性，已经被使用了相当一段时间。

## flex布局
弹性布局：用来为盒状模型提供最大的灵活性，采用 Flex 布局的元素，称为 Flex 容器（flex container）。它的所有子元素自动成为容器成员，称为 Flex 项目（flex item）。

![](https://user-gold-cdn.xitu.io/2018/8/28/1657fbd8aaaa003a?w=563&h=333&f=png&s=10005)

容器默认存在两根轴：水平的主轴（main axis）和垂直的交叉轴（cross axis）。主轴的开始位置（与边框的交叉点）叫做main start，结束位置叫做main end；交叉轴的开始位置叫做cross start，结束位置叫做cross end。

项目(flex item)默认沿主轴排列。单个item占据的主轴空间叫做main size，占据的交叉轴空间叫做cross size。

### container(容器)的属性
#### <strong style="color:#ff7373">flex-direction</strong> 

属性决定主轴的方向（即item的排列方向）
- row（默认值）：主轴为水平方向，起点在左端。
- row-reverse：主轴为水平方向，起点在右端。
- column：主轴为垂直方向，起点在上沿。
- column-reverse：主轴为垂直方向，起点在下沿。

![](https://user-gold-cdn.xitu.io/2018/8/28/1657fd540c700b53?w=754&h=282&f=jpeg&s=34107)

#### <strong style="color:#ff7373">flex-wrap</strong> 
默认情况下，项目(item)都排在一条线（又称"轴线"）上。flex-wrap属性定义，如果一条轴线排不下，如何换行。
- nowrap（默认）：不换行。

![](https://user-gold-cdn.xitu.io/2018/8/28/1657fddece76af97?w=233&h=65&f=jpeg&s=5212)
- wrap：换行，第一行在上方。

![](https://user-gold-cdn.xitu.io/2018/8/28/1657fde4c7d6c2a0?w=233&h=106&f=jpeg&s=6790)
- wrap-reverse：换行，第一行在下方。

![](https://user-gold-cdn.xitu.io/2018/8/28/1657fde7af8596bb?w=233&h=110&f=jpeg&s=7204)

####  <strong style="color:#ff7373">flex-flow</strong>
`flex-flow`属性是`flex-direction`属性和`flex-wrap`属性的简写形式，默认值为`row nowrap`

#### <strong style="color:#ff7373">justify-content</strong>
`justify-content`属性定义了项目在主轴上的对齐方式。
```css
.box {
  justify-content: flex-start | flex-end | center | space-between | space-around;
}
```
- flex-start（默认值）：左对齐
- flex-end：右对齐
- center： 居中
- space-between：两端对齐，项目之间的间隔都相等
- space-around：每个项目两侧的间隔相等。所以，项目之间的间隔比项目与边框的间隔大一倍。

<img src="https://user-gold-cdn.xitu.io/2018/9/6/165aca3bac8168fa?w=902&h=499&f=jpeg&s=41688" width=800 style="display:block;margin:auto"/>

#### <strong style="color:#ff7373">align-items</strong>
`align-items`属性定义项目在交叉轴上如何对齐。
```css
.box {
  align-items: flex-start | flex-end | center | baseline | stretch;
}
```
具体的对齐方式与交叉轴的方向有关，下面假设交叉轴从上到下。
- flex-start：交叉轴的起点对齐。
- flex-end：交叉轴的终点对齐。
- center：交叉轴的中点对齐。
- baseline: 项目的第一行文字的基线对齐。
- stretch（默认值）：如果项目未设置高度或设为auto，将占满整个容器的高度。

<img src="https://user-gold-cdn.xitu.io/2018/8/28/1657fe28914dc972?w=617&h=786&f=png&s=8343" width=500 style="display:block;margin:auto"/>

#### <strong style="color:#ff7373">align-content</strong>
`align-content`属性定义了多根轴线的对齐方式。如果项目只有一根轴线，该属性不起作用
```css
.box {
  align-content: flex-start | flex-end | center | space-between | space-around | stretch;
}
```
- flex-start：元素位于容器的开头。各行向弹性盒容器的起始位置堆叠。弹性盒容器中第一行的侧轴起始边界紧靠住该弹性盒容器的侧轴起始边界，之后的每一行都紧靠住前面一行。
- flex-end：元素位于容器的结尾。各行向弹性盒容器的结束位置堆叠。弹性盒容器中最后一行的侧轴起结束界紧靠住该弹性盒容器的侧轴结束边界，之后的每一行都紧靠住前面一行。
- center：元素位于容器的中心。各行向弹性盒容器的中间位置堆叠。各行两两紧靠住同时在弹性盒容器中居中对齐，保持弹性盒容器的侧轴起始内容边界和第一行之间的距离与该容器的侧轴结束内容边界与第最后一行之间的距离相等。（如果剩下的空间是负数，则各行会向两个方向溢出的相等距离。）
- space-between：元素位于各行之间留有空白的容器内。各行在弹性盒容器中平均分布。如果剩余的空间是负数或弹性盒容器中只有一行，该值等效于'flex-start'。在其它情况下，第一行的侧轴起始边界紧靠住弹性盒容器的侧轴起始内容边界，最后一行的侧轴结束边界紧靠住弹性盒容器的侧轴结束内容边界，剩余的行则按一定方式在弹性盒窗口中排列，以保持两两之间的空间相等。
- space-around：元素位于各行之前、之间、之后都留有空白的容器内。各行在弹性盒容器中平均分布，两端保留子元素与子元素之间间距大小的一半。如果剩余的空间是负数或弹性盒容器中只有一行，该值等效于'center'。在其它情况下，各行会按一定方式在弹性盒容器中排列，以保持两两之间的空间相等，同时第一行前面及最后一行后面的空间是其他空间的一半。
- stretch（默认值）：元素被拉伸以适应容器。各行将会伸展以占用剩余的空间。如果剩余的空间是负数，该值等效于'flex-start'。在其它情况下，剩余空间被所有行平分，以扩大它们的侧轴尺寸。

![](https://user-gold-cdn.xitu.io/2018/8/28/16580076cc3b7d59?w=460&h=600&f=jpeg&s=47298)

### item(项目)的属性
#### <strong style="color:#ff7373">order</strong>
`order`属性定义项目的排列顺序。数值越小，排列越靠前，默认为0。
#### <strong style="color:#ff7373">flex-grow</strong>
`flex-grow`属性定义项目的放大比例，默认为0，即如果存在剩余空间，也不放大
如果所有项目的flex-grow属性都为1，则它们将等分剩余空间（如果有的话）。如果一个项目的flex-grow属性为2，其他项目都为1，则前者占据的剩余空间将比其他项多一倍。
<img src="https://user-gold-cdn.xitu.io/2018/8/29/165839dbaea97e27?w=802&h=211&f=png&s=7337" width=400 style="display:block;margin:auto"/>
#### <strong style="color:#ff7373">flex-shrink</strong>
`flex-shrink`属性定义了项目的缩小比例，默认为1，即如果空间不足，该项目将缩小。
如果所有项目的flex-shrink属性都为1，当空间不足时，都将等比例缩小。如果一个项目的flex-shrink属性为0，其他项目都为1，则空间不足时，前者不缩小。**负值对该属性无效**

#### <strong style="color:#ff7373">flex-basis</strong>
`flex-basis`属性用于设置或检索弹性盒伸缩基准值。定义了在分配多余空间之前，项目占据的主轴空间（main size）。浏览器根据这个属性，计算主轴是否有多余空间。它的默认值为auto，即项目的本来大小。

它可以设为一个长度单位或者一个百分比，规定灵活项目的初始长度。
```css
div:nth-of-type(2) {
    flex-basis: 80px;
}
```
<img src="https://user-gold-cdn.xitu.io/2018/8/29/16583a44d5f7240d?w=710&h=206&f=jpeg&s=13766" width=400 style="display:block;margin:auto"/>

#### <strong style="color:#ff7373">flex</strong>
`flex`属性是`flex-grow`, `flex-shrink` 和 `flex-basis`的简写，默认值为`0 1 auto`。后两个属性可选。
该属性有两个快捷值：`auto `(`1 1 auto`) 和 `none` (`0 0 auto`)。

建议优先使用这个属性，而不是单独写三个分离的属性，因为浏览器会推算相关值。

#### <strong style="color:#ff7373">align-self</strong>
`align-self`属性允许单个项目有与其他项目不一样的对齐方式，可覆盖`align-items`属性。默认值为`auto`，表示继承父元素的`align-items`属性，如果没有父元素，则等同于`stretch`。

该属性可能取6个值，除了auto，其他都与align-items属性完全一致。

- auto：默认值。元素继承了它的父容器的 align-items 属性。如果没有父容器则为 "stretch"。
- stretch：元素被拉伸以适应容器。如果指定侧轴大小的属性值为'auto'，则其值会使项目的边距盒的尺寸尽可能接近所在行的尺寸，但同时会遵照'min/max-width/height'属性的限制。
- center：元素位于容器的中心。弹性盒子元素在该行的侧轴（纵轴）上居中放置。（如果该行的尺寸小于弹性盒子元素的尺寸，则会向两个方向溢出相同的长度）。
- flex-start：元素位于容器的开头。弹性盒子元素的侧轴（纵轴）起始位置的边界紧靠住该行的侧轴起始边界。
- flex-end：元素位于容器的结尾。弹性盒子元素的侧轴（纵轴）起始位置的边界紧靠住该行的侧轴结束边界。
- baseline：元素位于容器的基线上。如弹性盒子元素的行内轴与侧轴为同一条，则该值与'flex-start'等效。其它情况下，该值将参与基线对齐。

![](https://user-gold-cdn.xitu.io/2018/8/29/16583b23053d5d96?w=1083&h=267&f=jpeg&s=42577)

## 布局技巧

### 单列布局

<img src="https://user-gold-cdn.xitu.io/2018/8/17/16547379f55886cf?w=580&h=410&f=jpeg&s=27413" width=400 style="display:block;margin:auto"/>

常见的这两种单列布局的特点都是**定宽**，**水平居中**的，设置width或max-width和margin:auto即可；

### 二列&三列布局

<img src="https://user-gold-cdn.xitu.io/2018/8/17/16547573001367ea?w=542&h=513&f=jpeg&s=34135" width=400 style="display:block;margin:auto"/>
二列布局的特征是侧边栏固定宽度，主栏自适应宽度。

三列布局的特征是左右两侧固定宽度，中间列自适应宽度。

#### 1.float + margin
将两个侧边栏分别向左向右浮动，通过设置中间的主栏的margin为它们留出空间，形成三列布局
```css
.left{
    float: left;
    width: 200px;
}
.right{
    float: right;
    width: 200px;
}
.main{
    margin-left: 220px; /*预留出定宽两栏的空间*/
    margin-right: 220px;
}
```
只设置一个浮动即可得到两列布局
#### 2.position + margin
通过将两个侧边栏的position设置为absolute，然后将左边栏的left设置为0，右边栏的right设置为0，主栏设置margin为边栏留出空间，即可得到三列布局。
```css
.left{
    position: absolute;
    left: 0;
    width: 200px;
}
.right{
    position: absolute;
    right: 0;
    width: 200px;
}
.main{
    margin-left: 220px;
    margin-right: 220px;
}
```
同样，将定位元素改为一个可以得到两列布局。

#### 圣杯布局
- 三者都设置向左浮动。
- 设置main宽度为100%，设置两侧栏的宽度。
- 设置负边距，left设置负左边距为100%，right设置负左边距为负的自身宽度。
- 设置main的padding值给左右两个子面板留出空间。
- 设置两个子面板为相对定位，左边栏的left值为负的自身宽度，右边栏的right值为负的自身宽度。
```html
 <div class="container">         
    <div class="main"></div>        
    <div class="left"></div>        
    <div class="right"></div>  
</div>
```
```css
.main{
    float: left;       
    width: 100%;   
}
.left{     
    float: left;        
    width: 200px;        
    margin-left: -100%;               
    position: relative;  
    left: -200px; 
}
.right{
    float: left;        
    width: 300px;        
    margin-left: -300px; 
    position: relative; 
    right: -300px;  
}
.container {        
    padding: 0 300px 0 200px;   
}
```

二列的实现方法：

如果是左边带有侧栏的二列布局，则去掉right，不设置主面板的padding-right值。

####  双飞翼布局
双飞翼布局和圣杯布局的思想有些相似，都利用了浮动和负边距，但双飞翼布局在主栏外加了一层div并设置margin用来容纳侧栏，两侧栏的负边距都是相对于外层div而言，main的margin值变化便不会影响两个侧栏，这样就省略了将侧栏拉到主栏那一行后进行的relative定位（因为双飞翼布局留白就在父元素的内容区，而圣杯布局的留白在父元素内容区之外）。
```html
<div class="wrapper">
      <div class="main"></div>
</div>
<div class="left"></div>        
<div class="right"></div>
```
```css
.wrapper {        
    float: left;       
    width: 100%;
}  
.main {    
    margin: 0 300px 0 200px;
}
.left {       
    float: left;        
    width: 200px;        
    margin-left: -100%;   
}   
.right {        
    float: left;        
    width: 300px;        
    margin-left: -300px; 
 }
```
- 圣杯采用的是padding，而双飞翼采用的margin，解决了圣杯布局main的最小宽度不能小于左侧栏的缺点。
双飞翼布局不用设置相对布局，以及对应的left和right值。简单说起来就是“双飞翼布局比圣杯布局多创建了一个div，但不用相对布局了”。
- 如果引入相对布局，可以实现三栏布局的各种组合，例如对右侧栏设置position: relative; left: 200px;,可以实现left+right+main的布局。

二列的实现方法：

如果是左边带有侧栏的二栏布局，则去掉右侧栏，不要设置main-wrap的margin-right值，其他操作相同。反之亦然。
####  flex布局
Flex 是 Flexible Box 的缩写，意为“弹性布局”，用来为盒状模型提供最大的灵活性。除了在PC端兼容性较差，没有太大的缺陷，多用于移动端布局。

```css
.layout {
    display: flex;
}
.main {
    flex: 1;
}
.aside {
    width: 200px;
}
```

![](https://user-gold-cdn.xitu.io/2018/8/20/16556d8c7af65230?w=2558&h=1190&f=png&s=169081)


### 常用居中方法

#### margin: auto;

<div style="width: 700px;height:50px;line-height:50px;text-align:center;margin: 0 auto;border:1px solid #ccc">这是一个设置了
  width: 600px;
  margin: 0 auto; 
的div</div>

这是大家很常见的居中方式，元素会占据你所指定的宽度，然后剩余的宽度会一分为二成为左右外边距。**不过问题是**，当浏览器窗口比元素的宽度还要窄时，浏览器会显示水平滚动条。

在这种情况下使用 max-width 替代 width 可以使浏览器更好地处理小窗口的情况。这点在移动设备上显得尤为重要~

<div style="max-width: 700px;height:50px;line-height:50px;text-align:center;margin: 0 auto;border:1px solid #ccc">这是一个设置了
  max-width: 600px;
  margin: 0 auto; 
的div</div>

调整窗口大小看一下两者的区别吧
居中在布局中很常见，我们假设DOM文档结构如下，子元素要在父元素中居中：
```html
<div class="parent">
    <div class="child"></div>
</div>
```
#### 水平居中
- **子元素为行内元素**：对父元素设置text-align:center;
- **子元素为定宽块状元素**: 设置左右margin值为auto;
- **子元素为不定宽块状元素**: 设置子元素为display:inline,然后在父元素上设置text-align:center;
- **通用方案**: flex布局，对父元素设置display:flex;justify-content:center;

#### 垂直居中
垂直居中对于子元素是单行内联文本、多行内联文本以及块状元素采用的方案是不同的。

- **父元素一定，子元素为单行内联文本**：设置父元素的height等于行高line-height
- **父元素一定，子元素为多行内联文本**：设置父元素的display:table-cell或inline-block，再设置vertical-align:middle;
- **块状元素**:设置子元素position:absolute 并设置top、bottom为0，父元素要设置定位为static以外的值，margin:auto;
- **通用方案**: flex布局，给父元素设置{display:flex; align-items:center;}。
