---
layout:     post
title:      Core Animation 基础知识总结 
category: iOS
tags: [iOS]
description: iOS 动画绝大部分都是通过 Core Animation 框架来完成的。Core Animation 会将大部分的实际绘图任务交给图形硬件来处理，图形硬件会加速图形渲染的速度。所以使用 Core Animation 制作的动画都拥有更高的帧率，而且显示效果更加平滑，当然它不会加重 CPU 的负担而影响程序的运行速度。
---

## 简介
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Core Animation提供了很多类来完成动画效果，有些复杂效果只需要几行代码就可以完成。本篇文章会针对这些类做介绍，后面部分会利用这些类完成一些复杂的动画效果，并对这些效果做解析。它有对应的：[Demo](https://github.com/benlinhuo/AnimationSet).

##类图

##类解析

### CAAnimation / CAMediaTiming

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;CAAnimation类是所有动画对象的父类，负责控制动画的持续时间和速度等，是个 `抽象类`，不能直接使用，应该使用它具体的子类。这个类实现了协议 `CAMediaTiming` ，该协议定义了多个属性，所以实现这个协议的类也就拥有了这些属性。定义的代码：`@interface CAAnimation : NSObject
    <NSCoding, NSCopying, CAMediaTiming, CAAction>`


#### `CAMediaTiming` 协议定义的属性如下：

- beginTime: 动画的开始时间，这个需要注意的是，如果你需要在5s之后执行动画，并不能简单的直接赋值。而是需要CACurrentMediaTime() + 5这样来设置。
- duration: 动画的持续时间，默认为0.25秒
- speed: 它用于设置当前对象的时间流相对于父级对象时间流的流逝速度,比如一个动画beginTime是0,但是speed是2,那么这个动画的1秒处相当于父级对象时间流中的2秒处. speed越大则说明时间流逝速度越快,那动画也就越快.比如一个speed为2的layer其所有的父辈的speed都是1,它有一个subLayer,speed也为2,那么一个8秒的动画在这个运行于这个subLayer只需2秒(8 / (2 * 2)).所以speed有叠加的效果。即：默认的值为 1.0.这意味着动画播放按照默认的速度。如果你改变这个值为 2.0,动画会用 2 倍的速度播放。 这样的影响就是使持续时间减半。如果你指定的持续时间为 6 秒,速度为 2.0,动画就会播放 3 秒钟---一半的 持续时间。
- timeOffset: 将一个动画看作一个环,timeOffset改变的其实是动画在环内的起点,比如一个duration为5秒的动画,将timeOffset设置为2(或者7,模5为2),那么动画的运行则是从原来的2秒开始到5秒,接着再0秒到2秒,完成一次动画.
- autoreverses: 在正常执行完动画以后，它会按照原动画的反向再次执行一遍。如原动画是 1倍 大小变成 0.2倍 大小，执行结束会立马执行从 0.2倍 大小变成 1倍 大小。避免了直接跳转到开始的值。
- repeatCount: 动画的重复次数
- repeatDuration: 这个属性指定了动画应该被重复多久。动画会一直重复,直到设定的时间流逝完。它不应该和 repeatCount 一起使用。 
- fillMode: 填充模式。这个有4种，我们后面细讲,其中需要注意的一点是，`fillMode只会在removeOnCompletion设置为NO的时候才会生效`。
	- kCAFillModeRemoved 这个是默认值,也就是说当动画开始前和动画结束后,动画对layer都没有影响,动画结束后,layer会恢复到之前的状态
	- kCAFillModeForwards 当动画结束后,layer会一直保持着动画最后的状态
	- kCAFillModeBackwards 这个和kCAFillModeForwards是相对的,就是在动画开始前,你只要将动画加入了一个layer,layer便立即进入动画的初始状态并等待动画开始.你可以这样设定测试代码,将一个动画加入一个layer的时候延迟5秒执行.然后就会发现在动画没有开始的时候,只要动画被加入了layer,layer便处于动画初始状态
	- kCAFillModeBoth 理解了上面两个,这个就很好理解了,这个其实就是上面两个的合成.动画加入后开始之前,layer便处于动画初始状态,动画结束后layer保持动画最后的状态.

```
@protocol CAMediaTiming

@property CFTimeInterval beginTime;
@property CFTimeInterval duration;
@property float speed;
@property CFTimeInterval timeOffset;
@property float repeatCount;
@property CFTimeInterval repeatDuration;
@property BOOL autoreverses;
@property(copy) NSString *fillMode;

@end
```

#### CAAnimation 自带的属性如下：

- timingFunction: 控制动画的显示节奏。有五种选择，如下：
	- kCAMediaTimingFucntionLinear：线性动画
	- kCAMediaTimingFucntionEaseIn：先慢后快（慢进快出）
	- kCAMediaTimingFucntionEaseOut：先快后慢（快进慢出）
	- kCAMediaTimingFucntionEaseInEaseOut：先慢后快再慢
	- kCAMediaTimingFucntionDefault：默认，属于中间比较快
- removedOnCompletion: 默认为YES，代表动画执行完毕后就从图层上移除，图形会恢复到动画执行前的状态。如果想让图层保持显示动画执行后的状态，那就设置为NO，不过还要设置fillMode属性为kCAFillModeForwards
- <CAAnimationDelegate> delegate：动画代理，检测动画的开始和结束。

```
@protocol CAAnimationDelegate <NSObject>
@optional

- (void)animationDidStart:(CAAnimation *)anim;
- (void)animationDidStop:(CAAnimation *)anim finished:(BOOL)flag;

@end
```

### CAPropertyAnimation

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;是CAAnimation的子类，也是个抽象类，要想创建动画对象，应该使用它的两个子类：CABasicAnimation和CAKeyframeAnimation。上述类图中可以看到：

1. 它有 keyPath 属性：通过指定CALayer的一个属性名做为keyPath里的参数(NSString类型)，并且对CALayer的这个属性的值进行修改，达到相应的动画效果。比如，指定@”position”为keyPath，就修改CALayer的position属性的值，以达到平移的动画效果。
2. `+ (instancetype)animationWithKeyPath:(nullableNSString *)path;` 类方法，这就是用于创建 CABasicAnimation 和 CAKeyframeAnimation 动画实例的类方法。如：`CABasicAnimation *animation = [CABasicAnimation animationWithKeyPath:@"position.y"];`。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;上述的 keyPath 整理了一下，可设置为如下的属性：

keyPath | 解析
-------|------   
transform.scale（或者 transform.scale.x/y/z） | 缩放
transform.rotation（或者 transform.rotation.x/y/z） | 旋转 
transform.translation（或者 transform.translation.x/y/z） | 位置移动（目前没发现和属性 position 有啥区别） 
margin | 
zPosition | Z 轴位置
position | 
frame | 
bounds | 
backgroundColor | 背景色
cornerRadius | 圆角
borderWidth | 
contents | 
contentsRect | 
cornerRadius | 
mask | 
masksToBounds | 
opacity | 透明度
hidden | 
shadowColor | 
shadowOffset | 
shadowOpacity | 
shadowRadius | 