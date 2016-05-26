---
layout:     post
title:      "高仿朋友圈照片查看器（二）"
subtitle:   "用PhotoView实现手势缩放图片"
date:       2016-05-26
author:     "Woody"
catalog: true
tags:
    - Android
    - UI
    - 微信
---

## 简介

PhotoView是Chris Banes大神写的开源控件，[Github地址](https://github.com/chrisbanes/PhotoView),本文将介绍如何用PhotoView实现朋友圈手势缩放效果

## PhotoView解析

### Features
 
- 支持Pinch手势自由缩放
- 支持双击放大/还原
- 支持平滑滚动
- 在滑动父控件下能够运行良好。（例如：ViewPager）
- 支持基于 Matrix 变化（放大/缩小/移动）的事件监听

### 优势

- PhotoView 是 ImageView 的子类，支持所有 ImageView 的源生行为
- 任意项目可以非常方便的从 ImageView 升级到 PhotoView，不用做任何额外的修改
- 可以非常方便的与 ImageLoader/Picasso 之类的异步网络图片读取库集成使用
- 事件分发做了很好的处理，可以方便的与 ViewPager 等同样支持滑动手势的控件集成

### 总体思路

PhotoView 这个项目关键点其实就是 Touch 事件处理和 Matrix 图形变换的应用

Touch事件传递流程如下图，从架构上看，干净利落的将事件层层分离，交由不同的 Detector 处理，最后再将处理结果回调给 PhtotViewAttacher 中的 Matrix 去实现图形变换效果。以上部分摘抄自[PhotoView 源码解析](http://p.codekk.com/blogs/detail/54cfab086c4761e5001b253a)

![](http://7xtfm0.com1.z0.glb.clouddn.com/flow.png)

## 修改

PhotoView默认支持两级双击放大，我们的需求是双击放大，再双击还原。找到 `DefaultOnDoubleTapListener.java`

修改 `onDoubleTap()`方法

```java
 public boolean onDoubleTap(MotionEvent ev) {
        if (photoViewAttacher == null)
            return false;

        try {
            float scale = photoViewAttacher.getScale();
            float x = ev.getX();
            float y = ev.getY();

            if (scale < photoViewAttacher.getMediumScale()) {
                photoViewAttacher.setScale(photoViewAttacher.getMediumScale(), x, y, true);
            } else if (scale >= photoViewAttacher.getMediumScale() && scale < photoViewAttacher.getMaximumScale()) {
                photoViewAttacher.setScale(photoViewAttacher.getMaximumScale(), x, y, true);
            } else {
                photoViewAttacher.setScale(photoViewAttacher.getMinimumScale(), x, y, true);
            }
        } catch (ArrayIndexOutOfBoundsException e) {
            // Can sometimes happen when getX() and getY() is called
        }

        return true;
    }
```

修改为


```java 
public boolean onDoubleTap(MotionEvent ev) {
        if (photoViewAttacher == null)
            return false;

        try {
            float scale = photoViewAttacher.getScale();
            float x = ev.getX();
            float y = ev.getY();

            if(scale < photoViewAttacher.getMediumScale()){
				photoViewAttacher.setScale(photoViewAttacher.getMediumScale(),x,y,true);
			}else {
				photoViewAttacher.setScale(1.0f,x,y,true);
			}
        } catch (ArrayIndexOutOfBoundsException e) {
            // Can sometimes happen when getX() and getY() is called
        }

        return true;
    }
```


PhotoView默认缩放最小值为1.0f,通过 `setMinimumScale(float minimumScale) `方法设置最小缩放比例。朋友圈在缩放到最小时松手会还原到默认大小，修改 `PhotoViewAttacher.java` 找到`onTouch()`方法，修改 ACTION_UP对应的处理

```java 
  case ACTION_UP:
                    // If the user has zoomed less than min scale, zoom back
                    // to min scale
                    if (getScale() < 1.0f) {
                        RectF rect = getDisplayRect();
                        if (null != rect) {
                            v.post(new AnimatedZoomRunnable(getScale(), 1.0f,
                                    rect.centerX(), rect.centerY()));
                            handled = true;
                        }
                    }
         break;
```

## 其他

朋友圈的照片查看使用Gallery实现的，此例采用ViewPager实现，根据PhotoView主页介绍，在使用带滑动属性的ViewGroup时，需要在`onInterceptTouchEvent`中catch住IllegalArgumentException异常，如下

```java
/**
 * Hacky fix for Issue #4 and
 * http://code.google.com/p/android/issues/detail?id=18990
 * <p/>
 * ScaleGestureDetector seems to mess up the touch events, which means that
 * ViewGroups which make use of onInterceptTouchEvent throw a lot of
 * IllegalArgumentException: pointerIndex out of range.
 * <p/>
 * There's not much I can do in my code for now, but we can mask the result by
 * just catching the problem and ignoring it.
 *
 * @author Chris Banes
 */
public class PhotoViewPager extends MyViewPager {
	public PhotoViewPager(Context context, AttributeSet attrs) {
		super(context, attrs);
	}

	public PhotoViewPager(Context context) {
		super(context);
	}

	@Override
	public boolean onInterceptTouchEvent(MotionEvent ev) {
		try {
			return super.onInterceptTouchEvent(ev);
		} catch (IllegalArgumentException e) {
			e.printStackTrace();
			return false;
		}
	}
}
```

单击大图关闭则需要调用PhotoView的`setOnViewTapListener`来监听单击图片事件，并做处理。


[代码地址](https://github.com/nirvanawoody/WeixinPhotoViewer)