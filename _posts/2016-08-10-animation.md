---
layout:     post
title:      动画的基础知识总结 
category: iOS
tags: [iOS]
description: iOS 动画绝大部分都是通过 Core Animation 框架来完成的。Core Animation 会将大部分的实际绘图任务交给图形硬件来处理，图形硬件会加速图形渲染的速度。所以使用 Core Animation 制作的动画都拥有更高的帧率，而且显示效果更加平滑，当然它不会加重 CPU 的负担而影响程序的运行速度。
---

## Core Animation
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Core Animation提供了很多类来完成动画效果，有些复杂效果只需要几行代码就可以完成。本篇文章会针对这些类做介绍，后面部分会利用这些类完成一些复杂的动画效果，并对这些效果做解析。它有对应的：[Demo](https://github.com/benlinhuo/AnimationSet).

### 类图

![Core Animation 类图](/assets/images/2016-08-10-animationClass.png)

### 类解析

#### CAAnimation / CAMediaTiming

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;CAAnimation类是所有动画对象的父类，负责控制动画的持续时间和速度等，是个 `抽象类`，不能直接使用，应该使用它具体的子类。这个类实现了协议 `CAMediaTiming` ，该协议定义了多个属性，所以实现这个协议的类也就拥有了这些属性。定义的代码：`@interface CAAnimation : NSObject
    <NSCoding, NSCopying, CAMediaTiming, CAAction>`


##### `CAMediaTiming` 协议定义的属性如下：

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

##### CAAnimation 自带的属性如下：

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

##### iOS 动画的调用方式

1. 方式一：使用 Core Animation 类

```
 	CABasicAnimation *animation = [CABasicAnimation animationWithKeyPath:@"position"];
    animation.fromValue = [NSValue valueWithCGPoint:CGPointMake(0, SCREEN_HEIGHT / 2 -75)];
    animation.toValue = [NSValue valueWithCGPoint:CGPointMake(SCREEN_WIDTH, SCREEN_HEIGHT / 2 - 75)];
    animation.duration = 1.f;
    [_demoView.layer addAnimation:animation forKey:@"positionAnimation"];
```

2. 方式二：UIView［begin commit］模式

```
    _demoView.frame = CGRectMake(0, SCREEN_HEIGHT / 2 - 50, 50, 50);
    [UIView beginAnimations:nil context:nil];
    [UIView setAnimationDuration:1.f];
    _demoView.frame = CGRectMake(SCREEN_WIDTH, SCREEN_HEIGHT / 2 - 50, 50, 50);
    [UIView commitAnimations];
```

3. 方式三：UIView 代码块调用

```
	_demoView.frame = CGRectMake(0, SCREEN_HEIGHT / 2 - 50, 50, 50);
    [UIView animateWithDuration:1.f animations:^{
        _demoView.frame = CGRectMake(SCREEN_WIDTH, SCREEN_HEIGHT / 2 - 50, 50, 50);
        
    } completion:^(BOOL finished) {
        _demoView.frame = CGRectMake(SCREEN_WIDTH / 2 - 25, SCREEN_HEIGHT / 2 - 50, 50, 50);
    }];
```

#### CAPropertyAnimation

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;是CAAnimation的子类，也是个抽象类，要想创建动画对象，应该使用它的两个子类：CABasicAnimation和CAKeyframeAnimation。上述类图中可以看到：

1. 它有 keyPath 属性：通过指定CALayer的一个属性名做为keyPath里的参数(NSString类型)，并且对CALayer的这个属性的值进行修改，达到相应的动画效果。比如，指定@”position”为keyPath，就修改CALayer的position属性的值，以达到平移的动画效果。
2. `+ (instancetype)animationWithKeyPath:(nullableNSString *)path;` 类方法，这就是用于创建 CABasicAnimation 和 CAKeyframeAnimation 动画实例的类方法。如：`CABasicAnimation *animation = [CABasicAnimation animationWithKeyPath:@"position.y"];`。当然它的两个子类不用该方法创建，还是可使用 `CABasicAnimation *animation = [CABasicAnimation animation]; animation.keyPath = @"position.y";`，这种方式与上述创建方式等同。

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


#### CABasicAnimation

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;这个类提供了基于 `keyPath` 创建的基本动画，这个 `keyPath` 可以是上述列出来的任何属性，动画的变化内容。[demo](https://github.com/benlinhuo/AnimationSet) 中有对应的动画实例。

```
@interface CABasicAnimation : CAPropertyAnimation

@property(nullable, strong) id fromValue;
@property(nullable, strong) id toValue;
@property(nullable, strong) id byValue;

@end
```
- toValue: keyPath相应属性的结束值，到某个固定的值（类似transform的make含义）
注意：随着动画的进行,在长度为duration的持续时间内,keyPath相应属性的值从fromValue渐渐地变为toValue.
如果 `fillMode = kCAFillModeForwards和removedOnComletion = NO;` 那么在动画执行完毕后,图层会保持显示动画执行后的状态,但实质上,图层的属性值还是动画执行前的初始值,并没有真正被改变。比如: CALayer的postion初始值为(0,0),CABasicAnimation的fromValue为(10,10),toValue为 (100,100),虽然动画执行完毕后图层保持在(100,100) 这个位置,实质上图层的position还是为(0,0)。
- byValue: 不断进行累加的数值。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;上述三个属性配合使用达到动画效果。同时使用 byValue 和 fromValue ，那 toValue=fromValue+byValue。同时使用 byValue 和 toValue，那 fromValue=toValue - byValue。所以 fromValue、byValue、toValue，只能使用三者之二，不能三者同时使用。


#### CAKeyframeAnimation

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;它是在 CABasicAnimation 基础上做了扩展。CABasicAnimation只能从一个数值（fromValue）变到另一个数值（toValue）。CAKeyframeAnimation 可以使用一个数组来保存这些数值，实现多个点之间的动画效果。而 CABasicAnimation 可看作是最多只有2个关键帧的 CAKeyframeAnimation。

```
@interface CAKeyframeAnimation : CAPropertyAnimation

// 这是个数组类型，里面的元素称为“关键帧”（NSValue类型），动画对象会在指定时间内，依次展示其中的帧（第一个元素就是最开始的位置，所以对应到 keyTimes ，一般其中第一个元素值就是0，除非有特殊需求）
@property(nullable, copy) NSArray *values;

// 让层跟着路径移动，path 只对 CALayer 的 anchorPoint 和 position 起作用，如果设置了 path，则 values 将被忽略。
@property(nullable) CGPathRef path;

// 为对应的关键帧指定对应的时间点，如果不指定，各个关键帧之间的时间是平分的。取值范围[0,1]。各个关键帧的时间控制。前面使用values设置了四个关键帧，默认情况下每两帧之间的间隔为:8/(4-1)秒。如果想要控制动画从第一帧到第二针占用时间4秒，从第二帧到第三帧时间为2秒，而从第三帧到第四帧时间2秒的话，就可以通过keyTimes进行设置。keyTimes中存储的是时间占用比例点，此时可以设置keyTimes的值为0.0，0.5，0.75，1.0（当然必须转换为NSNumber），也就是说1到2帧运行到总时间的50%，2到3帧运行到总时间的75%，3到4帧运行到8秒结束。
@property(nullable, copy) NSArray<NSNumber *> *keyTimes;

// 我们可以为每一帧都指定 CAMediaTimingFunction * timingFunction （动画的速度模式）
@property(nullable, copy) NSArray<CAMediaTimingFunction *> *timingFunctions;

// 动画计算模式。还拿上面keyValues动画举例，之所以1到2帧能形成连贯性动画而不是直接从第1帧经过8/3秒到第2帧是因为动画模式是连续的（值为kCAAnimationLinear，这是计算模式的默认值）；而如果指定了动画模式为kCAAnimationDiscrete离散的那么你会看到动画从第1帧经过8/3秒直接到第2帧，中间没有任何过渡。其他动画模式还有：kCAAnimationPaced（均匀执行，会忽略keyTimes）、kCAAnimationCubic（平滑执行，对于位置变动关键帧动画运行轨迹更平滑）、kCAAnimationCubicPaced（平滑均匀执行）。具体可见下图
@property(copy) NSString *calculationMode;

// 不常用
@property(nullable, copy) NSArray<NSNumber *> *tensionValues;
@property(nullable, copy) NSArray<NSNumber *> *continuityValues;
@property(nullable, copy) NSArray<NSNumber *> *biasValues;

// 旋转模式：设置为kCAAnimationRotateAuto 或 kCAAnimationRotateAutoReverse 会随着旋转的角度做 ”自转“。它默认值是 nil，表示跟没有这个属性一样
@property(nullable, copy) NSString *rotationMode;

@end
```

![calculationMode](/assets/images/2016-08-10-calculationMode.png)

相关代码可见 [demo](https://github.com/benlinhuo/AnimationSet)

### CAAnimationGroup

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;我们可以将创建的多个动画添加到一个组中，那这些动画便会刻意同时并发执行。这个比较简单，它只有一个属性 `animations` ，是个数值，存放多个动画的集合。

```
@interface CAAnimationGroup : CAAnimation

@property(nullable, copy) NSArray<CAAnimation *> *animations;

@end
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;如果我们直接把多个动画添加到同一个 layer 上，则这多个动画也是同时并发执行的（如果这多个动画的开始时间相同）。

```
- (void)simultaneouslyAnimation
{
    //位移动画
    CAKeyframeAnimation *anima1 = [CAKeyframeAnimation animationWithKeyPath:@"position"];
    NSValue *value0 = [NSValue valueWithCGPoint:CGPointMake(0, SCREEN_HEIGHT / 2 - 50)];
    NSValue *value1 = [NSValue valueWithCGPoint:CGPointMake(SCREEN_WIDTH / 3, SCREEN_HEIGHT / 2 - 50)];
    NSValue *value2 = [NSValue valueWithCGPoint:CGPointMake(SCREEN_WIDTH / 3, SCREEN_HEIGHT / 2 + 50)];
    NSValue *value3 = [NSValue valueWithCGPoint:CGPointMake(SCREEN_WIDTH * 2 / 3, SCREEN_HEIGHT / 2 + 50)];
    NSValue *value4 = [NSValue valueWithCGPoint:CGPointMake(SCREEN_WIDTH * 2 / 3, SCREEN_HEIGHT / 2 - 50)];
    NSValue *value5 = [NSValue valueWithCGPoint:CGPointMake(SCREEN_WIDTH, SCREEN_HEIGHT / 2 - 50)];
    anima1.values = [NSArray arrayWithObjects:value0,value1,value2,value3,value4,value5, nil];
    anima1.duration = 4.0f;
    [_demoView.layer addAnimation:anima1 forKey:@"aa"];
    
    //缩放动画
    CABasicAnimation *anima2 = [CABasicAnimation animationWithKeyPath:@"transform.scale"];
    anima2.fromValue = [NSNumber numberWithFloat:0.8f];
    anima2.toValue = [NSNumber numberWithFloat:2.0f];
    anima2.duration = 4.0f;
    [_demoView.layer addAnimation:anima2 forKey:@"bb"];
    
    //旋转动画
    CABasicAnimation *anima3 = [CABasicAnimation animationWithKeyPath:@"transform.rotation"];
    anima3.toValue = [NSNumber numberWithFloat:M_PI*4];
    anima3.duration = 4.0f;
    [_demoView.layer addAnimation:anima3 forKey:@"cc"];
}
```

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;如果我们指定动画执行时间 duration 和 开始时间 beginTime 与另一动画的开始时间衔接上，那就可以形成动画A执行完，动画B开始执；动画B执行完，动画C开始执行的连续动画。

```
// 使用动画开始执行时间的控制来实现不同动画的连续执行
- (void)continuousAnimation
{
    CFTimeInterval currentTime = CACurrentMediaTime();
    //位移动画
    CABasicAnimation *anima1 = [CABasicAnimation animationWithKeyPath:@"position"];
    anima1.fromValue = [NSValue valueWithCGPoint:CGPointMake(0, SCREEN_HEIGHT / 2 - 75)];
    anima1.toValue = [NSValue valueWithCGPoint:CGPointMake(SCREEN_WIDTH / 2, SCREEN_HEIGHT / 2 - 75)];
    anima1.beginTime = currentTime;
    anima1.duration = 1.0f;
    anima1.fillMode = kCAFillModeForwards;
    anima1.removedOnCompletion = NO;
    [_demoView.layer addAnimation:anima1 forKey:@"aa"];
    
    //缩放动画
    CABasicAnimation *anima2 = [CABasicAnimation animationWithKeyPath:@"transform.scale"];
    anima2.fromValue = [NSNumber numberWithFloat:0.8f];
    anima2.toValue = [NSNumber numberWithFloat:2.0f];
    anima2.beginTime = currentTime+1.0f;
    anima2.duration = 1.0f;
    anima2.fillMode = kCAFillModeForwards;
    anima2.removedOnCompletion = NO;
    [_demoView.layer addAnimation:anima2 forKey:@"bb"];
    
    //旋转动画
    CABasicAnimation *anima3 = [CABasicAnimation animationWithKeyPath:@"transform.rotation"];
    anima3.toValue = [NSNumber numberWithFloat:M_PI*4];
    anima3.beginTime = currentTime+2.0f;
    anima3.duration = 1.0f;
    anima3.fillMode = kCAFillModeForwards;
    anima3.removedOnCompletion = NO;
    [_demoView.layer addAnimation:anima3 forKey:@"cc"];

}
```

#### CATransition

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;用于做专场动画，能够为层提供移出屏幕和移入屏幕的动画效果。如我们的 UINavigationController 可以使用 CATransition 的各种效果将控制器的视图推入屏幕。[Demo](https://github.com/benlinhuo/AnimationSet) 中有针对各种的 type 效果案例。

```
@interface CATransition : CAAnimation

@property(copy) NSString *type;

@property(nullable, copy) NSString *subtype;

// 动画起点和终点（它是指在整体动画的百分比，所以取值范围为[0, 1]）
@property float startProgress;
@property float endProgress;

@property(nullable, strong) id filter;

@end
```

- type：设置动画过渡的类型，它分两部分：一个是对外公开的，还有一部分是 private API。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;对外公开的有如下枚举：

```
kCATransitionFade      交叉淡化过渡
kCATransitionMoveIn    新视图移到旧视图上面
kCATransitionPush      新视图把旧视图推出去
kCATransitionReveal    将旧视图移开,显示下面的新视图

其他类型包装成字符串赋值（因为是 private API，所以就没有设置枚举）
``` 
![转场动画过渡效果](/assets/images/2016-08-10-transitionType.png)

- subType: 设置动画过渡方向，当然有一些效果可能就不具有方向。如 kCATransitionFade 淡化过渡效果，上图“转场动画过渡效果”也有指定。过渡方向有如下几种：

```
kCATransitionFromRight
kCATransitionFromLeft
kCATransitionFromTop
kCATransitionFromBottom
```

- filter：它默认为nil，一旦设置了此属性，type和subtype就会被忽略。这个属性一般用的很少。


## 2D仿射变换

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;仿射变换－`CGAffineTransform`，它是 `CoreGraphics` 框架中的类，用于设定 `UIView`  的 `transform` 属性，以控制视图的缩放、旋转和平移操作（是二维空间），实现的效果和上述 `Core Animation` 方案相同，不过它可能写的代码行数更少。`CGAffineTransform` 是一个可以和二维空间向量（如 CGPoint）做乘法的 3X2 矩阵，所以称为仿射变换，“仿射” 的意思是无论变换矩阵用什么值，图层中平行的两条线在变换之后仍然保持平行。

```
struct CGAffineTransform {
  CGFloat a, b, c, d;
  CGFloat tx, ty;
};
```
这是代码中定义的，它对应 3*3 矩阵的变换

![3X3矩阵](/assets/images/transform.png)

### CGAffineTransformIdentity

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;在线性代数中，这是恒等变换。一般在我们做完动画后，再重新动画，则需要先归位，即设置 `view.transform = CGAffineTransformIdentity;` ，否则的话后面动画可能就不是自己想要的效果，因为它的起始状态不是自己一开始设定的状态。`CGAffineTransformIdentity` 是单位矩阵，该矩阵没有缩放、平移、旋转。

#### 直接创建变换 `CGAffineTransformMake`

```
 CGAffineTransform CGAffineTransformMake(CGFloat a, CGFloat b, CGFloat c, CGFloat d, CGFloat tx, CGFloat ty)
```
直接创建变换，所需参数比较多，各个参数对应上述 3X3 矩阵的前两列。一般来说，我们是不使用这个来创建变换，计算比较复杂。

#### 直接创建变换 `CGAffineTransformMake?`

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;这种方式创建的变换都是针对某一种情况，例如移动、缩放、旋转。

```
CGAffineTransform CGAffineTransformMakeTranslation(CGFloat tx,
  CGFloat ty)
CGAffineTransform CGAffineTransformMakeScale(CGFloat sx, CGFloat sy)
CGAffineTransform CGAffineTransformMakeRotation(CGFloat angle)
```

1. `CGAffineTransformMakeTranslation` 创建一个平移的变化。即如果是一个 `UIView`，表示它的起始位置 x 会加上 tx，y 会加上 ty。
2. `CGAffineTransformMakeScale` 创建一个给定比例缩放的变换。如 `UIView`，引用了这个变换，则图片的宽度就会变成 `width*sx`，对应高度变成 `height*sy`。
	- `CGAffineTransformMakeScale(-1.0, 1.0)` ，表示水平翻转
	- `CGAffineTransformMakeScale(1.0, -1.0)` ，表示垂直翻转
3. `CGAffineTransformMakeRotation` 创建一个旋转角度的变化，参数是一个弧度。意思表示从当前位置旋转多少度。


#### 在已有变换基础上再添加另一种动效

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;比如有的时候，我们希望在缩放的同时平移，所以 `仿射变换` 也支持几种动画效果的叠加。

```
CGAffineTransform CGAffineTransformTranslate(CGAffineTransform t,
  CGFloat tx, CGFloat ty)
CGAffineTransform CGAffineTransformScale(CGAffineTransform t,
  CGFloat sx, CGFloat sy)
CGAffineTransform CGAffineTransformRotate(CGAffineTransform t,
  CGFloat angle)
```

这三个方法的第一个参数，都是已定义的一种变换，后面是再添加一种动效：平移、缩放、旋转。跟之前的解析一致。


#### 转换的改变

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;我们在定义好一种转换以后，可以对它做变化。

```
CGAffineTransform CGAffineTransformInvert(CGAffineTransform t)
CGAffineTransform CGAffineTransformConcat(CGAffineTransform t1,
  CGAffineTransform t2)
```
-  `CGAffineTransformInvert` 表示反向动画，意思是我指定当前大小放大2倍。则反向之后就是：从2倍大小缩小到正常大小
-  `CGAffineTransformConcat` 表示返回一个由 t1 和 t2 合并而成的转换

#### 转换的运用

```
CGPoint CGPointApplyAffineTransform(CGPoint point,
  CGAffineTransform t)
CGSize CGSizeApplyAffineTransform(CGSize size, CGAffineTransform t)
CGRect CGRectApplyAffineTransform(CGRect rect, CGAffineTransform t)
```

- `CGPointApplyAffineTransform` 把变化应用到一个点上，它返回还是一个点。所以这个方法最终也只是影响这个点所在的位置。如下实例：

```
// 把变化应用到一个点上
_demoView.transform = CGAffineTransformIdentity;
[UIView animateWithDuration:1.0f animations:^{
     CGAffineTransform t1 = CGAffineTransformMakeTranslation(100, 100);
     CGPoint point = CGPointApplyAffineTransform(CGPointMake(50, 50), t1);
     NSLog(@"point.x = %f, point.y = %f", point.x, point.y);
}];

// 打印结果为：
HBLAnimationSet[38618:9862114] point.x = 150.000000, point.y = 150.000000

所以只影响到指定的点（50，50），平移（100，100）之后，变成了（150，150）了
```

- `CGSizeApplyAffineTransform ` 把变化应用到一个区域上，它返回还是一个区域。所以这个方法最终也只是影响这个区域大小。如下实例：

```
实例1:
// 把变化应用到一个区域上
_demoView.transform = CGAffineTransformIdentity;
[UIView animateWithDuration:1.0f animations:^{
    CGAffineTransform t1 = CGAffineTransformMakeTranslation(100, 100);
    CGSize size = CGSizeApplyAffineTransform(CGSizeMake(50, 50), t1);
    NSLog(@"size.width = %f, size.height = %f", size.width, size.height);
}];

// 打印结果：
HBLAnimationSet[38666:9896059] size.width = 50.000000, size.height = 50.000000   
区域大小没变
```

当我们把变换由平移改成了缩放，则区域大小就应该变化了

```
// 把变化应用到一个区域上
_demoView.transform = CGAffineTransformIdentity;
[UIView animateWithDuration:1.0f animations:^{
    CGAffineTransform t1 = CGAffineTransformMakeScale(2, 2);
    CGSize size = CGSizeApplyAffineTransform(CGSizeMake(50, 50), t1);
    NSLog(@"size.width = %f, size.height = %f", size.width, size.height);
}];

// 打印结果
HBLAnimationSet[38701:9909686] size.width = 100.000000, size.height = 100.000000
发现区域大小的长和宽都变成原来2倍了
```

- `CGRectApplyAffineTransform ` 就是集合点和区域大小的一个变换影响。

#### 转换的检测

```
// 判断两个转换是否相等，如下代码便可知，只要两个动画的效果一样，则二者便相等
bool CGAffineTransformEqualToTransform(CGAffineTransform t1,
  CGAffineTransform t2)
  
// 判断当前转换是否处于原始状态
bool CGAffineTransformIsIdentity(CGAffineTransform t)
```
判断转换的实例如下：

```
// 判断两个转换是否相等的含义
CGAffineTransform t1 = CGAffineTransformMakeTranslation(100, 100);
CGAffineTransform t2 = CGAffineTransformMakeTranslation(100, 100);
NSLog(@"t1是否等于t2：%d", CGAffineTransformEqualToTransform(t1, t2));

// 打印结果
HBLAnimationSet[38495:9807678] t1是否等于t2：1
```


### UIView 的类方法实现转场动画

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;如下两个方法是用于创建过渡动画的，主要用于 UIView 进入或者离开视图。

```
// 单视图
+ (void)transitionWithView:(UIView *)view duration:(NSTimeInterval)duration options:(UIViewAnimationOptions)options animations:(void (^ __nullable)(void))animations completion:(void (^ __nullable)(BOOL finished))completion 

// 双视图
+ (void)transitionFromView:(UIView *)fromView toView:(UIView *)toView duration:(NSTimeInterval)duration options:(UIViewAnimationOptions)options completion:(void (^ __nullable)(BOOL finished))completion 
```
我们之前有见过视图控制器（ `UIViewController`） 的转场动画，那其实视图（`UIView`）也是可以进行转场动画的。

其实 `UIView` 的几个用来动画的类方法，和使用 CAAnimation 方式的区别是：CAAnimation 动画还可以针对 CALayer 类型的图形层，而 `UIView` 就只能服务于自己。


## 3D仿射变换

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;3D，动画效果会涉及到 Z 轴，效果更炫。不过它是针对 layer 图层的属性 `transform`，2D 是针对 `UIView`的。


### iOS 坐标系统中有如下几个概念：

- position：CGPoint 类型，`它表示指定图层相对于父图层的位置`，该值是基于父图层的坐标系；
- bounds：CGRect 类型，指定图层大小（bounds.size）和原点（bounds.origin），`它是基于自身坐标系统的（所以很多情况下，bounds的origin都是（0，0））`。如果改变 bounds 的 origin，则该图层的子图层坐标都会跟着改变。也就是说，`改变自身的坐标系，本身在父图层的位置不变，但它上的子图层位置变化`
- anchorPoint：CGPoint 类型（锚点），指定了 bounds 相对于 position 的值，`同时也作为一个变化时候的中心点`。锚点使用空间坐标系取值范围是0-1，默认0.5，即中心点。

  1> anchorPoint 的图形解析如下：

  ![anchorPoint在中心位置](/assets/images/anchorPoint1.png)

  ![anchorPoint在边缘位置](/assets/images/anchorPoint2.png)

  2> anchorPoint 的移动解释如下：
  
  ![移动前，anchorPoint位置](/assets/images/anchorPointBefore.png)
  
  移动前，锚点位置如上图，为（0.5，0.5）。当我们把锚点位置更改为（0，0），变成下图，其实我们可以看出来锚点没变，只是整个视图 position 变化了。一般来说，在我们修改完 anchorPoint 之后，会引起 layer 的 position 变化，所以一般需要重新修改 frame 来使 layer 回到正确位置上。（应该注意修改顺序，frame 实际是不保存，没有值的，它是根据 position 和 anchorPoint 算出来的）
  
  ![移动后，anchorPoint位置](/assets/images/anchorPointAfter.png)

- frame：它本身是不会保存的，它是根据 position 和 bounds 获取到的。

### CATransform3D

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`CATransform3D`数据结构定义了一个同质的三维变换（4X4矩阵），用于图层的旋转、缩放、偏移、歪斜和应用的透视。图层的两个属性指定了变换矩阵：transform（是结合 anchorPoint 的位置来对图层和图层上的子图层进行变换） 和 sublayers.transform（结合 anchorPoint 的位置来对图层进行变换，不包括本身）。使用需要在工程中导入 `QuartzCore.framework`，文件中 `#import <QuartzCore/CATransform3D.h>`。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`CATransform3DIdentity` 是单位矩阵，它没有缩放、旋转、歪斜、透视。该矩阵应用到图层上，就是设置了默认值的。

```
struct CATransform3D
{
CGFloat     m11（x缩放）,    m12（y切变）,      m13（旋转）,     m14（）;

CGFloat     m21（x切变）,    m22（y缩放）,      m23（）,        m24（）;

CGFloat     m31（旋转）,     m32（ ）,         m33（）,        m34（透视效果，要操作的这个对象要有旋转的角度，否则没有效果。正直/负值都有意义）;

CGFloat     m41（x平移）,    m42（y平移）,      m43（z平移）,   m44（）;
};
```

说明：

- 整体比例变换时，也就是m11==m22时，若m33>1，图形整体缩小，若0<m33<1，图形整体放大，若m33<0,发生关于原点的对称等比变换。
- 单设ｍ12或ｍ21的时候是切变效果，当【ｍ12=角度】和【ｍ21=－角度】的时候就是旋转效果了。两个角度值相同。

![等式](/assets/images/3dTransform.png)

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;由上可以看到，m34实际影响了 Z 轴方向的 translation，m34=-1/D，默认值是0，我们应该尽可能的让 m34 这个值尽可能小，但又必须有明显的远小近大的效果。

```
-(CATransform3D)getTransForm3DWithAngle:(CGFloat)angle
{
        
     CATransform3D transform =CATransform3DIdentity;//获取一个标准默认的CATransform3D仿射变换矩阵
     transform.m34=4.5/-2000;//透视效果
     transform=CATransform3DRotate(transform,angle,0,1,0);//获取旋转angle角度后的rotation矩阵。  
     return transform;   
}
```
如上，我们的视差可以通过在旋转的时候使得离视角近的地方放大，离视角远的地方缩小，即所谓的视差来形成3D的效果。如上代码，`设置透视效果的第二行代码一定要在第三行代码之前`。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;3D 仿射变换的 API 和 2D类似，具体不详解释了。不过因为加入了 Z 轴，有几个注意点：

1. 在平移的时候，有 tz，表示Z轴偏移位置，往外为正数。它是值越大，图层越往外（越接近人眼）。在两个图层叠加的时候，可以看的很明显。

## 总结

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;以上都是一些动画的基础知识，如果我们想要构建出复杂好看的动画效果，则可能要融合上述多种动画方式。下一节会有几个常见的复杂动画实现，和上述简单动画在一个 demo 中。






