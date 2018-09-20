---
title: addEventListener的第三个参数
date: 2016-12-08 16:07:04
tags: javascript
categories: 前端
id: 1
---
之前一直忽略了`addEventListener`的第三个参数，因为一般情况下不写它并不会造成什么后果。但是，既然知道它有第三个参数，就想弄明白它是干嘛的。
<!-- more -->

`addEventListener`的第三个参数为`useCapture`，是一个`boolean`值，可以为`true`或`false`。也可以不写，不写的时候默认为`false`。当将它设置为`false`时，监听器将在`事件冒泡`（Bubble）阶段执行，也就是事件将由内向外执行，最具体的元素先接收事件，再逐级传递给它的父元素；当设置为`true`，监听器将在`事件捕获`（Capture）阶段执行，也就是事件将由外向内执行，与事件冒泡正好相反，它认为当某个事件发生时，父元素应该更早接收到事件，具体元素则最后接收到事件。当然，如果我们不是同时给父元素和子元素都设置了事件的话，这个属性的取值是不造成影响的。

下面看代码：

```html
<div class="outer">
	<div class="inner">inner</div>
	outer
</div>
```
当第三个参数为`false`时：
```js
document.querySelector('.outer').addEventListener('click',function () {
	console.log('outer');
},false);
document.querySelector('.inner').addEventListener('click',function () {
	console.log('inner');
},false);
```
![](/images/1/boxDemo.png)

当点击内层div，输出顺序如下：

![](/images/1/console1.png)


当第三个参数为`true`时：
```js
document.querySelector('.outer').addEventListener('click',function () {
	console.log('outer');
},true);
document.querySelector('.inner').addEventListener('click',function () {
	console.log('inner');
},true);
```
当点击内层div，输出顺序如下：

![](/images/1/console2.png)


*那么，当把外层div的第三个参数设置为`true`，内层div的第三个参数设置为`false`时，会是什么结果呢？*
```js
document.querySelector('.outer').addEventListener('click',function () {
	console.log('outer');
},true);
document.querySelector('.inner').addEventListener('click',function () {
	console.log('inner');
},false);
```
试试就知道了。当点击内层div，输出顺序如下（同上图）：

![](/images/1/console2.png)

为什么是这个结果呢？
先再来看看给多层div设置事件，并设置不同的`useCapture`的结果吧：

代码：
```html
<div class="red">
	<div class="yellow">
		<div class="green">
			<div class="blue"></div>
		</div>
	</div>
</div>
```
```js
document.querySelector('.red').addEventListener('click',function () {
	console.log('red');
},true);
document.querySelector('.yellow').addEventListener('click',function (event) {
	console.log('yellow');
},false);
document.querySelector('.green').addEventListener('click',function (event) {
	console.log('green');
},false);
document.querySelector('.blue').addEventListener('click',function () {
	console.log('blue');
},true);
```
![](/images/1/boxDemo2.png)

当点击最内层div时，控制台输出顺序如下：

![](/images/1/console3.png)

原因分析：

首先，我们需要了解DOM事件流的三个阶段：
1. 事件捕获阶段
2. 处于目标阶段
3. 事件冒泡阶段

当事件发生时，首先发生的是事件捕获，为父元素截获事件提供了机会。然后，事件到了具体元素时，在具体元素上发生，最后再发生事件冒泡。

*如图所示（图片源于网络，若侵权请告知）：*
![](/images/1/net.png)
所以，当我们点击最内层蓝色div时，最外层红色div最先捕获到事件，浏览器检测到其`useCapture`属性为`true`，即事件捕获阶段会触发事件，所以函数执行，输出red；浏览器接着检测次外层黄色div的`useCapture`属性，发现其为`false`，即事件捕获阶段不会触发事件，所以函数不执行；然后浏览器再检测绿色div的`useCapture`属性，发现其依然为`false`，所以函数依然不执行；
然后就到了目标阶段（蓝色div），不论其`useCapture`属性的值是多少，函数执行，输出blue。
接下来就是事件冒泡阶段了，由内至外进行`useCapture`属性的检测，如果其为`false`则执行函数，所以先输出green，再输出yellow，不再输出red。

当然，我们还可以同时给蓝色div设置阻止冒泡：
```js
document.querySelector('.blue').addEventListener('click',function (event) {
	event.stopPropagation();
	console.log('blue');
},false);
```
此时，当事件到达蓝色div时，就不再进行冒泡了，控制台输出如下：

![](/images/1/console4.png)

**end**

---

补充：最近刚看到的一般文章 [大佬的升级版](https://github.com/justjavac/the-front-end-knowledge-you-may-dont-know/issues/6)