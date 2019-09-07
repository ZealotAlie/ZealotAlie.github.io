---
title: "基于物理的大气渲染-单次散射的暴力求解"
tags: 图形学
      C#
      Unity
---

这篇文章是基于知乎上的一篇笔记[基于物理的大气渲染](https://zhuanlan.zhihu.com/p/36498679)。

因为目前正在学习渲染有关的知识，正好看到了就实现出来。

首先声明这篇文章的重点在于**加深对大气渲染方程的理解**，而不是要提出一种渲染大气的好方法，代码中使用的暴力算法也是不建议用于生产环境的。<!--more-->

## 渲染方程概述

详细的渲染方程的推导请看[[知乎]基于物理的大气渲染](https://zhuanlan.zhihu.com/p/36498679)，包括其中提到的参考资料。

本文主要实现地面上看天空时，所看到的的大气散射状况，所以可以着重看《2003 Modeling Skylight and Aerial Perspective》，另外Gems2中也有有关章节，我的代码中，估算光学深度的方程就来源于Gems2的源码。

直接从最终的单次散射方程入手

![1](/assets/images/2019-07-13-Atmosphere/1.jpg)
<br>*来源[Rendering] 基于物理的大气渲染-https://zhuanlan.zhihu.com/p/36498679*

画出示意图如下，

![2](/assets/images/2019-07-13-Atmosphere/2.jpg)

阳光从CP方向入射，首先到达P点并散射到PA方向，A为观察者的位置，我们看到的B的颜色的一部分，就是阳光经过CPA到达A时的光照值。

这里需要注意的是两个过程，其一是衰减，即从CP再到PA过程中的衰减。主要由散射造成，因为光在穿过分子时会被散射到各个方向，因此依然保持原方向前进的光的量就会减少。

另一个是P点时散射到A点的光的量，因为达到P点的光只有恰巧散射到PA方向才可能最终被观察者观测到。

回到散射方程，我们要在代码中计算散射方程的值，首先要确定哪些是常量。

对瑞利散射而言，β的值和波长的四次方成反比，因此对固定颜色的光，β可以认为是常量。ρ可以用指数函数来估算，

![3](/assets/images/2019-07-13-Atmosphere/3.jpg)

而瑞利散射的相位函数P也是已知的，所以问题的关键在于求解CPA路径的总光学距离（Optical Depth）。

我们可以通过相对光学距离来估算总光学距离，在《2003 Modeling Skylight and Aerial Perspective》和《Gems 2》的源码中都有类似的估测。

![4](/assets/images/2019-07-13-Atmosphere/4.jpg)
<br>*《2003 Modeling Skylight and Aerial Perspective》- relative optical length*

Gems2中的公式

```glsl
half OpticalDepthScale(half cos)
{
	half x = 1.0 - cos;
	return exp(-0.00287 + x * (0.459 + x * (3.83 + x * (-6.80 + x * 5.25))));
}
```

它实际上是某一点到任意方向的光学距离和该点到天顶方向的光学距离的比值

![5](/assets/images/2019-07-13-Atmosphere/5.jpg)
<br>*From-《2003 Modeling Skylight and Aerial Perspective》*

OK回到示意图

![6](/assets/images/2019-07-13-Atmosphere/6.jpg)

通过BA和Z0A的角度，我们可以估算出相对光学距离，通过CP和Z1P的角度，可以估算CP的相对光学距离，同理也可以估算BP。

下一步是求解Z0A和Z1P的光学距离。回到光学距离的定义公式，

![7](/assets/images/2019-07-13-Atmosphere/7.jpg)

前文已经给出了ρ和高度关系的估算公式，所以我简单对exp(-h/H)求积分得出天顶光学距离和高度的关系。

## 代码实现

完整的代码我已经上传到[GitHub-Atmosphere](https://github.com/ZealotAlie/Atmosphere)，所以不详细介绍，有兴趣的可以直接下载源码。

实现上来说，比较麻烦的是参数设定。我并没有完全按照物理量来计算β的值，而是直接用(1/λ)^4乘以一个系数来代替，所以为了避免调参的麻烦，我直接把散射方程的函数图绘制出来，

![8](/assets/images/2019-07-13-Atmosphere/8.jpg)

顺便推荐一下这个非常好用的在线绘图软件[Desmos](https://www.desmos.com/)。这个曲线代表当阳光与天顶方向的角度为x时，直接从观察者到太阳的那条射线上的光照值。假设夕阳时可见的阳光角度正好是90度，那此时看到的天空应该是偏红的，所以我将参数调节到大致红光将要超过蓝光和绿光的状态。

![13](/assets/images/2019-07-13-Atmosphere/13.jpg) ![9](/assets/images/2019-07-13-Atmosphere/9.jpg) ![14](/assets/images/2019-07-13-Atmosphere/14.jpg)

上面三张图分别是正午时的天顶，傍晚时的地平线和正午时的地平线。

可以很明显注意到的是，正午时的地平线看起来光非常不足。主要问题在于我们只计算了一次散射而没考虑多次散射，所以导致天边是黑乎乎一片。

另外由于地球本身是会遮挡光线的，因此在计算散射时要舍去不可能照射到的光线。

![10](/assets/images/2019-07-13-Atmosphere/10.jpg)

在图中α角之外的光线都是无法照射进来的，在代码中通过比较地平线与PA的余弦值以及光线和PA的余弦值来计算。为了避免夜晚时一片漆黑，光线被遮挡时函数返回了0.1而不是0。

```glsl
inline half GetAccessibleSunLightFactor(half horizonLineCos, half sunCos)
{
	return sunCos > horizonLineCos ? 1 : 0.1;
}
```

## 总结

最后我们加上米氏散射。米氏散射和瑞利散射类似，区别在于米氏散射的β值与波长无关，原因在于产生米氏散射和瑞利散射的粒子的物理尺度不同。造成瑞利散射的主要是空气中的气体分子，而造成米氏散射的是大气中的水汽，悬浮微粒等大颗粒，它们的大小比起光的波长来说要大得多，因此不同颜色的光在这些大颗粒中的散射几乎是一致的。

加上米氏散射之后，程序运行的效果如下

![11](/assets/images/2019-07-13-Atmosphere/11.jpg)
![12](/assets/images/2019-07-13-Atmosphere/12.jpg)

整体上比起之前要更加真实。

实际上，基于物理的天空的渲染是非常复杂的一门课题。除了多级散射之外，不同环境不同温度不同时间不同经纬度下，大气的密度和组成都是不同的，另外还要考虑云层，地面反射光线，甚至是地面上的人造光源的影响。

本文实现的程序充其量只能称之为演示程序，如果要真正实现可以在游戏中使用的大气渲染，还是要认真研读2008年的那篇论文。但通过阅读和实现程序能够加深我们对散射方程的理解，在这方面还是非常有意义的。

由于最近工作比较忙，所以暂时不会对这个课题有更深入的研究。另外由于个人的技术和知识都十分有限，在渲染方面也完全是一个新人，所以有错误的地方希望能指正。

## 参考资料

- [知乎 - Lele Feng - 基于物理的大气渲染](https://zhuanlan.zhihu.com/p/36498679)

- 《2003 Modeling Skylight and Aerial Perspective》

- 《GPU Gems2》

## 项目源码
- https://github.com/ZealotAlie/Atmosphere