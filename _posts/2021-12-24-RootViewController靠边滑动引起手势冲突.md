---
layout:     post
title:      RootViewController靠最左侧滑动会引起手势冲突
category: iOS
tags: [iOS]
description: 真机测试我们经常会遇到在RootViewController往最左侧一直滑动的时候会引起页面卡死。尤其是当点击要push一个新页面时
---

# RootViewController最左侧一直滑动引起手势冲突

## 1、案例一

我们的页面组成是整体是UITabbarController，然后下面有4个tabbar，每个tabbar对应一个UINavigationController。

对应下面截图：
![img] (手势冲突案例1.png)

###发生的问题

当我们在首页/商城页面时，如果一直往左滑动（很靠边的滑），就会出现页面卡死的情况。从后台切换到前台，卡死的页面又正常了。

###原因
RootViewController本身是有左滑事件的，interactivePopGestureRecognizer。但是iPhone本身如果一直左滑是会退出App的。这两种手势会冲突，导致再次点击页面push的时候就会卡死

###解决方案：

```c++
在ViewController的viewDidAppear方法中设置
1、self.navigationController.interactivePopGestureRecognizer.enabled = NO;
2、UINavigationControllerDelegate
如果有继承UINavigationController的子类，则在该子类中设置UINavigation的delegate=self;
同时在代理方法中：
- (void)navigationController:(UINavigationController *)navigationController didShowViewController:(UIViewController *)viewController animated:(BOOL)animated
{
    self.interactivePopGestureRecognizer.enabled = NO;
}
```

## 2、案例二

我们的页面组成是整体是UITabbarController，然后下面有4个tabbar，每个tabbar对应一个UINavigationController。当前RootViewController头部有多个tab：推荐/保险/健康。每个tab对应的是childController。本质是一个ViewController包含多个childViewController.

对应下面截图：
![img] (手势冲突案例2.png)

###发生的问题

当我们在头条页面时，在childController位置一直往最左侧滑，再往上在tab和childController交汇处，一直左滑，再点击某个跳转页面（带动画）。则页面会卡死。从后台切换到前台又恢复正常。

###原因
R手势pop的问题。当处在navi的根控制器时候, 做一个侧滑pop的操作, 看起来没任何变化, 但是再次push其它控制器时候就会出现上述问题了。这种情况是会出现在我们自定义的navigation中，因为继承自UINavigation后，原先的右划手势被禁掉了，而我们经常会加上一句话打开手势
```c++
    self.interactivePopGestureRecognizer.delegate = self;

```
这时候我们如果在根视图里面执行右划手势，相当于执行了一个pop操作（只是我们没有看到效果而已）。然后接着去执行push，自然就push不到下一级页面了。

###解决方案：

```c++
同方案一，只不过在方案一的基础之上，还要保证childViewController
 1、self.navigationController.interactivePopGestureRecognizer.enabled = NO;
 2、同方案一
```
