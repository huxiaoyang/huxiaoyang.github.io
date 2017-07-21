---
title: 自定义ScrollView
date: 2017-07-21 11:15:08
categories: iOS
tags: 
  - Swift
  - iOS
---

UIScrollView 是iOS Cocoa框架中重要的控件，几乎每个APP中都会用到，所以使用它，是对每个iOS开发者最基本的要求。

本文试图去了解UIScrollView它的工作原理，并且尝试去自定义一个ScrollView，同时解决多层ScrollView嵌套时的手势连贯性问题：

1. 了解UIScrollView
1. 自定义ScrollView
1. 添加必要的动画效果
1. 解决ScrollView嵌套UITableView手势冲突

# 了解UIScrollView

要了解 `UIScrollView`，首先我们需要了解 `UIView` 的 [`bounds`](https://developer.apple.com/documentation/uikit/uiview/1622580-bounds) 属性：

> 边界矩形，在自己的坐标系中描述视图的位置和大小。

一般来说 `view` 的 `bounds` 原点默认是（0，0），比如iPhone4的尺寸（320，480），在iPhone4上新建一个view ：

``` bash
let view = UIView(frame: CGRect(x: 0, y: 0,width: 320, height: 1000))
```

view的高度明显超出屏幕的高度，所以屏幕内并不能完全显示view，只能展示480，所以应该是view竖直方向 `0 ~ 480` 之间的内容

我们看到的效果应该是

{% asset_img 盗图1.png 盗图 %}

这时如果我们修改 `view` 的 `bounds` ：

``` bash
var bounds = view.bounds
bounds.origin = CGPoint(x: 0, y: 100)
view.bounds = bounds
```

这时屏幕内展示的view内容应该是竖直方向 `100 ~ 580`之间的内容，看起来view向下移动了100，但是实际上这只是相对于它自己的坐标，它的 `frame` 并没有改变。

{% asset_img 盗图2.png 盗图 %}

如果你能理解这些，那你会发现，改变视图的 `bounds` 类似 `UIScrollView` 的 `contentOffset`。

# 自定义ScrollView

事实上就是如此，我们可以通过 `UIPanGestureRecognizer` 去获取手势，然后改变 `bounds` 去实现自定义一个 `ScrollView`
 大神轮子：[Demo](https://github.com/rounak/CustomScrollView)

# 添加必要的动画效果

只是实现滑动的功能很明显是不能满足 `挑剔的产品` 需求的。我们需求添加和 `UIScrollView` 一样的动画效果，惯性、弹跳、橡胶。

这里提供两种发方式可以实现这些效果：

方法一：使用 `Facebook Pop` 实现这些特效。这里不详细介绍，已经有人造了轮子

Demo：[Facebook Pop 版本](https://github.com/grp/CustomScrollView)

方法二：使用 `UIKit Dynamics` 实现这些特效。同样有现成的轮子可参考

Demo：[UIKit Dynamics 版本](https://github.com/fastred/CustomScrollView)

# 解决ScrollView嵌套UITableView手势冲突

基本的思路是自定义 `ScrollView` 然后在 `UIPanGestureRecognizer` 手势结束后，进行速度传递

轮子：[ScrollView嵌套UITableView](https://github.com/huxiaoyang/CustomScrollView)

# 参考资料

> 提示：如果不能打开，请自行翻墙

[https://oleb.net/blog/2014/04/understanding-uiscrollview/](https://oleb.net/blog/2014/04/understanding-uiscrollview/)
[http://holko.pl/2014/07/06/inertia-bouncing-rubber-banding-uikit-dynamics/](http://holko.pl/2014/07/06/inertia-bouncing-rubber-banding-uikit-dynamics/) 
[http://iosdevtips.co/post/84571595353/replicating-uiscrollviews-deceleration-with#_=_](http://iosdevtips.co/post/84571595353/replicating-uiscrollviews-deceleration-with#_=_)
[http://mobilists.eleme.io/2016/03/15/%E7%94%A8UIKit-Dynamics%E6%A8%A1%E4%BB%BFUIScrollView/](http://mobilists.eleme.io/2016/03/15/%E7%94%A8UIKit-Dynamics%E6%A8%A1%E4%BB%BFUIScrollView/)