---
title: JSTracker-前端监控平台
categories: 
  - 工具&平台
tags:
  - JSTracker
  - 监控
date: 2015-10-28 07:11:43
author: 溪夏
published: true
---

![JSTracker-前端监控平台](http://gtms03.alicdn.com/tps/i3/TB1R8BeGpXXXXX_XVXXjpEL0VXX-396-240.jpg)

## JSTracker-前端监控平台

一直以来我们的后端代码都是处于7*24小时的实时监控状态的，一旦有任何异常，对应的开发同学就马上收到报警，第一时间处理异常。 但是对于前端来说，我们大部分情况是最后才知道页面出现异常的，这时候已经是故障了。

为了让前端也能和后端一样，将线上的Javascript代码监控起来，异常发生的时候前端要第一个知道，于是便有了前端的监控平台：JSTracker


### 采集哪些数据

在解决怎么采集之前，还需要解决采集哪些数据，哪些数据是有用的

主要原则就是避开用户敏感字段，采集浏览器版本、操作系统版本、报错的msg信息等

### Javascript异常的时候捕获异常

主动捕获异常方案，主要是onError和addEventListener
onError在ie6开始就支持了，所以JSTracker的主动采集是使用的onError

onError可以采集到file、line、col等信息，但是实际情况中却大部分收集到的是script error
![image](http://gtms03.alicdn.com/tps/i3/TB15K6KKXXXXXXTaXXXbb5uUFXX-1166-360.png)

原因是浏览器的同源性策略（CROS），在高级浏览器中如果浏览器捕获到了错误信息，如果js文件所在的host（如：g.alicdn.com）和当前的页面地址（如：www.taobao.com）是跨域的，那么浏览器会把onError中的msg替换为Script error. 其余字段也会做替换。
webkit的源代码：
![image](http://gtms04.alicdn.com/tps/i4/TB19rzGKXXXXXXAaXXX5JvL5XXX-966-459.png)

script error这个问题也是目前大家最诟病的一个问题：当报警发生之后，我们只知道有问题，但是很难知道哪里出了问题。


其实有解决方案的：

 * 首先CDN的http头山更需要加上一个http头 Access-Control-Allow-Origin。最近CDN已经加上了这个头。
![image](http://gtms01.alicdn.com/tps/i1/TB1cdz2KXXXXXXqXFXXusByLVXX-712-390.png)

 * 在引入JS文件的时候需要在script标签上添加crossorigin属性，这个属性添加请一定确保上一条已经完成，否则js文件不会执行！！！。

```
<script src=”http://somremotesite.example/script.js” crossorigin></script>
```

这个解决方式看起来完美，但是要做的改造实在是太多了，目前还是主动上报为主。

### 主动上报异常

onError的方案会采集到全面的浏览器报错，但是太全了，出了刚才的script error问题，还会采集到

 * ISP在页面中注入的脚本报错（我们的前端质量很高的，但是被这些牛氓们一弄，啥质量都没了）
 * 插件问题（乱七八糟的插件也是一样的令人讨厌，注入奇怪的东西在页面中，引起报错）

为了让采集到的错误更有价值，需要暴露接口给让前端门自己在代码中上报异常（作为一名合格的代码工程师，一定要有抛异常的习惯呢）。

现在我们提供了自定义上报的接口,没错就是这么简单，你不用担心JSTracker脚本有没有加载，只要在你的代码中直接用就ok了。
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
自己上报的错误对于定位也是很容易的
![image](http://gtms02.alicdn.com/tps/i2/TB1ROPIKXXXXXauaXXX58beWpXX-1172-322.png)

### 一些细节问题

#### 在代码中使用try catch是否对性能有影响

try catch对性能的影响微乎其微，但是一些用法会让性能受很大的影响， 参考这个实验：http://
xx

总结下来就是：在try 语句块中不要定义太多的变量，最好是只写一个函数调用，避免try运行中变量拷贝造成的性能损耗。类似的不只是try，定义function也是一样的。

#### 采集到的数据如何发出去

JSTracker的数据发送了非常大，这块都交给后端帮我们处理，采集部分只需要关注准确无误的把数据发送出去。

最初的简单的发送方案，直接用GET请求，将参数拼接在URL后面：

```javascript
var url = 'xxx';
new Image().src = url
```
后来发现这样发送有一定概率丢失数据,当浏览器回收内存的时候这个请求是发不出去的，所以要将这个变量hold住：

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

我们为了防止缓存，经常会用毫秒的时间作为随机数（如：+new Date()），但是在极端情况下可能1ms就会发出两条log，这样第二条log就会丢失。

```
var _make_rnd  = function(){
    return (+new Date()) + '.r' + Math.floor(Math.random() * 1000);
  };
```

#### 360浏览器识别

为了能更好的统计这个浏览器，需要识别360SE、360EE。众所周知3Q大战之后360的UA就和chrome的一样，所以我们要用一些特别的方法识别它们。目前为止能够识别的方案，这个方案会随时更新，适应360的变化。

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
