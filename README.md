# echarts-changed-legend
对echarts压缩包进行修改，当legend过多时，可以进行翻页。 具体使用阅读文档http://blog.csdn.net/danhuan/article/details/72831245


**问题：**
图例可以跟地图有联动效果，用来当列表使用，与地图有联动效果，简直太棒了，但是echarts图例太多，实在太占用空间，遮挡图表，又无法移动legend层。当屏幕小，满屏幕都是图例呀。。。如下图，头疼。
![image.png](http://upload-images.jianshu.io/upload_images/6166110-89b85aa3930bbd75.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

翻阅echarts相关文档，百度，Q群等途径寻找解决方法，都没有得到想要答案。于是鼓起勇气尝试修改源码。
开始的想法是：右边一列，不换行显示，出现滚动条，可以向下滚动。后来研究了echarts源码，觉得在canvas上加滚动条有点困难，所以选择了 向下翻页 的方式。

##效果图：
![image.png](http://upload-images.jianshu.io/upload_images/6166110-16710f7fc4d69133.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

##原理
echarts创建legend层是先创建每一条图例，再计算位置，全部渲染出来。发现echarts里一段主要代码，就是这里实现了**"图例换行"**。
```javascript
// Wrap when width exceeds maxHeight or meet a `newline` group
if (nextY > maxHeight || child.newline) {
	x += currentLineMaxSize + gap;
	y = 0;
	nextY = moveY;
	currentLineMaxSize = rect.width;
}
else {
	currentLineMaxSize = Math.max(currentLineMaxSize, rect.width);
}
```
那么，是不是可以把 ”换行“ 变成 “换页”！ ^.^oh，good idea，get ! 于是这么干！

##步骤如下：
为了避免 改动 会引起其他问题。通过自定义新的字段参数控制。
主要需要改动以下几个地方：

##1.html页面中改动
我在初始化echarts的div标签下面添加`<div class="js-eclegend-tool" style="position: absolute;right: 20px;top: 40%"></div>`，用于存放上下翻页按钮，下面会用到。我写在echarts容器下面，css样式决定它的位置，代码如下：
```html
<div class="ibox">
        <!--初始化echarts实例的容器-->
	<div id="eBaiduMap" class="ibox-size" style="width: 100%;"></div>
        <!--用于存放上下翻页按钮.js-eclegend-tool容器，页面中的修改只需增加下面这一句-->
	<div class="js-eclegend-tool" style="position: absolute;right: 20px;top: 40%"></div>
</div>
```

##2.修改option配置
实例化echarts时，修改option配置，即添加`pagemode: true`,留意注释为`//注意`的地方
```javascript
//modify by danhuan
legend: {
    orient: 'vertical', //注意
    right:0,
    top: 0, //注意
    //bottom:0,
    //left:0,
    //width:200,
    pagemode: true, //注意,自定义的字段，开启图例分页模式，只有orient: 'vertical'时才有效
    height:"100%",
    data: legendData,
    itemHeight:18,
    itemWidth: 18,
    textStyle: {
        fontWeight: 'bolder',
        fontSize: 12,
        color:'#fff'
    },
    inactiveColor:'#aaa',
    //padding: [20, 15],
    backgroundColor: 'rgba(0, 0, 0, 0.7)',
    shadowColor: 'rgba(0, 0, 0, 0.5)',
    shadowBlur: 5,
    zlevel: 100
},
```

##3.修改echarts源码，总共两处修改
####第一处修改，在源码中找到以下代码：
![image.png](http://upload-images.jianshu.io/upload_images/6166110-9bdb45c875a8da57.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
为了你们方便复制查找，我把该截图的代码粘贴出来，即：
```javascript
layout: function (group, componentModel, api) {
	            var rect = layout.getLayoutRect(componentModel.getBoxLayoutParams(), {
	                width: api.getWidth(),
	                height: api.getHeight()
	            }, componentModel.get('padding'));
	            layout.box(
	                componentModel.get('orient'),
	                group,
	                componentModel.get('itemGap'),
	                rect.width,
	                rect.height
	            );

	            positionGroup(group, componentModel, api);
	        },
```

找到这几行代码，修改为以下代码：

```javascript
layout: function (group, componentModel, api) {
	var rect = layout.getLayoutRect(componentModel.getBoxLayoutParams(), {
		width: api.getWidth(),
		height: api.getHeight()
	}, componentModel.get('padding'));

	/*modify by danhuan 为了避免影响到其他模块，传参数判断修改 legend 是否要分页  s*/
	if(componentModel.get('pagemode')){//如果 legend 启用分页 modify by danhuan
		layout.box(
			componentModel.get('orient'),
			group,
			componentModel.get('itemGap'),
			rect.width,
			rect.height,
			componentModel.get('pagemode') //modify by danhuan
		);
	}
	if(!componentModel.get('pagemode')){
		layout.box(
			componentModel.get('orient'),
			group,
			componentModel.get('itemGap'),
			rect.width,
			rect.height
		);
	}
	/*modify by danhuan 传参数判断修改 legend 是否要分页  e*/

	positionGroup(group, componentModel, api);
},
```
这样做的目的是为了将步骤2的自定义参数 pagemode: true 传进echarts ,为了避免有其他方法调用layout方法导致其他图表错误，所以用了判断if(componentModel.get('pagemode')){...}，有参数 pagemode: true 时，才会进入改动的代码。

####第二处修改：
找到以下代码function boxLayout(orient, group, gap, maxWidth, maxHeight)...截图如下：
![image.png](http://upload-images.jianshu.io/upload_images/6166110-39de41d4fb81e064.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

将该函数修改为：
```javascript
function boxLayout(orient, group, gap, maxWidth, maxHeight,pagemode) {
	//console.log(group);
	var x = 0;
	var y = 0;
	if (maxWidth == null) {
		maxWidth = Infinity;
	}
	if (maxHeight == null) {
		maxHeight = Infinity;
	}
	var currentLineMaxSize = 0;
	group.eachChild(function (child, idx) {
		//console.log(child, idx);

		var position = child.position;
		var rect = child.getBoundingRect();
		var nextChild = group.childAt(idx + 1);
		var nextChildRect = nextChild && nextChild.getBoundingRect();
		var nextX;
		var nextY;
		if (orient === 'horizontal') {
			var moveX = rect.width + (nextChildRect ? (-nextChildRect.x + rect.x) : 0);
			nextX = x + moveX;
			// Wrap when width exceeds maxWidth or meet a `newline` group
			if (nextX > maxWidth || child.newline) {
				x = 0;
				nextX = moveX;
				y += currentLineMaxSize + gap;
				currentLineMaxSize = rect.height;
			}
			else {
				currentLineMaxSize = Math.max(currentLineMaxSize, rect.height);
			}
		}
		else {
			var moveY = rect.height + (nextChildRect ? (-nextChildRect.y + rect.y) : 0);

			nextY = y + moveY;

			/*by danhuan s*/
					if(pagemode){

						//console.log(pagemode);
						if (nextY > maxHeight || child.newline) {
							var html = '<div class="js-prePage"> <img style="width: 32px;height: 32px;cursor: no-drop" src="/mapout-web-visual/img/up-disable.png"  title="已经是第一页"/></div>';
							html +=	'<div class="js-nextPage"> <img  style="width: 32px;height: 32px;cursor: pointer" src="/mapout-web-visual/img/down-icon.png" title="下一页"/></div>';
							$('.js-eclegend-tool').html(html);
						}
						else {
							currentLineMaxSize = Math.max(currentLineMaxSize, rect.width);
						}
					}else{
						//console.log(pagemode);
						// Wrap when width exceeds maxHeight or meet a `newline` group
						if (nextY > maxHeight || child.newline) {
							x += currentLineMaxSize + gap;
							y = 0;
							nextY = moveY;
							currentLineMaxSize = rect.width;
						}
						else {
							currentLineMaxSize = Math.max(currentLineMaxSize, rect.width);
						}
					}
					/*by danhuan e*/
		}

		if (child.newline) {
			return;
		}

		position[0] = x;
		position[1] = y;

		orient === 'horizontal'
			? (x = nextX + gap)
			: (y = nextY + gap);
	});
}
```
修改的代码意思：在图例总高度大于legend容器高度，需要换行时, 不让其换行，而是一直向下画图例， 同时向 步骤1 建的容器$('.js-eclegend-tool')里面添加 ‘上一页’、‘下一页’ 按钮。

##4.添加按钮事件
 给”上一页“， ”下一页“按钮添加事件，通过echarts的setOption改变legend层的top，实现其翻页

```javascript
/*=====legend 的分页控制 事件=s===*/
        var PageEvent = function (i) {
            var percent = -i * 98 + '%';
            myChart.setOption({
                legend: {
                    top: percent
                }
            });
        };

        if (option.legend.pagemode) {
            $('body').on('click', '.js-prePage', function () {

                if (clickCount > 0) {
                    clickCount = clickCount - 1;
                    PageEvent(clickCount);
                    //console.log(clickCount+'上一页');
                    $('.js-prePage img').attr({'src': '/mapout-web-visual/img/up-icon.png', 'title': '上一页'});
                    $('.js-prePage img').css('cursor','pointer');
                    //$('.js-nextPage img').attr('src','/mapout-web-visual/img/down-icon.png');
                    if(clickCount==0){
                        $('.js-prePage img').attr({'src': '/mapout-web-visual/img/up-disable.png', 'title': '已经是第一页'});
                        $('.js-prePage img').css('cursor','no-drop');
                    }
                } else {
                    //console.log(clickCount+'已经是第一页');
                    $('.js-prePage img').attr({'src': '/mapout-web-visual/img/up-disable.png', 'title': '已经是第一页'});
                    $('.js-prePage img').css('cursor','no-drop');
                }
            });
            $('body').on('click', '.js-nextPage', function () {
                clickCount = clickCount + 1;
                //console.log(clickCount);
                PageEvent(clickCount);
                $('.js-prePage img').attr({'src': '/mapout-web-visual/img/up-icon.png', 'title': '上一页'});
                $('.js-prePage img').css('cursor','pointer');
            });
        }
        /*=====legend 的分页控制 事件=e===*/
```

按钮图标：
![up-icon.png](http://upload-images.jianshu.io/upload_images/6166110-0bb59947cac4598e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![up-disable.png](http://upload-images.jianshu.io/upload_images/6166110-63a4f16ad0f910db.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![down-icon.png](http://upload-images.jianshu.io/upload_images/6166110-487ce05977840180.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![down-disable.png](http://upload-images.jianshu.io/upload_images/6166110-d4d27181c3b4debf.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

以上仅个人想法，请多多指教！希望能得到您更多宝贵的想法。

本文本人简书地址：http://www.jianshu.com/p/869853a7c8ce
