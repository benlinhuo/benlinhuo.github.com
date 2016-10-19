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

![Core Animation 类图](/assets/images/2016-08-10-animationClass.png)

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

#### iOS 动画的调用方式

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

### CAPropertyAnimation

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


### CABasicAnimation

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


### CAKeyframeAnimation

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

### CATransition

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
