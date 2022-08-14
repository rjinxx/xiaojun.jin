---
title: 知其然亦知其所以然-NSOperation并发编程
subtitle: ""

# Summary for listings and search engines
summary: 对于iOS的并发编程, 用的最普遍的就是GCD了, GCD结合Block可以so easy的实现多线程并发编程. 但如果你看一些诸如AFNetworking, SDWebImage的源码, 你会发现它们使用的都是NSOperation。

# Link this post with a project
projects: []

# Date published
date: "2019-10-08T00:00:00Z"

# Date updated
lastmod: "2019-10-08T00:00:00Z"

# Is this an unpublished draft?
draft: false

# Show this page in the Featured widget?
featured: false

# Featured image
# Place an image named `featured.jpg/png` in this page's folder and customize its options here.
image:
  caption: ''
  focal_point: ""
  placement: 2
  preview_only: false

authors:
- 金小俊

tags:
- iOS
- NSOperation

categories:
- iOS
- NSOperation
---

对于iOS的并发编程, 用的最普遍的就是GCD了, GCD结合Block可以so easy的实现多线程并发编程. 但如果你看一些诸如AFNetworking, SDWebImage的源码, 你会发现它们使用的都是NSOperation。我们先来简单PK下GCD和NSOperation(当然这里也包括NSOperationQueue).

1). NSOperation是基于GCD之上的更高一层封装, 拥有更多的API(e.g. suspend, resume, cancel等等).
2). 在NSOperationQueue中, 可以指定各个NSOperation之间的依赖关系.
3). 用KVO可以方便的监测NSOperation的状态(isExecuted, isFinished, isCancelled).
4). 更高的可定制能力, 你可以继承NSOperation实现可复用的逻辑模块.

Soga, 原来NSOperation这么拽! Apple官方文档和网络上有很多NSOperation的资料, 但大部分都是很书面化的解释(臣妾看不懂啊%>_<%), 看着看着就云深不知处了. 所以这篇文章我会以灰常通俗的方式来解释NSOperation的并发编程. Okay, let's go!

###并发编程的几个概念

并发编程简单来说就是让CPU在同一时间运行多个任务. 这里面有几个容易混淆的[概念](https://www.objc.io/issues/2-concurrency/concurrency-apis-and-pitfalls/), 我们先来一个个的梳理下:

1). 串行(Serial) VS. 并行(Concurrent)
串行和并行描述的是任务和任务之间的执行方式. 串行是任务A执行完了任务B才能执行, 它们俩只能顺序执行. 并行则是任务A和任务B可以同时执行. 

2). 同步(Synchronous) VS. 异步(Asynchronous)
同步和异步描述的其实就是函数什么时候返回. 比如用来下载图片的函数A: {download image}, 同步函数只有在image下载结束之后才返回, 下载的这段时间函数A只能搬个小板凳在那儿坐等... 而异步函数, 立即返回. 图片会去下载, 但函数A不会去等它完成. So, 异步函数不会堵塞当前线程去执行下一个函数!

3). 并发(Concurrency) VS. 并行(Parallelism)
这个更容易混淆了, 先用[Ray大神](http://www.raywenderlich.com/79149/grand-central-dispatch-tutorial-swift-part-1)的示意图和说明来解释一下: 并发是程序的属性(property of the program), 而并行是计算机的属性(property of the machine).

![Concurrency_vs_Parallelism.png](http://upload-images.jianshu.io/upload_images/268805-e22356a9458b1c5f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

还是很抽象? 那我再来解释一下, 并行和并发都是用来让不同的任务可以"同时执行", 只是并发是伪同时, 而并行是真同时. 假设你有任务T1和任务T2(这里的任务可以是进程也可以是线程):

a. 首先如果你的CPU是单核的, 为了实现"同时"执行T1和T2, 那只能分时执行, CPU执行一会儿T1后马上再去执行T2, 切换的速度非常快(这里的切换也是需要消耗资源的, context switch), 以至于你以为T1和T2是同时执行了(但其实同一时刻只有一个任务占有着CPU).

b. 如果你是多核CPU, 那么恭喜你, 你可以真正同时执行T1和T2了, 在同一时刻CPU的核心core1执行着T1, 然后core2执行着T2, great!

> 其实我们平常说的并发编程包括狭义上的"并行"和"并发", 你不能保证你的代码会被并行执行, 但你可以以并发的方式设计你的代码. 系统会判断在某一个时刻是否有可用的core(多核CPU核心), 如果有就并行(parallelism)执行, 否则就用context switch来分时并发(concurrency)执行. 最后再以Ray大神的话结尾: Parallelism requires Concurrency, but Concurrency does not guarantee Parallelism!

###并发吧, NSOperation!
NSOperation可以自己独立执行(直接调用[operation start]), 也可以放到NSOperationQueue里面执行, 这两种情况下是否并发执行是不同的. 我们先来看看NSOperation独立执行的并发情况.

#####1. 独立执行的NSOperation
NSOperation默认是非并发的(non-concurrent), 也就说如果你把operation放到某个线程执行, 它会一直block住该线程, 直到operation finished. 对于非并发的operation你只需要继承NSOperation, 然后重写main()方法就妥妥滴了, 比如我们用非并发的operation来实现一个下载需求:
```
@implementation YourOperation 

- (void)main 
{
    @autoreleasepool {
        
        if (self.isCancelled) return;
        
        NSData *imageData = [[NSData alloc] initWithContentsOfURL:imageURL];
        
        if (self.isCancelled) { imageData = nil; return; }
        
        if (imageData) {
            UIImage *downloadedImage = [UIImage imageWithData:imageData];
        }
        
        imageData = nil;
        
        if (self.isCancelled) return;
        
        [self.delegate performSelectorOnMainThread:@selector(imageDownloaderDidFinish:)                                                                  
                                        withObject:downloadedImage
                                     waitUntilDone:NO];
    }
}

@end
```
>由于NSOperation是可以cancel的, 所以你需要在operation程序内部执行过程中判断当前operation是否已经被cancel了(isCancelled). 如果已经被cancel那就不往下执行了. 当你在外面调用[operation cancel]后, isCancelled会被置为YES.

NSOperation有三个状态量`isCancelled`, `isExecuting`和`isFinished`. isCancelled上面解释过. main函数执行完成后, isExecuting会被置为NO, 而isFinished则被置为YES.

那肿么实现并发(concurrent)的NSOperation呢? 也很简单:
1). 重写isConcurrent函数, 返回YES, 这个告诉系统各单位注意了我这个operation是要并发的.
2). 重写start()函数. 
3). 重写isExecuting和isFinished函数

>为什么在并发情况下需要自己来设定isExecuting和isFinished这两个状态量呢? 因为在并发情况下系统不知道operation什么时候finished, operation里面的task一般来说是异步执行的, 也就是start函数返回了operation不一定就是finish了, 这个你自己来控制, 你什么时候将isFinished置为YES(发送相应的KVO消息), operation就什么时候完成了. Got it? Good.

还是上面那个下载的例子, 我们用并发的方式来实现:

```
- (BOOL)isConcurrent {
    return YES;
}

- (void)start 
{
    [self willChangeValueForKey:@"isExecuting"];
    _isExecuting = YES;
    [self didChangeValueForKey:@"isExecuting"];

    NSURLRequest * request = [NSURLRequest requestWithURL:imageURL];
    _connection = [[NSURLConnection alloc] initWithRequest:request
                                                  delegate:self];
    if (_connection == nil) [self finish];
}

- (void)finish
{
    self.connection = nil;
    
    [self willChangeValueForKey:@"isExecuting"];
    [self willChangeValueForKey:@"isFinished"];

    _isExecuting = NO;
    _isFinished = YES;

    [self didChangeValueForKey:@"isExecuting"];
    [self didChangeValueForKey:@"isFinished"];
}

#pragma mark - NSURLConnection delegate
- (void)connection:(NSURLConnection *)connection didReceiveResponse:(NSURLResponse *)response {
    // to do something...
}

- (void)connection:(NSURLConnection *)connection didReceiveData:(NSData *)data {
    // to do something...
}

- (void)connectionDidFinishLoading:(NSURLConnection *)connection {
    [self finish];
}

- (void)connection:(NSURLConnection *)connection didFailWithError:(NSError *)error {
    [self finish];
}

@end
```

Wow, 并行的operation好像有那么点意思了. 这里面还有几点需要mark一下:
a). operation的executing和finished状态量需要用willChangeValueForKey/didChangeValueForKey来触发KVO消息.
b). 在调用完NSURLConnection之后start函数就返回了, 后面就坐等connection的回调了.
c). 在connection的didFinish或didFail回调里面设置operation的finish状态, 告诉系统operation执行完毕了.

如果你是在主线程调用的这个并发的operation, 那一切都是非常的perfect, 就算你当前在操作UI也不影响operation的下载操作. BUT, 如果你是在子线程调用的, 或者把operation加到了非main queue, 那么问题来了, 你会发现这货的NSURLConnection delegate不走了, what's going on here? 要解释这个问题就要请出另外一个武林高手NSRunLoop, Okay, 下面进入NSRunLoop的show time.

#####Hey, NSRunLoop你是神马东东?
关于NSRunLoop推荐看一下[孙源@sunnnyxx](http://v.youku.com/v_show/id_XODgxODkzODI0.html)的分享视频. 其实从字面上就可以看出来, RunLoop就是跑圈, 保证程序一直在执行. App运行起来之后, 即使你什么都不做, 放在那儿它也不会退出, 而是一直在"跑圈", 这就是RunLoop干的事. 主线程会自动创建一个RunLoop来保证程序一直运行. 但子线程默认不创建NSRunLoop, 所以子线程的任务一旦返回, 线程就over了.

上面的并发operation当start函数返回后子线程就退出了, 当NSURLConnection的delegate回调时, 线程已经木有了, 所以你也就收不到回调了. 为了保证子线程持续live(等待connection回调), 你需要在子线程中加入RunLoop, 来保证它不会被kill掉.

RunLoop在某一时刻只能在一种模式下运行, 更换模式时需要暂停当前的Loop, 然后重启新的Loop. RunLoop主要有下面几个模式:

* NSDefalutRunLoopMode    : 默认Mode, 通常主线程在这个模式下运行
* UITrackingRunLoopMode   : 滑动ScrollView是会切换到这个模式
* NSRunLoopCommonModes: 包括上面两个模式

> 这边需要特别注意的是, 在滑动ScrollView的情况下, 系统会自动把RunLoop模式切换成UITrackingRunLoopMode来保证ScrollView的流畅性.

```
[NSTimer scheduledTimerWithTimeInterval:1.f
                                 target:self
                               selector:@selector(timerAction:)   
                               userInfo:nil
                                reports:YES];
```
当你在滑动ScrollView的时候上面的timer会失效, 原因是Timer是默认加在NSDefalutRunLoopMode上的, 而滑动ScrollView后系统把RunLoop切换为UITrackingRunLoopMode, 所以timer就不会执行了. 解决方法是把该Timer加到NSRunLoopCommonModes下, 这样即使滑动ScrollView也不会影响timer了.
```
[[NSRunLoop currentRunLoop] addTimer:timer forMode:NSRunLoopCommonModes];
```
另外还有一个trick是当tableview的cell从网络异步加载图片, 加载完成后在主线程刷新显示图片, 这时滑动tableview会造成卡顿. 通常的思路是tableview滑动的时候延迟加载图片, 等停止滑动时再显示图片. 这里我们可以通过RunLoop来实现.
```
[self.cellImageView performSelector:@sector(setImage:)
                         withObject:downloadedImage
                         afterDelay:0
                            inModes:@[NSDefaultRunLoopMode]];
```
当NSRunLoop为NSDefaultRunLoopMode的时候tableview肯定停止滑动了, why? 因为如果还在滑动中, RunLoop的mode应该是UITrackingRunLoopMode.

好了, 既然我们已经了解RunLoop的东东了, 我们可以回过头来解决上面子线程并发NSOperation下NSURLConnection的Delegate不走的问题, 各位童鞋且继续往下看^_^

#####呼叫NSURLConnection的异步回调
现在解决方案已经很清晰了, 就是利用RunLoop来监督线程, 让它一直等待delegate的回调. 上面已经说到Main Thread是默认创建了一个RunLoop的, 所以我们的[Option 1](http://www.dribin.org/dave/blog/archives/2009/05/05/concurrent_operations/)是让start函数在主线程运行(即使[operation start]是在子线程调用的).
```
- (void)start 
{
    if (![NSThread isMainThread]) {
        [self performSelectorOnMainThread:@selector(start)
                               withObject:nil
                            waitUntilDone:NO];
        return;
    }
    // set up NSURLConnection...
}
```
或者这样:
```
- (void)start
{
    [[NSOperationQueue mainQueue] addOperationWithBlock:^{
        self.connection = [NSURLConnection connectionWithRequest:self.request delegate:self];
    }];
}
```
这样我们可以简单直接的使用main run loop, 因为数据delivery是非常快滴. 然后我们就可以将处理incoming data的操作放到子线程去...

Option 2是让operation的start函数在子线程运行, 但是我们为它创建一个RunLoop. 然后把URL connection schedule到上面去. 我们先来瞅瞅AFNetworking是怎么做滴:
```
+ (void)networkRequestThreadEntryPoint:(id)__unused object 
{
    @autoreleasepool {
        [[NSThread currentThread] setName:@"AFNetworking"];
        NSRunLoop *runLoop = [NSRunLoop currentRunLoop];
        [runLoop addPort:[NSMachPort port] forMode:NSDefaultRunLoopMode];
        [runLoop run];
    }
}

+ (NSThread *)networkRequestThread 
{
    static NSThread *_networkRequestThread = nil;
    static dispatch_once_t oncePredicate;
    dispatch_once(&oncePredicate, ^{
        _networkRequestThread = [[NSThread alloc] initWithTarget:self selector:@selector(networkRequestThreadEntryPoint:) object:nil];
        [_networkRequestThread start];
    });
    return _networkRequestThread;
}

- (void)start 
{
    [self.lock lock];
    if ([self isCancelled]) {
        [self performSelector:@selector(cancelConnection) onThread:[[self class] networkRequestThread] withObject:nil waitUntilDone:NO modes:[self.runLoopModes allObjects]];
    } else if ([self isReady]) {
        self.state = AFOperationExecutingState;
        [self performSelector:@selector(operationDidStart) onThread:[[self class] networkRequestThread] withObject:nil waitUntilDone:NO modes:[self.runLoopModes allObjects]];
    }
    [self.lock unlock];
}
```
AFNetworking创建了一个新的子线程(在子线程中调用NSRunLoop *runloop = [NSRunLoop currentRunLoop]; 获取RunLoop对象的时候, 就会创建RunLoop), 然后把它加到RunLoop里面来保证它一直运行.

> 这边我们可以简单的判断下当前start()的线程是子线程还是主线程, 如果是子线程则调用[NSRunLoop currentRunLoop]创新RunLoop, 否则就直接调用[NSRunLoop mainRunLoop], 当然在主线程下就没必要调用[runLoop run]了, 因为它本来就是一直run的.

P.S. 我们还可以使用CFRunLoop来启动和停止RunLoop, 像下面这样:
```
[self.connection scheduleInRunLoop:[NSRunLoop currentRunLoop]
                           forMode:NSRunLoopCommonModes];
CFRunLoopRun();
```
等到该Operation结束的时候, 一定要记得调用CFRunLoopStop()停止当前线程的RunLoop, 让当前线程在operation finished之后可以退出.

#####2. NSOperationQueue里面执行NSOperation
NSOpertion可以add到NSOperationQueue里面让Queue来触发其执行, 一旦NSOperation被add到Queue里面那么我们就不care它自身是不是并发设计的了, 因为被add到Queue里面的operation必定是并发的. 而且我们可以设置Queue的maxConcurrentOperationCount来指定最大的并发数(也就是几个operation可以同时被执行, 如果这个值设为1, 那这个Queue就是串行队列了).

为嘛添加到Queue里面的operation一定会是并发执行的呢? Queue会为每一个add到队列里面的operation创建一个线程来运行其start函数, 这样每个start都分布在不同的线程里面来实现operation们的并发执行.

重要的事情再强调一遍: 我们这边所说的并发都是指NSOperation之间的并发(多个operation同时执行), 如果maxConcurrentOperationCount设置为1或者把operation放到[NSOperationQueue mainQueue]里面执行, 那它们只会顺序(Serial)执行, 当然就不可能并发了.

> [NSOperationQueue mainQueue]返回的主队列, 这个队列里面任务都是在主线程执行的(当然如果你像AFNetworking一样在start函数创建子线程了, 那就不是在主线程执行了), 而且它会忽略一切设置让你的任务顺序的非并发的执行, 所以如果你把NSOperation放到mainQueue里面了, 那你就放弃吧, 不管你怎么折腾, 它是绝对不会并发滴. 当然, 如果是[[NSOperationQueue alloc] init]那就是子队列(子线程)了.

那...那不对呀, 如果我在子线程调用[operation start]函数, 或者把operation放到非MainQueue里面执行, 但是在operation的内部把start抛到主线程来执行(利用主线程的main run loop), 那多个operation其实不都是要在主线程执行的么, 这样还能并发? Luckily, 仍然是并发执行的(其实我想说的是那必须能并发啊...哈哈).

我们可以先来看看单线程和多线程下的各个任务(task)的并发执行[示意图](http://codewala.net/2015/07/29/concurrency-vs-multi-threading-vs-asynchronous-programming-explained/):
![concurrent-single thread.png](http://upload-images.jianshu.io/upload_images/268805-9352ade9528a6f4c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

Yes! 和上面讨论狭义并发(Concurency)和并行(Parallelism)概念时的理解是一样的, 在单线程情况下(也就是mainQueue的主线程), 各个任务(在我们这里就是一个个的NSOperation)可以通过分时来实现伪并行(Parallelism)执行.

![concurrent-mutlithreaded.png](http://upload-images.jianshu.io/upload_images/268805-a80aa917da88f87f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

而在多线程情况下, 多个线程同时执行不同的任务(各个任务也会不停的切换线程)实现task的并发执行.

另外, 我们在往Queue里面添加operation的时候可以指定它们的依赖关系, 比如[operationB addDependency:operationA], 那么operationB会在operationA执行完毕之后才会执行. 还记得这边"执行完毕(isFinished)"的概念吗? 在并发情况下这个状态量是由你自己设定的, 比如operationA是用来异步下载一张图片, 那么只有图片下载完成之后或者超过timeout下载失败之后, isFinished状态量被标记为YES, 这时Queue才会从队列里面移除operationA, 并启动operationB. 是不是很cool? O(∩_∩)O~~

###NSOperation实验课

下面我们进入实验课啦, 要想真正了解某个东东, 还是需要打开Xcode, 写上几行代码, 然后Commard+R. 为了帮Apple提升Xcode的使用率:-D, 我会给出几个case, 童鞋们可以自己编写test code来验证:

1). 创建两个operation, 然后直接[operation start], 在NSOperation并发设计和非并发设计的情况下, 查看这两个operation是否同时执行了(最简单的打log看是不是交替打印).
2). 在主线程和子线程下分别调用[operation start], 看看执行情况. 
3). 创建operation并放到NSOperationQueue里面执行, 分别看看mainQueue和非mainQueue下的执行情况.
4). maxConcurrentOperationCount设置后的执行情况.
5). 试试NSOperation的依赖关系设置, [operationB addDependency:operationA].
6). 写个完整的demo吧, 比如简单的HTTP Downloader.

最后送上干货Demo, [RJHTTPDownloader](https://github.com/RylanJIN/RJHTTPDownloader), 用NSOperation实现的一个下载类. 有的童鞋肯定会说用AFNetwroking就可以了, 为嘛要自己去写呢? 这个嘛, 偶是觉得别人的代码再怎么看和用都不是你的, 自己动手写的才真正belongs to you! 而且这也不算是重复造轮子, 只是学习轮子是怎么构造的, 这样一步一步的慢慢积累, 总有一天我们也能写出像AFNetworking这样的代码! 共勉.