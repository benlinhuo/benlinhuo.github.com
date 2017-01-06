---
layout:     post
title:      iOS framework 那些事儿
category: iOS
tags: [iOS]
description: 我们项目中经常会用到 framework，你有想过什么情况下使用、怎么使用、怎么制作 framework 等等事儿吗？
---

## framework、动态库、静态库

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;库就表示是一段编译好的二进制代码，加上头文件就可以供别人使用。为啥会用到？一是为了不让别人看到源码，只暴露头文件。二是可以减少编译的时间（因为库是已经编译好的二进制了），运行的时候只需要链接（Link），不需要编译。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;链接有两种方式：静态和动态，即对应的静态库和动态库。

1. 静态库（Linux 或者 Mac 下 .a 或者 Windows 下的 .lib），是指在编译的时候直接拷贝一份，复制到目标程序中，这样在运行时就不会有外部依赖，直接运行即可。缺点是使目标程序体积增大。

2. 动态库（Linux 下的 .so ，Mac 下的 .dylib，Windows 下的 .dll）。动态在编译时并不会被拷贝到目标程序中，目标程序只会存储指向动态库的引用。只有等程序运行时，动态库才会被真正加载进来。优点：不需要拷贝到目标程序中，不会影响目标程序的体积。缺点：依赖外部环境。

### iOS Framework

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;iOS 的 Framework，实际上是一种打包方式，将库的二进制文件、头文件和有关的资源文件打包到一起，方便管理和分发。 iOS 8之前，iOS 平台是不支持使用动态 Framework 的，开发者能使用的就只是系统提供的 UIKit.framework，Foundation.Framework等，这是从安全角度来说的。我们如果想要在 iOS 平台共享代码的话，唯一的选择就是打包成静态库 .a 文件，同时附上头文件（例如微信SDK）。不过这种方式比较麻烦。还是愿意使用 framework。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;iOS 8之后添加了动态库的支持，原因可能是 App Extension 的出现。App Extension 和 App 是两个分开的可执行文件，同时需要共享代码，这种情况下动态库的支持就是必不可少的了。但是这种动态 Framework 和系统的 UIKit.Framework 还是有很大区别。系统的 Framework 不需要拷贝到目标程序中，我们自己做出来的 Framework 哪怕是动态的，最后也还是要拷贝到 App 中（App 和 App Extension 的 Bundle 是共享的），因此苹果又把这种 Framework 称为 Embedded Framework。

#### App Extension 简单介绍

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;iOS 的应用程序都是沙盒机制，对于不同应用程序之间是不可以相互通信的。所以 App Extension 也是在一定程序上弥补了这个缺陷。它主要的作用是为了让用户在其他应用中也可以使用我们应用服务。比如用户可以在Today的widgets中查看应用展示的简略信息，而不用再进到我们的应用中，这将是一种全新的用户体验。iOS8 之后才有。

关于 App Extension 的介绍可见如下链接：

[http://www.cocoachina.com/industry/20140627/8960.html](http://www.cocoachina.com/industry/20140627/8960.html)

[http://www.cocoachina.com/ios/20140812/9366.html](http://www.cocoachina.com/ios/20140812/9366.html)


## framework 制作

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;我们制作的 framework 一般目的是为了隐藏源代码（例如不同公司之间的业务交流）或者不给修改源代码，这样可以将私密代码打包成 framework ，只暴露接口给别人使用即可，别人并不知道其中的细节。它有模拟器和真机之分，区别的主要是 framework 对 CPU 架构的支持。

**模拟器：** iphone4s-5：i386,     iphone5s-6plus: x86_64

**真机：** iphone3gs-4s：armv7，iphone5-5c：armv7s （静态库只要支持 armv7，就可以跑在 armv7s 的架构上），iphone5s-6plus：armv64

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;armv6，armv7，armv7s 是 ARM CPU 的不同指令集，原则是向下兼容的。当我们编译时选择不同选项（具体可见链接），可以编译成模拟器或者真机。


### 命令行查询 ARM 架构

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;命令查看支持的 ARM 架构（lipo -info xxx）。通过命令也可以将模拟器和真机合并成一个 framework （lipo -create xxx1  xxx2  -output  xxx）

![命令行查询实际操作](/assets/images/framework-arm.png)

制作 framework 其实挺简单的，Command + Shift + N 新建项目，选择 “Cocoa Touch Framework” 选项即可。详细可查看链接：

[http://www.jianshu.com/p/6c033c39884a](http://www.jianshu.com/p/6c033c39884a)


## framework 使用中的问题

### 1. Link binary with libraries 和 Embed Frameworks 区别

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;我们经常会把某一个 framework 既存放到 `Link binary with libraries`(将某个 framework 拖放到左侧的 bundle 中，它便会自动出现在该选项中)，又存放到 `Embed Frameworks`（这个需要将左侧 bundle 中的该 framework 拖动到该项目即可）。当然有些 framework 就只存在于 `Link binary with libraries`中。 界面可见下图：

![界面](/assets/images/framework-position.png)

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;二者的区别在 `stackoverflow` 上描述如下：

**Link binary with libraries：** Link frameworks and libraries with your project’s object files to produce a binary file. You can link a target’s source files against libraries in the target’s active SDK or against external libraries.

**Embed Frameworks：** You can create an embedded framework to share code between your app extension and its containing app.

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;对于要添加到项目中的framework，都需要添加到 "Link binary with libraries" 中。而对于 `Embed Framework`，是针对动态framework的，也就是说它希望在 App Extension 和 主App 之间共享这个 framework，因为iOS 自身的限制，所以动态的 framework 并不能像它自身特性一样，可以在任一应用之间共享之，最多只能在 App Extension 和 它的主 APP 共享。


### 2. 动态库framework，包含模拟器架构上传 AppStore 验证不通过

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;我们创建的 framework 一般都会有真机和模拟器两部分的架构一起，但是在上传 AppStore 的时候却告知不可含有模拟器部分的架构。提示信息大致如下：

```
1. "Unsupported Architecture. Your executable contains unsupported architecture '[x86_64, i386]'."

2. "Invalid segment Alignment. The App Binary at XXXX.app/Frameworks/XXXX.framework/XX does not have proper segment alignment. Try rebuilding the app with the latest xcode version." (即便使用最新Xcode也是不行）

3.  "The Binary is invalid. The encryption info in the LC_ENCRYPTION_INFO load command is either missing or invalid, or the binary is already encrypted. This binary does not seem to have been built with Apple's Linker."
```

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;因为平时开发需要模拟器，因此平时都是支持两个架构的，那这样在我们上传 AppStore 的时候就只能把模拟器的架构给删除出去。删除的方式有2种。

#### 方式一：脚本替换开发环境和上传appstore所使用的不同framework

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;思路很简单，准备好开发环境一套 framework（包含真机和模拟器架构），和上传appstore 使用的一套 framework（只有真机架构）。替换framework只是framework的内容变了，对于项目的配置文件来说没有任何变化，所以我们只需要写个脚本 `rm` 之前的 framework，`cp` 当前需要的framework 到正确的项目路径即可。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;这个方案的缺点是脚本的执行是手动的。每次打包上传appstore，需要记得把framework 替换成只有真机架构的，开发环境又需要记得替换成真机和模拟器架构都有的。但是如果你是用脚本打包便没有这个缺点了。

#### 方式二：run script

![run script 方式](/assets/images/runScript-BuildPhases.png)

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;如上图，我们添加了脚本之后，这个脚本的执行顺序是先 `Link`，再 "Run Script"。

![先 Link](/assets/images/framework-link.png)

![再 Run Script](/assets/images/framework-runscript-error.png)

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;如果我们把方式一种的脚本放在此处，让build 过程中自动执行该脚本。我们会发现报错（当只有真机架构，却用模拟器运行），提示架构不支持，原因是我们的脚本没起到作用。因为脚本是在 `run script` 时运行，而 `Link` 的时候就会使用 framework 链接了，该步骤在脚本执行之前就需要了。所以使用该脚本根本行不通。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;换个脚本，脚本的内容不是替换framework，而只是将framework 中的模拟器架构删除。如下脚本是网上看到的一段：

```
APP_PATH="${TARGET_BUILD_DIR}/${WRAPPER_NAME}"

# This script loops through the frameworks embedded in the application and
# removes unused architectures.
find "$APP_PATH" -name '*.framework' -type d | while read -r FRAMEWORK
do
    FRAMEWORK_EXECUTABLE_NAME=$(defaults read "$FRAMEWORK/Info.plist" CFBundleExecutable)
    FRAMEWORK_EXECUTABLE_PATH="$FRAMEWORK/$FRAMEWORK_EXECUTABLE_NAME"
    echo "Executable is $FRAMEWORK_EXECUTABLE_PATH"

    EXTRACTED_ARCHS=()

    for ARCH in $ARCHS
    do
        echo "Extracting $ARCH from $FRAMEWORK_EXECUTABLE_NAME"
        lipo -extract "$ARCH" "$FRAMEWORK_EXECUTABLE_PATH" -o "$FRAMEWORK_EXECUTABLE_PATH-$ARCH"
        EXTRACTED_ARCHS+=("$FRAMEWORK_EXECUTABLE_PATH-$ARCH")
    done

    echo "Merging extracted architectures: ${ARCHS}"
    lipo -o "$FRAMEWORK_EXECUTABLE_PATH-merged" -create "${EXTRACTED_ARCHS[@]}"
    rm "${EXTRACTED_ARCHS[@]}"

    echo "Replacing original executable with thinned version"
    rm "$FRAMEWORK_EXECUTABLE_PATH"
    mv "$FRAMEWORK_EXECUTABLE_PATH-merged" "$FRAMEWORK_EXECUTABLE_PATH"

done
``` 

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;自己尝试这段脚本是有点小问题，可能需要更改，主要是因为上线appstore不是自己负责，也没对应的账号权限，所以没有办法去验证打出来的包是否就把模拟器架构删除了。有兴趣的同学，可以尝试。


##### run script 脚本调试

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;当我们在脚本中 `echo "ssssss"`类似的代码，这样子的输出结果可以在哪儿看到呢？如下图：

![run script 调试](/assets/images/runscript-debug.png)


##### Pods 中的 run script 脚本

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;因为项目中看到如下内容：

![Pods-runScript](/assets/images/pods-framework.png)

所以一开始以为Pods中源码生成的 framework （./Pods/Products），都是通过脚本生成的。后来通过测试发现，其实在我们 `pod install`，就已经生成对应 framework了，上图中的 `[CP] Embed Pods Frameworks` 和 `[CP] Copy Pods Resources`即使被删除也对项目的运行没有影响，而且这部分的脚本都是 `pod install` 过程中自动生成的。















