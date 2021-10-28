---
layout:     post
title:      iOS组件化搭建私有库(gitlab)
category: iOS
tags: [iOS]
description: 详解整理了搭建私有库的教程，直接用这种方式创建的库可以自带Example
---

# iOS组件化搭建私有库(gitlab)

## 1、创建索引库

### 1.1 首先检查当前电脑的索引库

```c++
pod repo
```

### 1.2 在gitlab上创建一个新的库，这个库用来保存私有库的podspec文件，所以我们一般起名字最好是 xxxSpec用以区分这个库的作用（如zhonganinfo-za_spec）

### 1.3创建本地索引库，然后将其于刚才创建的远程索引库相关联，<font color=red>注意！！！</font>此时的远程索引库是空的！空的！空的！但是必须得有master分支，所以你可以添加一个readme文件。

```c++
pod repo add xxxSpec(如：zhonganinfo-za_spec) 刚才创建的远程索引库的gitlab的地址(如：git@git.zhonganinfo.com:chenliqun/zhonganinfo-za_spec.git)
```

## 2、开始创建本地私有库
### 2.1 创建本地私有库（<font color=red>注意！这个库是存代码的，不要和刚才的索引库混淆了！！</font>）

```c++
pod lib create 私有库名称(如：ZAINetWorking)
```

### 2.2 然后在更新一下这个工程的pod库（切换到Example目录）

```c++
pod install
```

### 2.3 编写podspec文件里面的内容如下图

![img] (podspec编写.png)
ps：这里要注意两点！！

<1> s.homepage需要设置刚创建的私有代码仓库的地址, 不是私有索引库的地址！！！

<2> s.source 需要设置的是私有代码仓库的源地址(选择使用HTTPS地址)！！！

## 3、将私有库push到远程仓库
1）在gitlab上创建远程私有库。注意！！！这个库是存远程私有库代码的，不要跟远程索引库混淆啦！！！

2）将本地私有库推送到远程私有库

```c++
git status -- 查看当前git存了什么文件 
git add . -- 将所有文件缓存到待提交文件区域
git commit -m "上传工程" -- 提交文件，写上备注
git remote add origin 远程仓库地址 -- 添加要推送的远程仓库地址
git push -u origin master -- 将代码推送到远程仓库的master分支
```
## 4进行本地校验
1）将推送上去的文件，在本地进行校验一下，注意！！！ 这里坑就来啦！！！
一定要看你的私有库是否又依赖了其他的私有库，我最开始依赖了其他的私有库，所以在这个阶段怎么也验证不过去

2）最开始的验证命令是这样的( <font color=red>需要切换到xxx.podspec所在的目录</font>)

```c++
pod lib lint --allow-warnings
```
3）因为我依赖了其他的第三方库，所以我必须要将其他第三方库的索引库地址也得写上，就变成了这个样子

```c++
pod lib lint --sources="cocoapods库地址,私有库远程地址" --allow-warnings
```
```c++
例如：pod lib lint --sources="git@git.zhonganinfo.com:chenliqun/zhonganinfo-za_spec.git" --use-libraries --allow-warnings
```
4）但是这个第三方私有库又依赖了其他的库，所以还要对这个命令进行加工，之后变成了这个样子

```c++
pod lib lint --sources="cocoapods库地址,私有库远程地址" --use-libraries --allow-warnings
（例如：私有库依赖了AFNetworking，则这样：pod lib lint --sources="https://github.com/CocoaPods/Specs" --allow-warnings）
```
时间有一点长，得慢慢等

## 5.打Tag
1）验证通过之后，别忘了添加tag，这里添加的tag要跟刚才在spec文件里面写的版本号要一致，要一致，要一致！！！

```c++
git tag 版本号（要跟spec文件里面写的版本号一致）
git push --tags
```

# 6.远程校验
1）tag打上去之后，进行远程校验，其实和本地校验一样，本地如何校验通过的，远程只需要把lib字段改成spec就可以了，例如我的

```c++
pod spec lint --sources="cocoapods库地址,私有库远程地址" --use-libraries --allow-warnings
（例如：私有库依赖了AFNetworking，则这样：pod spec lint --sources="https://github.com/CocoaPods/Specs" --allow-warnings）

例如我的：
pod spec lint --sources="https://github.com/CocoaPods/Specs.git,git@ocean.wyzecam.com:wyze-app/wyzespec.git" --use-libraries --allow-warnings
```

# 7.将spec文件推送到最开始创建的索引库
1）所有验证通过之后，要将spec文件推送到最开始创建的远程索引库当中

```c++
pod repo push xxxSpec（本地索引库的名称，如：zhonganinfo-za_spec）xxx.podspec（私有库，如：ZAINetWorking.podspec ）
```

2)  这个时候验证一下你的私有库

```c++
pod repo update -- 先更新一下pod库，不然找不到你刚上传的私有库
（有报错的话可以试下：pod repo update --verbose）
pod search 私有库
```
