---
layout:     post
title:      App URL Scheme and Universal Link
category: iOS
tags: [iOS]
description: 因业务项目的导流（分享的H5链接打开，可以到对应app的具体业务页面），或者多款项目app之间互导流量（app 之间相互跳转）。
---

### NSMachPort

在说到Runloop，我们经常会碰到一个东西：NSPort。它一般是运用在两个线程之间要互相通信。如下介绍的也就是个基础，我们可以看如下2个实例：

1. 案例一：主线程触发创建子线程，然后把获取的端口带给子线程，子线程也获取端口，然后利用之前传递过来的主线程端口，发送消息给子线程。

```
// ViewController.m
- (void)startThread {
    // 获取主线程port，这样子线程就可以通过该端口给主线程发送消息了
    NSPort *mainPort = [NSMachPort port];
    if (mainPort) {
        mainPort.delegate = self;
        // 把port加入runloop，接收port消息。如果runloop退出了，便监控不到发送来的消息
        [[NSRunLoop currentRunLoop] addPort:mainPort forMode:NSDefaultRunLoopMode];
        // 创建子线程
        [NSThread detachNewThreadSelector:@selector(createChildThreadWithPort:) toTarget: [ChildThreadSendMessage class] withObject:mainPort];
    }
}

#pragma mark - NSPortDelegate

- (void)handlePortMessage:(NSMessagePort*)message {
    NSLog(@"主__接到子线程传递的消息！");
}


// ChildThreadSendMessage.m
// 子线程创建
+ (void)createChildThreadWithPort:(NSPort *)mainPort {
    ChildThreadSendMessage *childThread = [ChildThreadSendMessage new];
    [childThread sendMessageToMain:mainPort];
    do {
        [[NSRunLoop currentRunLoop] runMode:NSDefaultRunLoopMode beforeDate:[NSDate distantFuture]];
    } while (![childThread shouldExit]);
}

- (void)sendMessageToMain:(NSPort*)mainPort {
    NSPort *childPort = [NSMachPort port];
    childPort.delegate = self;
    //将自己的port添加到runloop
    //作用1、防止runloop执行完毕之后退出
    //作用2、接收主线程发送过来的port消息
    [[NSRunLoop currentRunLoop] addPort:childPort forMode:NSDefaultRunLoopMode];
    // 发送消息到主线程
    NSMutableArray *array  =[[NSMutableArray alloc]initWithArray:@[@"1",@"2"]];

    [mainPort sendBeforeDate:[NSDate date] msgid:10 components:array from:childPort reserved:0];
}

#pragma mark - NSPortDelegate
- (void)handlePortMessage:(NSPortMessage *)message
{
    NSLog(@"子__接收到主线程消息");
}
```


2. 案例二：一般 Notification 在什么线程发送的，则在什么线程处理，比如我是在子线程执行 `[[NSNotificationCenter defaultCenter] postNotificationName:@"NotificationName" object:nil userInfo:nil];`，则执行的方法  `processNotification` 则在什么线程。（`[[NSNotificationCenter defaultCenter]
     addObserver:self
     selector:@selector(processNotification:)
     name:@"NotificationName"
     object:nil];`（和该行代码在什么线程执行无关） ）。所以我们可以考虑在子线程发送通知，然后处理方法中发现在子线程，则保存起来，然后发送给主线程处理（这个发送过程就可以通过NSMachPort）。

```
#import "SecondViewController.h"

@interface SecondViewController ()<NSMachPortDelegate>

@property (nonatomic, strong) NSMutableArray    *notificationsQueue;    //存储子线程发出的通知的队列
@property (nonatomic, strong) NSThread          *mainThread;            // 处理通知事件的预期线程
@property (nonatomic, strong) NSLock            *lock;                  // 用于对通知队列加锁的锁对象，避免线程冲突
@property (nonatomic, strong) NSMachPort        *mackPort;              // 用于向期望线程发送信号的通信端口

@end

@implementation SecondViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    self.view.backgroundColor = [UIColor whiteColor];
    
    //打印注册观察者的线程，此处也就是主线程
    NSLog(@"register notificaton thread = %@", [NSThread currentThread]);
    
    [self setUpThreadingSupport]; // 对相关的成员属性进行初始
    
    [[NSNotificationCenter defaultCenter]
     addObserver:self
     selector:@selector(processNotification:)
     name:@"NotificationName"
     object:nil];
    
    dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
        NSLog(@"post notificaton thread = %@", [NSThread currentThread]);
        [[NSNotificationCenter defaultCenter] postNotificationName:@"NotificationName" object:nil userInfo:nil];
    });
}


/**
 对相关的成员属性进行初始
 */
- (void) setUpThreadingSupport {
    if (self.notificationsQueue) {
        return;
    }
    self.notificationsQueue = [[NSMutableArray alloc] init];    //队列：用来暂存其他线程发出的通知
    self.lock = [[NSLock alloc] init];                          //负责栈操作的原子性
    self.mainThread = [NSThread currentThread];                 //记录处理通知的线程
    self.mackPort = [[NSMachPort alloc] init];                  //负责往处理通知的线程所对应的RunLoop中发送消息的
    [self.mackPort setDelegate:self];
    
    [[NSRunLoop currentRunLoop] addPort:self.mackPort           //将Mac Port添加到处理通知的线程中的RunLoop中
                                forMode:(__bridge NSString *)kCFRunLoopCommonModes];
}


/**
 从子线程收到Mach Port发出的消息后所执行的方法
 在该方法中从队列中获取子线程中发出的NSNotification
 然后使用当前线程来处理该通知
 
 RunLoop收到Mac Port发出的消息时所执行的回调方法。
 */
- (void)handleMachMessage:(void *)msg {
    
    NSLog(@"handle Mach Message thread = %@", [NSThread currentThread]);
    
    //在子线程中对notificationsQueue队列操作时，需要加锁，保持队列中数据的正确性
    [self.lock lock];
    
    //依次取出队列中所暂存的Notification，然后在当前线程中处理该通知
    while ([self.notificationsQueue count]) {
        NSNotification *notification = [self.notificationsQueue objectAtIndex:0];
        
        [self.notificationsQueue removeObjectAtIndex:0]; //取出队列中第一个值
        
        [self.lock unlock];
        [self processNotification:notification];    //处理从队列中取出的通知
        [self.lock lock];
        
    };
    
    [self.lock unlock];
}


- (void)processNotification:(NSNotification *)notification {
    
    if ([NSThread currentThread] == _mainThread) {
        //处理出队列中的通知
        NSLog(@"handle notification thread = %@", [NSThread currentThread]);
        NSLog(@"process notification");
        
    } else { //在子线程中收到通知后，将收到的通知放入到队列中存储，然后给主线程的RunLoop发送处理通知的消息
        NSLog(@"transfer notification thread = %@", [NSThread currentThread]);
        
        // Forward the notification to the correct thread.
        [self.lock lock];
        [self.notificationsQueue addObject:notification];    //将其他线程中发过来的通知不做处理，入队列暂存
        [self.lock unlock];
        
        //通过MacPort给处理通知的线程发送通知，使其处理队列中所暂存的队列
        [self.mackPort sendBeforeDate:[NSDate date]
                           components:nil
                                 from:nil
                             reserved:0];
        
    }
}

@end
```

以上2个案例的demo，可以查看[NSMachPort 实例](https://github.com/benlinhuo/BLOGExample/tree/master/RunloopMachPort)
































