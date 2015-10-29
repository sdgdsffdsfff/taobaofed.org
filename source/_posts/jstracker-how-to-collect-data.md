---
title: JSTracker 之前端异常数据采集
categories: 
  - 工具&平台
tags:
  - JSTracker
  - 监控
date: 2015-10-28 07:11:43
author: 溪夏
published: true
---

![JSTracker 之前端异常数据采集](http://gtms01.alicdn.com/tps/i1/TB1wsEtKXXXXXXTXXXXlzJZ8pXX-900-500.jpg)

## JSTracker - 淘宝前端监控平台

基本上服务器端的代码都是处于 7x24 小时的实时监控状态的，一旦有任何异常对应的开发同学就马上收到报警，并且第一时间处理。 但是对于前端来说，往往是实际用户那里的脚本报错后才知道页面出现异常，这时候已经是故障了。

为了让前端也能和后端一样，需要将线上的 JavaScript 代码监控起来，当用户端浏览器出现异前端第一时间被通知到。于是便有了淘宝前端的监控平台：JSTracker。


### 采集哪些数据

在解决怎么采集之前，还需要解决采集哪些数据，哪些数据是有用的？

主要原则就是避开用户敏感字段，采集浏览器版本、操作系统版本、报错的 msg 信息等。

### JavaScript 异常的时候捕获异常

主动捕获异常方案主要是 onError 和 addEventListener，
onError 在 IE6 开始就支持了，所以 JSTracker 的主动采集是使用的 onError。

onError 可以采集到 file、line、col 等信息，但是实际情况中却大部分收集到的是 script error。
![收集到的 script error](http://gtms03.alicdn.com/tps/i3/TB15K6KKXXXXXXTaXXXbb5uUFXX-1166-360.png)

原因是浏览器的同源性策略（CROS），在高级浏览器中如果浏览器捕获到了错误信息，如果 JS 文件所在的域名（如：g.alicdn.com）和当前的页面地址（如：www.taobao.com）是跨域的，那么浏览器会把 onError 中的 msg 替换为 script error，其余字段也会做替换。

webkit 的源代码：
![webkit 的源代码](http://gtms04.alicdn.com/tps/i4/TB19rzGKXXXXXXAaXXX5JvL5XXX-966-459.png)

Script error 这个问题也是目前大家最诟病的一个问题：当报警发生之后，我们只知道有问题，但是很难知道哪里出了问题。


其实是有解决方案的：

 * 首先 JavaScript 请求的 http 返回头上需要加上一个 Access-Control-Allow-Origin 头。
 ![Access-Control-Allow-Origin 头](http://gtms01.alicdn.com/tps/i1/TB1cdz2KXXXXXXqXFXXusByLVXX-712-390.png)

 * 在引入 JavaScript 文件的时候需要在 script 标签上添加 crossorigin 属性，在加这个属性前请一定确保上一条已经完成，否则 JavaScript 文件不会执行！！！。

```
<script src="http://somremotesite.example/script.js" crossorigin></script>
```

这个解决方式看起来完美，但是要做的改造实在是太多了，目前还是主动上报为主。

### 主动上报异常

onError 的方案会采集到全面的浏览器报错，但是太全了，出了刚才的 script error 问题，还会采集到。

 * ISP 在页面中注入的脚本报错（我们的前端质量很高的，但是被这些牛氓们一弄，啥质量都没了）。
 * 插件问题（乱七八糟的插件也是一样的令人讨厌，注入奇怪的东西在页面中，引起报错）。

为了让采集到的错误更有价值，需要暴露接口给让前端门自己在代码中上报异常（作为一名合格的代码工程师，一定要有抛异常的习惯呢）。

现在我们提供了自定义上报的接口，没错就是这么简单，你不用担心 JSTracker 脚本有没有加载，只要在你的代码中直接用就行了。

```
window.JSTracker2 = window.JSTracker2 || [];
try{
    //your code
}catch(e){
    JSTracker2.push({
      msg: "xx_api_failed"
    });
}
```

自己上报的错误对于定位也是很容易的：
![自己上报的错误](http://gtms02.alicdn.com/tps/i2/TB1ROPIKXXXXXauaXXX58beWpXX-1172-322.png)

### 一些细节问题

#### 在代码中使用 try catch 是否对性能有影响

try catch 对性能的影响微乎其微，但是一些用法会让性能受很大的影响， 参考这个实验：http://taobaofed.org/blog/2015/10/28/try-catch-runing-problem/

总结下来就是：在 try 语句块中不要定义太多的变量，最好是只写一个函数调用，避免 try 运行中变量拷贝造成的性能损耗。类似的不只是 try，定义 function 也是一样的。

#### 采集到的数据如何发出去

JSTracker 的数据发送了非常大，这块都交给后端帮我们处理，采集部分只需要关注准确无误的把数据发送出去。

最初的简单的发送方案，直接用 GET 请求，将参数拼接在 URL 后面：

```javascript
var url = 'xxx';
new Image().src = url;
```
后来发现这样发送有一定概率丢失数据,当浏览器回收内存的时候这个请求是发不出去的，所以要将这个变量 hold住：

```javascript

  var win = window;
  var n = 'jsFeImage_' + _make_rnd(),
    img = win[n] = new Image();
  img.onload = img.onerror = function () {
    win[n] = null;
  };
  img.src = src;

```
#### 随机数造成的数据丢失

我们为了防止缓存，经常会用毫秒的时间作为随机数（如：+new Date()），但是在极端情况下可能1ms就会发出两条 log，这样第二条 log 就会丢失。

```
var _make_rnd  = function(){
    return (+new Date()) + '.r' + Math.floor(Math.random() * 1000);
  };
```

#### 360浏览器识别

为了能更好的统计这个浏览器，需要识别360SE、360EE。众所周知 3Q 大战之后360的 UA 就和 Chrome 的一样，所以我们要用一些特别的方法识别它们。目前为止能够识别的方案，这个方案会随时更新，适应360的变化。

```javascript
var is360 = function(){
  try {
    if(/UBrowser/i.test(navigator.userAgent)){
      return '';
    }

    if (typeof window.scrollMaxX !== 'undefined') {
      return '';
    }

    var _track = 'track' in document.createElement('track');
    var webstoreKeysLength = window.chrome && window.chrome.webstore ? Object.keys(window.chrome.webstore).length : 0;

    if (window.clientInformation && window.clientInformation.languages && window.clientInformation.languages.length > 2) {
      return '';
    }

    if (_track) {
      return webstoreKeysLength > 1 ? ' QIHU 360 EE' : ' QIHU 360 SE';
    }

    return '';
  }catch(e){
    return '';
  }
}();
```
