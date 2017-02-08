# iframe引入子页面与主页面的viewport设置冲突问题探讨

------

为了兼容retina屏，前端开发往往会采用根据dpr(设备像素比device pixel ratio),来动态设置viewport scale值的兼容方式。具体方式可以参见[移动端，多屏幕尺寸高清屏retina屏适配的解决方案](http://www.cnblogs.com/cench/p/5314044.html)

这种兼容方案，在单独使用的时候，是一种相对完美的方案。但是如果有一个A页面，在A页面中通过iframe引入B页面，而A，B两个页面的viewport scale值设置不同，会出现冲突问题，导致页面被缩放。我在chrome的mobile浏览模拟器中试验，得到的结果是:
> * 如果主页面设置了viewport scale，无论子页面是否设置了viewport，主页面和子页面的viewport都会以主页面的viewport scale设置为准；
> * 如果主页面没有设置，而子页面设置了，则主页面的viewport也会自动变为子页面的viewport设置。


推荐几种处理方式:

1, 最优的:主页面与子页面的viewport缩放策略保持一致。(废话...)

2, 若已知主页面和子页面的的viewport设置方式，可以通过css的transform scale来缩放子页面。实例代码如下：
```
//html
<div class="spaceWrap">
  <iframe class="viewPort" src="http://www.apple.com">
  </iframe>
</div>

//css
.spaceWrap { 
  width: 400px; 
  height: 225px; 
  padding: 0; 
  overflow: hidden; 
}

.viewPort { 
  width: 1600px; 
  height: 900px; 
  border: 1px solid black;
}

.viewPort {
    -webkit-transform: scale(0.25);
    -webkit-transform-origin: 0 0;
}

```
但这种方式，我发现在iphone 6上有时会出现闪退...

3,若完全不知道子页面的viewport的缩放策略，且不同子页面的缩放策略还千差万别
	3.1 若主页面和子页面在同一个域中:
		通过js抓取到iframe中中viewport的缩放比，然后反向的设置主页面的缩放比

	3.2 若主页面和子页面不在同一个域中:
		可以通过前后端结合的方式，服务端抓取子页面，放到自己的域中，然后参照3.1的方法。（抓取到的页面不能直接用来展示，因为base url的改变，使得很多请求都会失败。）
这种方式比较麻烦，且会严重拖慢页面展现的时间。

