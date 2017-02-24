---
layout:     post
title:      "Android Choreographer和VSYNC机制"
date:       2017-02-24
author:     "Woody"
catalog: true
tags:
    - Android
    - 源码分析
---

## 背景介绍

Android系统的UI流畅度一直被人诟病，Google团队在4.1（Jelly Bean）对UI流畅性进行改进，通过Project Butter对Android Display系统进行了重构，引入了3个核心元素，即VSYNC、Triple Buffer和Choreographer

## VSYNC机制

VSYNC是Vertical Synchronization（垂直同步）的缩写，可以把它简单理解为一种定时中断。在开始之前，我们需要了解两个基本的概念

1. Refresh Rate（刷新率）代表硬件设备，例如手机屏幕，显示器在一秒内刷新屏幕的次数，这取决于硬件的固定参数，例如60HZ
2. Frame Rate（帧速率）代表GPU在一秒内绘制操作的帧数，单位fps

GPU获取图形数据进行绘制，硬件将绘制好的数据呈现在屏幕上，为了使画面在屏幕上流畅的显示，需要保持Frame Rate在60以上，也就是一次绘制操作的时间不超过16ms，Frame Rate与Refrash Rate不会总保持一致，如果GPU绘制速度大于屏幕刷新速度，就会出现截断现象，这是因为GPU使用一块内存绘制帧数据，新的图像帧会从上到下一行一行覆盖旧图像帧，当屏幕刷新时并不知道缓冲区现在的数据状态，这样就抓取了GPU中还没有绘制完的一帧数据，如下图。

![](http://7xtfm0.com1.z0.glb.clouddn.com/20150411195154316)

解决方法是双缓冲技术（double buffering），GPU首先将帧数据写到back buffer中，然后把它拷贝到另一块区域frame buffer。当绘制下一帧时，数据先写到back buffer，屏幕刷新时，只从frame buffer中获取。如果GPU正在写back buffer，这时候来了刷新屏幕，VSYNC就会阻止这个拷贝过程,实际上Android最多会使用到三重缓存，也就是Triple Buffer，后面会介绍。

如果Frame Rate < Refresh Rate则会造成丢帧和卡顿，因此我们要尽量保证每一帧的渲染时间在16ms之内

在没有VSYNC机制如下图所示

![](http://7xtfm0.com1.z0.glb.clouddn.com/1354956012_2098.png)

 - 时间从0开始，进入第一个16ms：Display显示第0帧，CPU处理完第一帧后，GPU紧接其后处理继续第一帧。三者互不干扰，一切正常。
 - 时间进入第二个16ms：因为早在上一个16ms时间内，第1帧已经由CPU，GPU处理完毕。故Display可以直接显示第1帧。显示没有问题。但在本16ms期间，CPU和GPU却并未及时去绘制第2帧数据（注意前面的空白区），而是在本周期快结束时，CPU/GPU才去处理第2帧数据。
- 时间进入第3个16ms，此时Display应该显示第2帧数据，但由于CPU和GPU还没有处理完第2帧数据，故Display只能继续显示第一帧的数据，结果使得第1帧多画了一次（对应时间段上标注了一个Jank）。
- 通过上述分析可知，此处发生Jank的关键问题在于，为何第1个16ms段内，CPU/GPU没有及时处理第2帧数据？原因很简单，CPU可能是在忙别的事情（比如某个应用通过sleep固定时间来实现动画的逐帧显示），不知道该到处理UI绘制的时间了。可CPU一旦想起来要去处理第2帧数据，时间又错过了！

为了解决这个问题，引入了VSYNC机制，如下图所示

![](http://7xtfm0.com1.z0.glb.clouddn.com/vsync.png)

每收到VSYNC中断，CPU就开始处理各帧数据。整个过程非常完美。
不过，仔细琢磨却会发现一个新问题： 上图中，CPU和GPU处理数据的速度似乎都能在16ms内完成，而且还有时间空余，也就是说，CPU/GPU的FPS（帧率，Frames Per Second）要高于Display的FPS。确实如此。由于CPU/GPU只在收到VSYNC时才开始数据处理，故它们的FPS被拉低到与Display的FPS相同。但这种处理并没有什么问题，因为Android设备的Display FPS一般是60，其对应的显示效果非常平滑。
如果CPU/GPU的FPS小于Display的FPS，会是什么情况呢？请看下图：

![](http://7xtfm0.com1.z0.glb.clouddn.com/jank.png)

在第二个16ms时间段，Display本应显示B帧，但却因为GPU还在处理B帧，导致A帧被重复显示。
同理，在第二个16ms时间段内，CPU无所事事，因为A Buffer被Display在使用。B Buffer被GPU在使用。注意，一旦过了VSYNC时间点，CPU就不能被触发以处理绘制工作了

为什么CPU不能在第二个16ms处开始绘制工作呢？原因就是只有两个Buffer。如果有第三个Buffer的存在，CPU就能直接使用它，而不至于空闲。出于这一思路就引出了Triple Buffer。如下图

![](http://7xtfm0.com1.z0.glb.clouddn.com/triple.png)

第二个16ms时间段，CPU使用C Buffer绘图。虽然还是会多显示A帧一次，但后续显示就比较顺畅了。
是不是Buffer越多越好呢？回答是否定的。由图4可知，在第二个时间段内，CPU绘制的第C帧数据要到第四个16ms才能显示，这比双Buffer情况多了16ms延迟。所以，Buffer最好还是两个，三个足矣。

### Choreographer

Android系统从4.1(API 16)开始加入Choreographer这个类来控制同步处理输入(Input)、动画(Animation)、绘制(Draw)三个UI操作。其实UI显示的时候每一帧要完成的事情只有这三种。Choreographer接收显示系统的时间脉冲(垂直同步信号-VSync信号)，在下一个frame渲染时控制执行这些操作。Choreographer中文翻译过来是"舞蹈指挥"，字面上的意思就是优雅地指挥以上三个UI操作一起跳一支舞。这个词可以概括这个类的工作，如果android系统是一场芭蕾舞，他就是Android UI显示这出精彩舞剧的编舞，指挥台上的演员们相互合作，精彩演出。







