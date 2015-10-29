---
title: Are u ok？---记一次H5项目的安卓适配
categories: 
  - 无线开发
tags:
  - H5
  - 适配
  - 安卓
  - 无线
date: 2015-10-28 06:18:10
author: 愈之
published: true
---

![Are u ok？---记一次H5项目的安卓适配](http://gtms02.alicdn.com/tps/i2/TB15IMmKXXXXXXzXpXX2AXZ8pXX-900-500.png)

### Are u ok ?

当雷布斯难以和台下的印度粉丝语言沟通的时候，他不由自主地向台下的米粉呐喊Are u ok 抒发他不能讲中文却无比激动的情绪。

当我看到那台老款安卓机上的H5页面的时候，我也想对它说 Are u ok? 

我不是要卖给印度人手机，而是因为在这些低版本的安卓机面前我也常常词穷技穷。

### 问题背景

无线化风生水起呀，最近做的无线项目里有几个H5的网页，在大部分手机上表现得很正常，但是适配测试阶段测试提的一个问题：在中兴U930上，页面整体布局错乱。

眼看离提交手淘测试时间点越来越近了，PDPM测试问我能不能提交手淘啊，我只能说遗憾地说，"有一款手机上适配有问题，还要看看。"

因为这个适配问题让我很难办，不能审查DOM，定位只能靠猜和试。我说还要看看，真的是只能看看。

我问顺堂MDS(淘宝的无线调试平台)能不能调，顺堂居然羞涩地低下了头：安卓4.4以下的确实不行呢。因为devtool....（广告费你懂的）

PC端有坏孩子IE，到了移动端还有安卓系统碎片化的问题。世界太不和谐了。

### 有图有真相

一般情况下，H5页面显得很好很正常
![图1](http://img2.tbcdn.cn/L1/461/1/9f4117125688c27b1a06ecdafe8019f37ac6ceb4)

而在中兴U930上是这样的。

![图2](http://img3.tbcdn.cn/L1/461/1/7aa911ea32a8256f2bd91b4be7f5f01cc6b9058f)

丑的我都不好意思贴出来

### 问题分析

问题还是得解决的，没办法，只好从整个无线工程渲染和适配的过程开始想一遍了。

首先判断是浮动布局造成的影响，毕竟浮动布局容易引起的问题，于是将同行的元素全都改成成inline-block，重新布局后满心期待push，一看，问题依旧，该怎样还是怎样。

窗外天气不是很好，有点雾霾，要不把你这手机扔出去算了。-。-

从截图上看，原因可能是某个元素占据了整行，也有可能是文档流节点总宽度超出了容器宽度。

既然其他机型上宽度都是正常的，为什么唯独这款的宽度不正常呢？这个问题始终绕不过。因为到目前为止，始终还在样式布层面考虑。既然还是无法定位，说明仅仅考虑CSS可能无法找到问题根源或是线索了。

应该从现在项目依赖的平台和框架整个过一遍页面的适配和渲染过程。

### 设备兼容

我们的移动设备分辨率各式各样，视觉稿却是一个固定的大小。无线应用需要铺满整个屏幕，也就是自适应，和PC的不一样。那么我们的如何做到适应这么多分辨率呢？

还好有kimi(淘宝前端团队针对移动端H5开发提供的整体解决方案)。

无线kimi项目依赖于m-base, m-base通过设定根元素的rem，即通过HTML根元素相对长度单位来定义一个相对的基准单位，为什么说是相对的呢？因为这个基准单位的值在不同分辨率下是不同的。所以可能从m-base上能找到线索。

那么m-base的这个HTML根元素的font-size到底是如何计算出来的呢。

m-base从1.0.0开始，前端对rem的基准处理统一约定为：viewport/10，也等价于vw单位，后续的rem规范基准都以屏宽十分之一为准，和手淘首页新标准、天猫均保持一致。

也就是说，我们规定了（篇幅有限，这里只贴出部分规范）
1rem = viewport / 10;
750的设计稿，1rem = 75px; 1px = 1/75 rem; 200px = 200 / 75 = 2.666666667rem;
640的设计稿，1rem = 64px; 1px = 1/64 rem; 200px = 200 / 64 = 3.125rem;



### 这个标题叫什么好呢

分析到这就有点眉目了，可能是rem基准单位值设置大了;也可能是这设备对rem的计算不准确。第二点不在我们控制范围内。所以只考虑第一点：先查一下这个机型现在的1rem值和viewport宽度值，这个设备上只有祭出alert大法了。

** 打印meta 信息，打印viewport, document, device宽度信息**

```
    var metas = window.parent.document.getElementsByTagName("meta")
    for (i = 0; i < metas.length; i++) {
        alert(metas[i].getAttribute("name"))
        alert(metas[i].getAttribute("content"))
    }

    alert(/ZTE U930_TD/.test(navigator.userAgent));
    alert(document.documentElement.clientWidth);
    alert(document.body.clientWidth);
    alert(window.innerWidth);
    alert(window.pageXOffset);
    alert(window.outerWidth);
    alert(screen.width);
    alert(window.rem);
```


**打印结果**

| key | value |
| --- | --- |
| document.documentElement.clientWidth | 360 |
| document.body.clientWidth | 360  |
|window.innerWidth | 360 |
| window.pageXOffset | 360 |
| window.outerWidth | 540 |
| screen.width | 540 |
| window.rem | 40.67 |



可以看到viewport宽度是360px，与该机型屏幕分辨率960x540相差了很大，这不科学啊~
[该机型的屏幕参数见这里](http://detail.zol.com.cn/326/325040/param.shtml)

可以看到1rem=40.67px，这和预计的54px也不一样，而且也不是36px（viewport的1/10）。

奇怪，难道是该机型有推出过增强版或是乞丐版？而我手上的是乞丐版？于是去工信部网站查该设备。[工信部链接在这里](http://www.tenaa.com.cn/WSFW/LicenceShow.aspx?code=jpokusu3tlOFxUU4Lw2sNhOXuAZbYVdDfASF7DKgVYi6efm5qzmAAbUin3DJS6Mc)
似乎没有其他版本，但是意外看到了设备证书到期时间---本月10号就要到期了，哦呵呵。


### 定位问题

宽度我们改变不了，那么从基准rem着手。
为什么rem=40.67，按照设备信息算应该是54，按照实际的viewport算应该是36。

想不通，看m-base源码是怎么算的吧。
搜索git代码库找到m-base，运气不错，很快就搜到了相关代码

![图3](http://img2.tbcdn.cn/L1/461/1/5e1aeab961e13b61d4ccf56a09ca46295794bab4)

可以看到m-base对该机型做了特殊处理
40.678/1.13正好就是36，看来确实只有360px。

既然viewport就是360px，那么把rem设置为36px应该是正确的呀。
于是在项目代码里hack一段

```
window.rem = document.documentElement.clientWidth / 10;
var fontEl = document.createElement('style');
fontEl.innerHTML = 'html: {font-size:' + window.rem + 'px !important}';
document.documentElement.firstElementChild.appendChild(fontEl);
```

满心期待push后一看，没啥变化。打印window.rem，还是40.67。
应该是因为很多脚本在执行，无法保证在这一段代码在m-base后执行来获得更高的样式优先级，异步之殇啊。


那就粗暴一点直接设置标签属性
```
window.rem = document.documentElement.clientWidth / 10;
$('html').attr('style', 'font-size: ' + window.rem + 'px !important');
```


刷新页面果然生效了
![图4](http://img2.tbcdn.cn/L1/461/1/f07ed845db192f0f10e69dd1348c2d2fcc507c3f)

可以看到右边有一大片空白。
看来设备的真实viewport值大于360小于406.7，而rem大于36小于40.67。

到这里应该就算完全定位问题了。窗外雾霾似乎少了点，这手机也算捡回了条命。


### 解决问题

接下来就是是体力活了：采用折半搜索确定rem的准确值，算一次改一次刷新一次，尝试38.335太小39.5太小...试到大概40.06看上去差不多了

效果如图
![screenshot](http://img3.tbcdn.cn/L1/461/1/1747b910882969e04c7cb75d818d8433d4b4f67e)

已经很接近屏幕边缘了，问题修复到这其实可以算是fixed了

屏幕右边还是有一丝丝白边，对处女座不够友好。

但饭点都过了，手指刷的都不听话了。


可是我还想算的更精确。

> 只要是重复的事情，肯定可以被程序取代。
                                                                             --- 我

所以跑个任务吧

流程很简单

![流程图](http://img1.tbcdn.cn/L1/461/1/e1eb474fa4ffe0c59f69ab2449c4a33bfc7d8e19)

```
    var adaptTheDevice = function (startRem) {
        window.rem = startRem || window.rem;
        var _ss = $('#xiangsi-dianpu .sim-shop').first().find('.itemsinshop a');
        var flag = true;
        var stepping = 0.0001;
        var time_in = new Date().getTime();
        while (flag) {
            for (var i = 1; i < _ss.length; ++i) {
                if (_ss[i].offsetTop < 1) {
                    flag = false;
                    alert('error')
                }
                if (_ss[i - 1].offsetTop != _ss[i].offsetTop) {
                    alert(window.rem - stepping);
                    flag = false;
                }
            }
            console.log(window.rem);
            window.rem = window.rem + stepping;
            $('html').attr('style', 'font-size: ' + window.rem + 'px !important');
        }
        alert("used time: " + (new Date().getTime() - time_in) + " ms");
    };
```

**结果**

![screenshot](http://img2.tbcdn.cn/L1/461/1/6b948eedcc320cdc6751d2cc80d56e35f605e583)

![screenshot](http://img4.tbcdn.cn/L1/461/1/60552e09f2786f09ae51a4382f15904dfbb796e8)


好吧，只用了20s就跑下来了，适配的值定为40.076，以后再有丧心病狂的安卓4.0来袭就不用重复踩这个坑了，重跑一下这个任务就行了。

最后给m-base提个issue。



### 后记

解决这个问题一共花了等效时间约2人日，实际时间大约要一周。
据统计2014年每天都有约三部智能机上市，如果适配的单位是机器的话，现在市面上近共约四千部智能机每个型号都能测一遍吗？做得到吗？

更合理的适配单位应当是操作系统版本----以操作系统版本为适配标准的量尺，同时兼顾个别经典机型。

毕竟雷布斯有勇气讲英文了，我们为什么不向前走呢？









