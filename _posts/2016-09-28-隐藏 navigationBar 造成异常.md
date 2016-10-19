---
layout:     post
title:      隐藏 navigationBar 造成导航栏错乱
category: iOS
tags: [iOS]
description: 隐藏 navigationBar 经常容易造成异常UI，严重影响体检效果
---

## 隐藏导航栏方式

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;导航栏的隐藏很简单，如下前四种方式，只需要在 `ViewController` 的 `viewWillAppear` 中调用，然后在 `viewWillDisappear` 中显示（用隐藏对应的方法）。但是仅仅这么做，都会带来一些特殊情况下 UI 上的 bug（奇怪的 bug 可以查看参考链接中的介绍）。这些 bug 比较让人头疼，因为你可能使用某种方案把当前这个 UI bug 搞定了，但是又会在另一种特殊情况下出现 bug。但是第5种方案比较靠谱，目前用来出现的 bug 都在解决范围之内。有相关的 [Demo](https://github.com/benlinhuo/BLOGExample/tree/master/HBLNavigationStatusBarExample).
 
1. [self.navigationController setNavigationBarHidden: YES];
2. [self.navigationController setNavigationBarHidden: YES animated: NO];
3. [self.navigationController setNavigationBarHidden: YES animated: YES];
4. [self.navigationController setNavigationBarHidden: YES animated: animated];
5. self.navigationController.delegate = self


## 隐藏导航栏的有效方法

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;隐藏导航栏一个比较靠谱的方式就是使用：`self.navigationController.delegate = self`。一般情况下，我们应该自己写一个继承于 `UINavigationController` 的子类 `HBLBaseNavigationController`（[Demo](https://github.com/benlinhuo/BLOGExample/tree/master/HBLNavigationStatusBarExample)），可以在 `- (id)initWithRootViewController:(UIViewController *)rootViewController
` 方法中设置 `self.delegate = self;`。`UINavigationControllerDelegate` 代理中有一个方法 `- (void)navigationController:(UINavigationController *)navigationController willShowViewController:(UIViewController *)viewController animated:(BOOL)animated` ，在该方法中做如下处理：

```
- (void)navigationController:(UINavigationController *)navigationController willShowViewController:(UIViewController *)viewController animated:(BOOL)animated
{
    if([viewController isKindOfClass:[HBLBaseViewController class]])
    {
        HBLBaseViewController *baseVC = (HBLBaseViewController*)viewController;
        if([baseVC isHiddenNavigationBar])
        {
            [self setNavigationBarHidden:YES animated:YES];
        }
        else
        {
            [self setNavigationBarHidden:NO animated:YES];
        }
        // 这部分主要是设置导航栏左侧的返回按钮
        if([baseVC shouldShowLeft])
        {
            if(viewController.navigationItem.leftBarButtonItem == nil)
            {
                UIBarButtonItem* leftItem = [[UIBarButtonItem alloc]initWithImage:[UIImage imageNamed:@"nav_icon_back_normal"]
                                                                            style:UIBarButtonItemStylePlain
                                                                           target:self
                                                                           action:@selector(popAction:)];
                viewController.navigationItem.leftBarButtonItem = leftItem;
            }
        }
        else
        {
            viewController.navigationItem.leftBarButtonItem = nil;
        }
    }
    else
    {
        [self setNavigationBarHidden:NO animated:YES];
    }
}

```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;但为了我们能正确处理 `pop` 手势，我们一般都是在 `push` / `pop` 等过程中禁掉该手势 `    self.interactivePopGestureRecognizer.enabled = NO;
` ，等视图控制器可见的时候再使用该手势，如下代码：

```
// 它用一个对外公开的方法 ｀shouldPopGestureEnable｀ 来让用户指定某个视图控制器是否可以使用 pop 手势
- (void)navigationController:(UINavigationController *)navigationController didShowViewController:(UIViewController *)viewController animated:(BOOL)animated
{
    BOOL isEnable = YES;
    if([viewController isKindOfClass:[HBLBaseViewController class]])
    {
        HBLBaseViewController *baseVC = (HBLBaseViewController*)viewController;
        isEnable = [baseVC shouldPopGestureEnable];
    }
    self.interactivePopGestureRecognizer.enabled = isEnable;
}
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;其中还有一个问题：在 iOS7 及以后，如果使用 UINavigationController 或者其子类，则系统自带的附加了一个从屏幕左侧边缘开始滑动可以实现 pop 的手势。但是，因为上面我们在方法 `- (void)navigationController:(UINavigationController *)navigationController willShowViewController:(UIViewController *)viewController animated:(BOOL)animated` 中自定义了 `navigaitonItem` 的 `leftBarButtonItem`，则这个手势就会失效。解决方案有多种，我在 Demo 中使用的方案是在 `HBLBaseViewController` 或者 `HBLBaseNavigationController` 类中重新设置手势的 `delegate`：`self.navigationController.interactivePopGestureRecognizer.delegate = (id<UIGestureRecognizerDelegate>)self;`。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;运行 Demo，中间的两个tab：`微聊`：自己是有导航栏，分别以不同方式到另一个页面（另一个页面既可能是有导航栏，也可能没有导航栏）；`日记`：自己是没有导航栏的，和 `微聊` 一样，以不同方式到不同页面。这样的多种混合情况，在测试过程中并没有发现有异常 UI bug 。


## 上述有效方案的漏洞

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;上述的解决方案，在我们变化 `StatusBarStyle` 为白色 (`我的` tab 演示)，或者页面在滑动过程中，变化 `StatusBarStyle` （`首页` tab 演示），等进入下一个页面再返回时（多次使用 pop 手势），出现了导航栏错乱的 UI。如下图：

![我的页面](https://github.com/benlinhuo/benlinhuo.github.com/blob/master/assets/images/mytab.gif)
![首页页面](https://github.com/benlinhuo/benlinhuo.github.com/blob/master/assets/images/hometab.gif)

如上问题的解决方案有两种。

### 方案一
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;我之前设置 `StatusBarStyle` 的方式是使用 `- (UIStatusBarStyle)preferredStatusBarStyle
` 方法。如果我们能把设置的方式更改为全局设置 `[UIApplication sharedApplication].statusBarStyle = UIStatusBarStyleLightContent`，即如果某个页面要是白色的 `statusBar`，那我们可以在其 `viewWillAppear` 中设置白色，然后在 `viewDidDisappear` 方法中设置 `UIStatusBarStyleDefault` 即可。

### 方案二

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;在 `HBLBaseNavigationController` 类中添加如下方法的实现（具体原理后面会有讲解）：

```
- (UIViewController *)childViewControllerForStatusBarStyle
{
    return self.topViewController;
}
```

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;方案二相比而言更好，因为在 iOS7 之后，就已经推崇使用 `preferredStatusBarStyle ` 方式来更改状态栏了，所以顺应趋势。而且方案一，你需要在开始设置需要的状态值，然后在结束时释放当前页面需要的状态值（因为更多页面还是使用默认的状态值）。有时候更麻烦的逻辑，可能还需要标志位来区分，虽然解决了问题，但增加了逻辑的复杂度。

## 总结：更改状态栏样式的方案（iOS7及之后）

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;状态栏就是我们设置显示电池电量、时间、网络部分标识的颜色，它只能设置为两种：

1. 黑色（UIStatusBarStyleDefault，默认的）
2. 白色(UIStatusBarStyleLightContent)

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;在 iOS8 中，`UIStatusBarStyleBlackTranslucent` 与 `UIStatusBarStyleBlackOpaque` 相当于 `UIStatusBarStyleLightContent`。

### 方式一：全局设置

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;在 info.plist 文件中将 `View controller-based status bar appearance` 字段设置为 NO (如果不设置为 NO，则如下的全局设置方式将不起作用)，然后通过这种方式 `[UIApplication sharedApplication].statusBarStyle = UIStatusBarStyleLightContent` 。这种就是全局的设置方式，也就是它是针对整个 APP 的，一旦设置过一次之后就不需要再设置了。如果我们需要某个页面是白色，则需要在进入这个页面的时候通过该方式设置为白色，然后离开的时候再重新设置为黑色。它是和整个 APP 设置一一对应，有些时候方便，有些时候又是麻烦的。

### 方式二：设置只针对某个视图控制器

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;在 info.plist 文件中将 `View controller-based status bar appearance` 设置为YES，或者不设置，默认为 YES。如果将它设置为 NO，则该方案的设置方式不起作用。方案为：在 `UIViewController` 中覆写 `- (UIStatusBarStyle)preferredStatusBarStyle` 方法，返回 `UIStatusBarStyleLightContent` 或 `UIStatusBarStyleDefault` 即可。这种方案是比较灵活的，基本可以满足所有需求。如果你想要给某个视图控制器更改状态栏，就只要覆写这个视图控制器中的该方法就行了，它这个设置不影响其他的视图控制器，很是方便。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;这种方案还有关联的几个方法，如下：

1. - (UIStatusBarStyle)preferredStatusBarStyle;
2. - (UIViewController *)childViewControllerForStatusBarStyle;
3. - (void)setNeedsStatusBarAppearanceUpdate;

方法1: 它会在我们 `UIViewController` 显示的时候，主动调一次该方法，来改变状态栏；

方法2: 如果该 `UIViewController` 已经显示了，此时我通过代码想要更改状态栏（比如滑动 `UIScrollView` 来变更状态栏的展示颜色），可以通过方法 `setNeedsStatusBarAppearanceUpdate`（这个方法会通知系统去调用当前 `UIViewController` 的 `preferredStatusBarStyle` 方法，同 `UIView` 的 `setNeedsDisplay` 原理差不多，系统会在下次页面刷新时，调用重绘该 view） 

方法3: 

注意：当 `ViewController` 在 `UINavigationController` 中时，如果导航栏存在，则以 `UINavigationController` 中 `-(UIStatusBarStyle)preferredStatusBarStyle` 返回风格为标准；如果导航栏隐藏了，则以控制器中返回的风格为标准。

`childViewControllerForStatusBarStyle `接口返回值默认为 nil。当我们调用 `setNeedsStatusBarAppearanceUpdate` 时，系统会调用 `application.window` 的 `rootViewController` 的`preferredStatusBarStyle` 方法，我们的程序里一般都是用`UINavigationController` 做 `root` ，如果是这种情况，那我们自己的`UIViewController` 里的` preferredStatusBarStyle`根本不会被调用；这种情况下 `childViewControllerForStatusBarStyle` 就派上用场了， 我们要子类化一个 `UINavigationController` ，在这个子类里面重写`childViewControllerForStatusBarStyle` 方法，如下：

```
- (UIViewController *)childViewControllerForStatusBarStyle
{
    return self.topViewController;
}
```

代码写在 `UINavigationController` 的子类中，意思表示：你不要调用我自己(就是`UINavigationController`)的 `preferredStatusBarStyle ` 方法，而是去调用 `navigationController.topViewController` 的 `preferredStatusBarStyle` 方法，这样写的话，就能保证当前显示的`UIViewController`的`preferredStatusBarStyle`方法能正确设置状态栏颜色。

另外，有时我们的当前显示的 `UIViewController` 可能有多个`childViewController`，重写当前`UIViewController`的`childViewControllerForStatusBarStyle`方法，让`childViewController`的`preferredStatusBarStyle`生效(当前`UIViewController`的`preferredStatusBarStyle`就不会被调用了)。

总结来说，只要 `UIViewController` 重写的的`childViewControllerForStatusBarStyle` 方法返回值不是nil，那么，`UIViewController` 的 `preferredStatusBarStyle` 方法就不会被系统调用（默认情况下，`childViewControllerForStatusBarStyle` 返回nil，这个 `viewController` 的 `preferredStatusBarStyle` 被调用），系统会调用 `childViewControllerForStatusBarStyle` 方法返回的`UIViewController` 的 `preferredStatusBarStyle` 方法。方法三在我们上述解决的 bug 中就起到重要作用了。

参考链接：

[导航栏隐藏 && 导航栏错乱](http://www.jianshu.com/p/e4448c24d900)