---
layout:     post
title:      "高仿朋友圈照片查看器（三）"
subtitle:   "ViewPager支持阻尼回弹"
date:       2016-05-26
author:     "Woody"
catalog: true
tags:
    - Android
    - UI
    - 微信
---

## 简介

朋友圈查看大图左右滑动到边距时有个阻尼回弹效果，ViewPager默认是不支持的，so，继承ViewPager自定义一个。

## 关键代码

```java
    private boolean performDrag(float x) {
        boolean needsInvalidate = false;

        final float deltaX = mLastMotionX - x;
        mLastMotionX = x;

        float oldScrollX = getScrollX();
        float scrollX = oldScrollX + deltaX;
        final int width = getClientWidth();

        float leftBound = width * mFirstOffset;
        float rightBound = width * mLastOffset;
        boolean leftAbsolute = true;
        boolean rightAbsolute = true;

        final ItemInfo firstItem = mItems.get(0);
        final ItemInfo lastItem = mItems.get(mItems.size() - 1);
        if (firstItem.position != 0) {
            leftAbsolute = false;
            leftBound = firstItem.offset * width;
        }
        if (lastItem.position != mAdapter.getCount() - 1) {
            rightAbsolute = false;
            rightBound = lastItem.offset * width;
        }

        if (scrollX < leftBound) {
			//supportDamping代表是否支持回弹
			if(!supportDamping){
				if (leftAbsolute) {
					float over = leftBound - scrollX;
					needsInvalidate = mLeftEdge.onPull(Math.abs(over) / width);
				}
				scrollX = leftBound;
			}
        } else if (scrollX > rightBound) {
			if(!supportDamping){
				if (rightAbsolute) {
					float over = scrollX - rightBound;
					needsInvalidate = mRightEdge.onPull(Math.abs(over) / width);
				}
				scrollX = rightBound;
			}
        }
        // Don't lose the rounded component
        mLastMotionX += scrollX - (int) scrollX;
        scrollTo((int) scrollX, getScrollY());
        pageScrolled((int) scrollX);

        return needsInvalidate;
    }
```

完整代码参见[GitHub](https://github.com/nirvanawoody/WeixinPhotoViewer)

