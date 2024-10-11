
白板软件书写速度是其最核心的功能，注册StylusPlugin从触摸线程拿触摸点数据并在另一UI线程绘制渲染是比较稳妥的方案，具体的可以查看小伙伴德熙的[2019\-1\-28\-WPF\-高性能笔 \- lindexi \- 博客园 (cnblogs.com)](https://github.com)


上面StylusPlugin方案能提升在大屏目前如富创通、华欣触摸框的主要产品版本上，1帧16ms左右的书写性能。除了这个跳过一些流程来减少延时，我们还能继续优化书写性能么？答案肯定是可以的


本文我们介绍下书写加速的一类实现方向，通过预测下一个甚至N个点，提前绘制笔迹来降低书写延迟。


### 曲线拟合预测


书写预测，这里介绍下曲线拟合的方案：


取N个点拟合成一条曲线，算出它的曲线公式，然后下一个点可以输入它的X位置得到Y \-\- 以X方法或者Y方法为基准，拟合出以X为参数的曲线。


这里采用的是开源组件MathNet.Numerics，也可以使用其它的拟合曲线方案，目标就是先输出一条曲线公式。先安装其Nuget包MathNet.Numerics：




```
"MathNet.Numerics" Version="5.0.0" />
```


引用using MathNet.Numerics;我们以X坐标为基准预测Y坐标值，已经写好函数：




```
 1     private static Point[] PredictPoints(Point[] pointArray, int degree, int predictCount)
 2     {
 3         Debug.WriteLine("输入：" + string.Join(",", pointArray.Select(i => $"({i.X},{i.Y})")));
 4         var xList = pointArray.Select(i => i.X).ToArray();
 5         var yList = pointArray.Select(i => i.Y).ToArray();
 6         var lastX = xList[xList.Length - 1];
 7         var lastX1 = xList[xList.Length - 2];
 8         var lastPointLength = lastX - lastX1;
 9         double[] parameters = Fit.Polynomial(xList, yList, degree);
10         var predictPoints = new Point[predictCount];
11         for (int i = 0; i < predictCount; i++)
12         {
13             var currentX = lastX + (i + 1) * lastPointLength;
14             var currentY = 0d;
15             for (int j = 0; j < degree + 1; j++)
16             {
17                 var parameterJ = parameters[j];
18                 for (int k = 0; k < j; k++)
19                 {
20                     parameterJ *= currentX;
21                 }
22 
23                 currentY += parameterJ;
24             }
25 
26             var newPoint = new Point(currentX, currentY);
27             predictPoints[i] = newPoint;
28         }
29         Debug.WriteLine("输出：" + string.Join(",", predictPoints.Select(i => $"({i.X},{i.Y})")));
30         return predictPoints;
31     }
```


这里的double\[] parameters \= Fit.Polynomial(xList, yList, degree)，表示通过X以及Y系列数据，以阶数degree（如二阶曲线）计算出当前多项式参数值parameters。


如果degree是2阶，可以计算得到y值：


y \= parameters\[0] \+ parameters\[1]\*x \+ parameters\[2]\*x^2


好，曲线公式有了，那下面就是塞x坐标得到y值，也就是point。


如果是Y轴向上递增的二队曲线，如下图，从左到右绿色点是已知点列表，黄色为预测的点。这里我们依次预测4个点：


![](https://img2024.cnblogs.com/blog/685541/202410/685541-20241008174617808-1657555136.png)


这类场景，预测结果是比较正常的。


我们再看看抛物线的场景，沿X方向Y坐标值依次递减，即顺时针角度值增加，预测点如下：


![](https://img2024.cnblogs.com/blog/685541/202410/685541-20241008174744664-1718705877.png)


从上图可以看出，Y方向值递减时预测结果是抛物线的另一半，Y轴值另一侧反向加速递减。但我们的期望肯定不是像这类抛物线，我们期望它能延着绿色轨迹向斜上方走。


为何它会快速弹回来呢？原因就是拟合的2阶曲线，公式如此。


这类方法，只能做到一维的预测，即遇到X、Y方向值不变或者值变化较少时，曲线变化就会像抛物线一样到达它的顶端后，就会快速弹回来。


以拟合成曲线公式来计算，会有一定缺陷。这个我们也是能优化的，上方的函数里我们是以X轴为基准，得到的公式中X为变化量，下面换一下以Y轴为基准：




```
 1     public static Point[] PredictPointsY(Point[] pointArray, int degree, int predictCount)
 2     {
 3         Debug.WriteLine("输入：" + string.Join(",", pointArray.Select(i => $"({i.X},{i.Y})")));
 4         var xList = pointArray.Select(i => i.X).ToArray();
 5         var yList = pointArray.Select(i => i.Y).ToArray();
 6         var lastY = yList[yList.Length - 1];
 7         var lastY1 = yList[yList.Length - 2];
 8         var lastPointLength = lastY - lastY1;
 9         double[] parameters = Fit.Polynomial(yList, xList, degree);
10         var predictPoints = new Point[predictCount];
11         for (int i = 0; i < predictCount; i++)
12         {
13             var currentY = lastY + (i + 1) * lastPointLength;
14             var currentX = 0d;
15             for (int j = 0; j < degree + 1; j++)
16             {
17                 var parameterJ = parameters[j];
18                 for (int k = 0; k < j; k++)
19                 {
20                     parameterJ *= currentY;
21                 }
22 
23                 currentX += parameterJ;
24             }
25 
26             var newPoint = new Point(currentX, currentY);
27             predictPoints[i] = newPoint;
28         }
29         Debug.WriteLine("输出：" + string.Join(",", predictPoints.Select(i => $"({i.X},{i.Y})")));
30         return predictPoints;
31     }
```


这里的二阶拟合曲线就变成了：


x \= parameters\[0] \+ parameters\[1]\*y \+ parameters\[2]\*y^2


同样预测4个点，看下结果：


![](https://img2024.cnblogs.com/blog/685541/202410/685541-20241008180515433-2107338806.png)


这个预测趋势就符合我们的期望了。


所以上方X方向以及Y方向分别为基准，拟合二队曲线，然后输出预测点，我们综合取一个比较适合的点即可。


如何取呢？可以看到我们期望的点是变化趋势变化小的一个。即以最后俩个数据点的角度A为基准，预测点与最后数据点的向量角度B1与B2，顺时针角度变化较小的点是我们期望输出的。


![](https://img2024.cnblogs.com/blog/685541/202410/685541-20241008182318051-214889763.png)


### 曲线拟合其它问题\-预测失准


上面我们解决了坐标点场景，单方向拟合时输出点快速弹回的问题。在验证书写时，我们还发现一类预测失准问题：


![](https://img2024.cnblogs.com/blog/685541/202410/685541-20241008182951931-2107358779.png)


如上图，黄色点往上偏了（黄色点是直线，并且角度不符合原有趋势），真实期望我们是想要沿着原有角度减少的趋势，预测点为角度略偏下的方向：


![](https://img2024.cnblogs.com/blog/685541/202410/685541-20241008183143178-1143857242.png)


这里预测失准的原因是，X方向值无法正常预测。因为按我们上面曲线拟合的方案，这类抛物线场景是以Y轴为基准，输入Y得到X方向值，但按曲线变化的方向输入一个最后俩点之间Y轴变化量，预测点的X值应该是接近无限大的，超出了曲线范围。


这里可以根据曲线点角度变化量来处理，看上图点与点之间角度是按顺时针依次增加的，所以预测出来的点也应该要继续顺时针增加角度。所以可以将输出的点按最后俩点Point(n)、Point(n\-1\)之间向量角度值，或者再增加最后三点之间角度的差值angleChange。


我使用的是增加角度变化量，如下图输出效果：


![](https://img2024.cnblogs.com/blog/685541/202410/685541-20241008190348385-1698495792.png)


### 曲线拟合其它问题\-预测位置超出


我们做的毕竟是预测，预测点肯定不能完美代替真实书写触摸点。尤其是当我们按照设置下一个预测点太远，很大概率是会偏离原有曲线的


在上面PredictPoints预测函数内，使用了var lastPointLength \= lastX \- lastX1;来作为下一个预测点X方向的位置。但这其实是不符合实际情况的，因为你并不清楚下一个预测点也变化了 lastX \- lastX1的X方向距离，如果强行用此X变化量确定预测点，预测点偏离曲线的概率会很大。那如何解决呢？


我的解决方法是，通过上面PredictPoints计算出下一预测点的角度变化量changedAnge即可，然后再以lastPoint\-lastPoint1之间的距离作为半径围绕lastPoint进行旋转180\+changedAnge至一个新的点。这个新点作为最后预测点


![](https://img2024.cnblogs.com/blog/685541/202410/685541-20241010205608950-1228100165.png)


为何以lastPoint\-lastPoint1之间的距离作为半径？因为在快速书写过程中，触摸框帧率是固定的，书写速度不怎么变化的情况下，点与点之间的距离很接近，所以我们只要预测出下一点的changedAnge就行了。




```
1     var angleChanged = Math.Min(lastChangedAngle, Math.Max(angle1Changed, angle2Changed));
2     var rotateAngle = (180 + angleChanged).GetPositiveAngle();
3     var predictPoint = lastPoint1.Rotate(lastPoint, rotateAngle);
```


最后，我们按上面的方案验证下真实书写预测的效果。输出书写痕迹，我们换个颜色，蓝色为真实点、红色为预测点。


预测1个点，红色与真实书写曲线重合：


![](https://img2024.cnblogs.com/blog/685541/202410/685541-20241008190139921-97817669.png)


预测2个点，红色是第一个预测点，粉色是第二个预测点：


![](https://img2024.cnblogs.com/blog/685541/202410/685541-20241008190806329-410132531.png)


这里预测效果是在我的开发触摸屏设备（Dell P2418HT，1080p）上验证的，看上面效果粉色点偏移的较多。在这台触摸屏上，不建议预测2个点


需要注意的是，书写预测与屏幕报点率(帧率)强相关。一般情况下，输出俩点之间间隔时间越长，俩点之间间距越大，预测点的误差也会变大，但相应的预测距离变远了即书写延迟会降低很大。


我这Dell触摸屏触摸移动时，输入平均间隔33\.3ms，一次输入包含2\-3个点，点平均间隔在16\.6ms。俩点平均间隔16\.6ms，说明触摸框是60fps报点。另外，详细的触摸框报点率数据以及开启WM\_POINTER消息提升应用层的触摸输入帧率可以看下：[白板书写延迟\-触摸屏报点率 \- 唐宋元明清2188 \- 博客园 (cnblogs.com)](https://github.com):[wgetCloud机场](https://tabijibiyori.org)


根据上面触摸报点数据，上面书写预测方案预测1个点，在这台Dell触摸屏上书写延迟可以降低16\.67ms。


也在140帧高报点率的大屏触摸框上试了，可以稳定预测2个点，即可降低延迟2\*7\=14ms左右：


![](https://img2024.cnblogs.com/blog/685541/202410/685541-20241011104938199-2005555920.png)


 


 关键词：白板书写加速、高精度书写预测、触摸屏书写性能


