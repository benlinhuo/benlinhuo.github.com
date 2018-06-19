---
layout:     post
title:      App URL Scheme and Universal Link
category: iOS
tags: [iOS]
description: 因业务项目的导流（分享的H5链接打开，可以到对应app的具体业务页面），或者多款项目app之间互导流量（app 之间相互跳转）。
---

## 实现技术简介

因业务项目的导流（分享的H5链接打开，可以到对应app的具体业务页面），或者多款项目app之间互导流量（app 之间相互跳转）

从理想角度来说，URL Scheme 和 Universal Link 这两种技术都可以实现上述的两种需求。但是二者之间还是有区别的。

1. URL Scheme 
	它是为了方便 app 之间相互调用而设计的，注册自己独一的URL Scheme即可。通过方法 `[[UIApplication sharedApplication] openURL:@"weixin://"];` 打开。也可以通过 Safari 浏览器打开 app，也可以传递参数，不过会有如下提示。
	![提示](/assets/images/image-tip.jpeg)

	它需要在项目中配置 URL Types，如下图：    
	![URL Types配置](/assets/images/urlTypes.jpg)
	
	这种方式的跳转是协议跳转。但是微信或者QQ不能打开第三方应用（即zaapp:// 无法跳转到众安app），是因为微信和QQ使用的是自己研发的QQ浏览器，它不遵循 URL Schemes 协议。
	

2. Universal Link

	这是 iOS 9 之后才支持。
	
	通用链接的深层链接特性：通过 `传统的HTTP链接` 来启动APP，使用相同的网址打开网站和APP。通过唯一的网址，可以链接一个特定的视图（具体业务需求页面），不需要特别的scheme，例如京东app分享一件衣服的详情到微信，然后微信链接打开，会直接跳转到京东app，且会自动跳转到对应的详情页面。


3. Meta 标签

	这也是一种实现上述目标的方式。

	meta 标签格式如下：   
	
	```
	 <meta charset="UTF-8" name="apple-itunes-app" content="app-id=1234567890, affiliate-data=myAffiliateData, app-argument=yourScheme://">
	```
	
	这样添加meta标签之后的网页，使用 safari 打开的时候，就会在顶部显示自己 app 的导航条。如果没有安装 app，点击可以跳转到 appstore 下载，如果已安装了就直接通过顶部的 meta 标签唤醒 app 了。借用别人图：
	
	![meta 标签](/assets/images/meta-tag.jpeg)
	

## Universal Link 配置

因 Universal Link 是一个通用链接，所以 iOS9 以上的用户，可以直接点击该链接，无缝重定向到一个 app 应用，而不需要通过 safari 浏览器打开跳转。如果用户没有安装这个app，因为它本身就是 http 协议的普通链接，所以会在 safari 浏览器中直接打开该链接指向的网页页面。

### 1.创建配置文件 apple-app-site-association

##### 创建文件名为 apple-app-site-association(必须为该文件名，不能带后缀名)，包含固定格式的 json 文件；

```
{     
"applinks": {
        "apps": [],
        "details": [
            {
                "appID": "teamID.bundleId”,  
                "paths": ["/deaplink","/wwdc/news/","*"]
            },
            {
                "appID": "ABCD1234.com.apple.wwdc",
                "paths": [ "*" ]
            }
        ]
    }
}

```

上述的 teamID 可以通过苹果开发者网站，找到 `Membership` 选项卡：

![Membership 选项卡](/assets/images/urlteamId.png)

##### appID

如果说teamID是xxx，bundleId是com.B.app ，则appID 为 xxx.com.B.app

##### paths

paths 选项配置，实际上是限制哪些路径可以唤醒 app，哪些不能唤醒 app。格式如下：

```
"paths": [ "/wwdc/news/", "NOT /videos/wwdc/2010/*", "/videos/wwdc/201?/*"]

```
1> 使用 `*` 配置，则表示整个网站都可以使用

2> 使用特定的URL，例如 `/wwdc/news/` 来指定某一个特殊的链接

3> 在特定URL后面加 `*`，例如 `/videos/wwdc/2015/*`，来指定网站的某一部分

4> 除了使用 `*` 来匹配任意字符，也可以使用 `?` 来匹配单个字符，可以在路径中结合这两个字符使用，例如 `/foo/*/bar/201?/mypage`

必须要注意：配置的paths路径，是区分大小写的。


#### 2.将该文件上传到自己的服务器上，存放路径可以是服务器的根目录，又或者是 `.well-known` 子目录；

1> 确保使用链接 `https://yourdomain.com/apple-app-site-association` 可以访问到该 json 文件，其中 `yourdomain.com` 是服务器域名，配置app `Associated Domains` 的时候也会用到该域名。如：`https://static.aibenlin.com/apple-app-site-association` 配置的，是可以正常访问的

2> 也可以使用[苹果的验证网站](https://search.developer.apple.com/appsearch-validation-tool/)，验证文件是否能被苹果请求到。如果是未上线的应用，使用验证网站时可能出现如下提示： 

![验证配置文件是否有效](/assets/images/verifyjson.png)

如上图出现提示，则表示 `apple-app-site-association` 配置正确；若出现 404 错误码提示，则为 `apple-app-site-association` 文件未上传成功，或者使用路径 `https://yourdomain.com/apple-app-site-association` 无法访问


#### 3.配置app，然后在app里添加代理方法

##### 创建 web 网页和app应用之间的关联 -> app IDs 配置 和 项目配置

a. 进入开发者网站，找到自己的bundleId，可以点击 edit 按钮，开启 `associate domains`，如图：

![配置 `associate domains` ](/assets/images/associatedomains.png)

b. 在项目的 `Capablities` 中开启 `Associated domains`，如图： 

![配置 `Capablities ` ](/assets/images/capablitiesDomain.png)

上图中的 `domains` 可以添加多个，但前缀必须是 `applinks:`，`applinks:` 后为服务器的域名，同上述请求链接 `https://yourdomain.com/apple-app-site-association` 中域名 `yourdomain.com`。比如自己配置的链接： `applinks:static.aibenlin.com`

c. 代码接收 Universal Links 唤醒

```
// 使用 Universal Links 唤醒 app，使用该方法接收 url 做对应处理

-(BOOL)application:(UIApplication *)application continueUserActivity:(NSUserActivity *)userActivity restorationHandler:(void (^)(NSArray * _Nullable))restorationHandler{

    NSLog(@"userActivity : %@",userActivity.webpageURL.description);
    return YES;
}
```

#### 4.验证上述的配置

快捷验证：使用上述新配置的app证书，然后打个最新包安装到手机上；在备忘录中输入 `https://yourdomain.com/apple-app-site-association`，若出现下图提示，则表示配置成功

![验证配置 ](/assets/images/verifyConfig.png)

上述验证成功，表示请求 `apple-app-site-association` 文件成功，且用户可以使用 `Universal Links` 唤醒 app 了。

如果已经配置过 Universal Links ，则用户在第一次安装 app 时，苹果会发送一个请求，请求你服务器上的 `apple-app-site-association`。



## 总结

Unversal Link 工作流程图（来源于网上总结）：

![Unversal Link 工作流程图](/assets/images/universalLinkParse.png)


### 注意点

1) 使用抓包软件可以看到，只有初次安装app时才会去请求 `apple-app-site-association` ；

2) Universal Links 之所以可行，是因为运行在 iOS 系统上的所有app，当他们使用苹果的 `webKit` 打开某一个链接，`webKit` 会优先拦截到，然后通过配置文件 `apple-app-site-association` 的配置规则，来决定该链接是跳转到 app ，还是直接浏览器打开对应页面。

3) 通用链接 和 调用通用链接的网页 不要使用同一个域名，即如果通用链接域名为 `www.mydomain.com`，则通用链接所处的网页域名就不能是 `www.mydomain.com`

4) 其实上述的方案中，无论URI Scheme 还是 Universal Link 都无法解决一个问题，就是如果设备上该app还没有安装，保留住此时用户停留的上下文。例如，利用Universal Link，在没有安装app的情况下，iOS能够重定位到app store去引导用户去下载安装这个app，但是在安装之后，app只能打开首页，也就是说丢失了用户在点击跳转进入app之前的那个页面。这种问题，个人觉得解决方案：1. 在跳转到安装之前，利用服务端记录，然后在第一次进入app的时候，通过服务端数据跳转不同页面，这种方案依赖网络，可行性不高；2. 利用剪切板，在跳转下载页之前，按照约定把跳转链接放置剪切板，下载后进入，第一次打开app的时候，从剪切板获取对应数据，进行相应跳转。


在2017年初，测试 Universal Link ，微信中打开时可以的，但是2018年6月2号测试，发现微信打开不了 Universal Link 链接了，不过 QQ 可以。但尝试京东 APP 是可以的，查看资料，发现微信有做白名单机制，只有白名单的 APP 才可以通过 Universal Link 唤醒第三方app。

![Unversal Link QQ 和 微信的测试](/assets/images/universalLinkExample.mp4)

<iframe height="498" width="510" src="/assets/images/universalLinkExample.mp4"></iframe>


## 魔窗mLink

魔窗的深层链接实现方案就是使用了 `Universal Link`（iOS 9+）和 `URL Scheme` 。它通过 URL Scheme 和 Universal link 配置唤醒app的代码

	
```
//1. iOS9以下，通过url scheme来唤起app
- (BOOL)application:(UIApplication *)application openURL:(NSURL *)url sourceApplication:(NSString *)sourceApplication annotation:(id)annotation
{
    //必写
    [MWApi routeMLink:url];
    return YES;
}

//2. iOS9+，通过url scheme来唤起app
- (BOOL)application:(UIApplication *)app openURL:(NSURL *)url options:(nonnull NSDictionary *)options
{
    //必写
 	[MWApi routeMLink:url];
    return YES;
}

//3. 通过universal link来唤起app
- (BOOL)application:(UIApplication *)application continueUserActivity:(NSUserActivity *)userActivity restorationHandler:(void (^)(NSArray * _Nullable))restorationHandler
{
    //必写
    return [MWApi continueUserActivity:userActivity];
}
	
```

注意：当需要在openURL这个方法中处理第三方回调的时候（比如支付宝回调，微信回调等，比如分享到微信，从微信跳回来的时候，会执行方法 1 和 2），请注意区分，比如：

```
if(支付宝回调) return 支付宝回调处理逻辑;
else if(微信回调) return 微信回调处理逻辑;
else if(其他第三方回调) return 其他第三方回调处理逻辑;
else [MWApi routeMLink:url]; return YES;

```



参考链接：

[iOS9 Universal Links踩坑之旅，移动应用之deeplink唤醒app](https://www.jianshu.com/p/77b530f0c67b)

[iOS 9学习系列：打通 iOS 9 的通用链接（Universal Links）](http://www.cocoachina.com/ios/20150902/13321.html)

[iOS Universal Links & URL Scheme](https://www.jianshu.com/p/53588cf8dbc8?nomobile=yes)

[mLink iOS SDK 集成文档](http://www.magicwindow.cn/doc/mlink-sdk-ios.html#begin-start/senior-set)

[理解Deep Link & URI Schemes & Universal Link & App Link](https://www.jianshu.com/p/909999e398e6?utm_campaign=hugo&utm_medium=reader_share&utm_content=note&utm_source=weixin-friends&from=groupmessage&isappinstalled=0)
































