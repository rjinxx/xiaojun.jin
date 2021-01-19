---
title: 小红点解决方案思路分析
subtitle: ""

# Summary for listings and search engines
summary: 小红点(消息推送提醒)在现今的各个App中几乎无处不在，特别是内容的更新日渐频繁，大量的小红点被投放在各个业务入口。一般来说，小红点主要有三个应用。

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
- 移动开发
- 未读内容提示

categories:
- 移动开发
- 未读内容提示
---

小红点(消息推送提醒)在现今的各个App中几乎无处不在，特别是内容的更新日渐频繁，大量的小红点被投放在各个业务入口。一般来说，小红点主要有三个应用[场景](https://mp.weixin.qq.com/s?__biz=MTEwNTM0ODI0MQ==&mid=2653434543&idx=1&sn=8606f9b65db5cd272721d7868b4d662d):

- App有新添加的功能提醒用户使用
- 某一个已有的模块有功能上的更新
- 功能有内容的变化或业务上的提醒

常见的比如下图所示的QQ消息提示(红点为消息数目的提示), 朋友圈的新回复，店铺上架新品，最新优惠活动提醒等等。

![red-dot.png](http://upload-images.jianshu.io/upload_images/268805-d7fe08a12ec3b26d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/450)

###思路分析

通常情况下，小红点不是孤立使用的，一项功能或业务的运营涉及多个层级多个入口，所以小红点需要有清晰的路径导向，而且包含路径树的概念，父路径的小红点为子路径小红点的并集。其次就是小红点的具体显示，以及显示的具体样式。因此，总结一下后可以把小红点的功能模块归纳为两大块: 小红点路径监测+事件分发和小红点的UI显示。

#####小红点路径监测+事件分发

小红点所支持的路径格式设计为`root.xx.xx`, 小红点原则是父节点的小红点为子节点的小红点并集。`root`为默认的根路径。如下图所示, `root.first`为子路径, `root.second`为同级子路径。在纯红点模式下, `root`的小红点显示为`root.first, root.second`和`root.third`的并集，同理在数字显示模式下, `root`的badge数量为`root.first`, `root.second`和`root.third`的badge数量之和。而`root.first`的badge数量则又为`root.first.firstA`和`root.first.firstB`的和。

![path.png](http://upload-images.jianshu.io/upload_images/268805-22071ed84f5a5686.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

小红点的路径监测则是需要提供类似系统KVO的一个Observer, 用来观察路径所对应的小红点的变化，并且当子路径的红点发生变化是需要逐层分发到每一个父路径。当任意子路径有红点触发事件时，父路径也需显示红点。而当所有子路径的红点事情都清除后，父路径的红点才能清除。

总结一下，小红点路径监测需要实现下面的接口:
```
- (void)observePath:(NSString *)keyPath block:(RJBadgeNotificationBlock)block;
- (void)observePath:(NSString *)keyPath badgeView:(nullable id<P365BadgeView>)badgeView block:(nullable RJBadgeNotificationBlock)block;
```
第一个接口为某个被监测路径发生红点事件触发后提供block业务处理回调，第二个接口则为当发生事件后，在相应的badgeView上显示小红点UI, 这里传入的badgeView可以是一个button, 也可以是一个tab, 因而应该包括所有广义上的UI控件。

小红点的事件触发和分发则需要实现如下接口:
```
+ (void)setBadgeForKeyPath:(NSString *)keyPath;
+ (void)setBadgeForKeyPath:(NSString *)keyPath count:(NSUInteger)count;

+ (void)clearBadgeForKeyPath:(NSString *)keyPath;
+ (void)clearBadgeForKeyPath:(NSString *)keyPath forced:(BOOL)forced;
```
当App收到服务器推送有新内容更新时，需要对某个路径setBadge, 这边的setBadge会触发上面的observe block的回调。且如果消息为数量类型，比如未读消息时，还需要在setBadge的时候添加count属性。若用户点击了消息或进入了某个小红点提示的入口后，需要清除小红点消息，并且如果Observe的时候绑定了显示小红点的UI控件，也需要清除该控件上的小红点图标。

正常情况下，如果某个路径下面还有子路径有小红点，这个时候对该路径clearBadge是应该不起效果的，合理逻辑应该是当子路径的所有小红点都clear掉了后父路径自动清除。但如果这个情况下需要强制清除父路径红点，则需要在clear方法上加一个是否forced清除的参数。

#####小红点的UI显示

小红点的UI样式应该包括三种: **小红点**, **数字**和**自定义的icon或view**. 最基本的小红点主要用在业务入口处，用于内容、功能或动态更新的提醒。数字小红点则一般用来展示未读消息的数量。自定义的icon可以显示比如`new`, `免费`, `热门`等活动运营的提示，当然如果需要展示更复杂的UI设计也应该支持自定义view作为badge的功能。

既然可以展示三种样式的小红点UI, 那么就需要有一个优先级排序，结合上面的setBadge接口, 我们可以想到的规则是如果setBadge时没有设置count, 那么默认就是展示小红点, 如果设置了count, 那么就展示数字。另外在展示小红点的情况下，如果用户设置了自定义icon那么就优先展示icon, 按照这个思路，小红点样式的优先级就出来了: 数字的优先级最高，其次是自定义icon, 最后则是默认的圆形小红点。

对于UI, 我们都希望可以定制的，所以对于默认的圆形小红点应该可以调整它的半径，以及展示在控件上相对于右上角的offset, 而对于数字小红点应该可以调整它的字体和文字颜色。另外，如果数字的数值特别大，应该有个最高上限，比如超过99后就显示省略号。按照上面这些思路分析，我们可以得到下面所示的BadgeView接口:

```
@protocol RJBadgeView <NSObject>

@required

@property (nonatomic, strong) UILabel *badge;
@property (nonatomic, strong) UIFont  *badgeFont;      // default bold size 9
@property (nonatomic, strong) UIColor *badgeTextColor; // default white color
@property (nonatomic, assign) CGFloat badgeRadius;
@property (nonatomic, assign) CGPoint badgeOffset;     // offset from right-top

- (void)showBadge; // badge with red dot
- (void)hideBadge;

// badge with number, pass zero to hide badge
- (void)showBadgeWithValue:(NSUInteger)value;

@optional

@property (nonatomic, strong) UIView *customView;
/**
 convenient interface:
 create 'cusomView' (UIImageView) using badgeImage
 view's size would simply be set as half of image.
 */
@property (nonatomic, strong) UIImage *badgeImage;
```

小红点显示接口的调用理论上应该由内部来触发，也就是使用方调用:
```
+ (void)setBadgeForKeyPath:(NSString *)keyPath;
```
之后，
```
- (void)observePath:(NSString *)keyPath badgeView:(nullable id<P365BadgeView>)badgeView block:(nullable RJBadgeNotificationBlock)block;
```
这边所指定需要显示小红点的`badgeView`上会在小红点模块内部来调用`showBadge`. 当用户点击了显示小红点的控件后，应该在控件的点击事件里面调用`clearBadgeForKeyPath`来触发内部调用`hideBadge`. 简而言之，就是使用方不需要显式的来调用`badgeView`的`showBadge`或者`hideBadge`. 同理，当使用方调用:
```
+ (void)setBadgeForKeyPath:(NSString *)keyPath count:(NSUInteger)count;
```
会在内部调用badgeView的`showBadgeWithValue `. 当然如果使用方需要在某个控件上(e.g. badgeView -> UIButton)显示小红点，但是并不需要与某个路径关联，只是单纯的显示小红点，那应该也需要支持`[self.button showBadge]`的调用。

支持显示小红点的badgeView应该包括广义上的所有UI控件, iOS这边控件主要有3大种类: a). **UIView** b). **UIBarButtonItem** c). **UITabBarItem**, 所以我们可以对这三种类分别写一个category来创建小红点UI并显示在控件上，当然这三个category必须要conform上面的RJBadgeView Protocol:

```
@interface UIView (RJBadge) <RJBadgeView>
@interface UITabBarItem (RJBadge) <RJBadgeView>
@interface UIBarButtonItem (RJBadge) <RJBadgeView>
```

###接口优化

参照上面的讨论，我们需要对小红点路径进行监控，也就是要`observePath`, 类似于系统的KVO监测API, 这边会有下面几个需要考虑的问题:

1. 重复添加已有keyPath的observe
2. observe之后在observer退出或释放后忘记unobserve
3. 初始化小红点模块的复杂度和便利度 
4. block回调里面可能的循环引用问题

对于第一个问题，我们创建一个数据结构RJBadgeInfo, 用来存放小红点的相关信息，每次添加observe对info进行比较，如果已有监测则不去做重复添加。
```
@interface RJBadgeInfo : NSObject

@property (nonatomic, copy,   readonly) NSString                 *keyPath;
@property (nonatomic, weak,   readonly) RJBadgeController        *controller;
@property (nonatomic, copy,   readonly) RJBadgeNotificationBlock block;
@property (nonatomic, strong, readonly) id<RJBadgeView>          badgeView;

@end
```

第二个问题可以使用自释放的机制来实现observe的自动移除，这样就需要将badgeController作为观察者的成员变量，当observer释放之后badgeController也会释放，那么我们就在badgeController的  `dealloc`函数中去做observe的移除操作。使用方则无需关心何时去移除观察者，当然如果确实需要提前移除观察者，也可以调用unobservePath接口。

初始化函数生成badgeController并且以observer的成员变量存在，最简单和便捷的方式就是给所有NSObject对象通过category添加badgeController变量，这样用户无需显式去调用`alloc`方法，只需要`self.badgeController`即可动态生成badgeController对象。

```
@interface NSObject (RJBadgeController)

@property (nonatomic, strong) RJBadgeController *badgeController;

@end
```
在badgeController的`get`方法里面则是调用RJBadgeController的初始化方法生成对象并赋值给`self.badgeController`变量:
```
- (RJBadgeController *)badgeController
{
    id controller = objc_getAssociatedObject(self, NSObjectBadgeControllerKey);
    // lazily create the badgeController
    if (nil == controller) {
        controller           = [RJBadgeController controllerWithObserver:self];
        self.badgeController = controller;
    }
    return controller;
}

- (void)setBadgeController:(RJBadgeController *)badgeController 
{
    objc_setAssociatedObject(self, 
                             NSObjectBadgeControllerKey, 
                             badgeController, 
                             OBJC_ASSOCIATION_RETAIN_NONATOMIC);
}
```

最后一个循环引用的问题，在badge的block里面用参数observer来代替self, 我们对observer(即self.badgeController的self)进行weak化处理并通过block回调参数传出:

```
[self.badgeController observePath:@"root.xx" 
                        badgeView:button 
                            block:^(RJViewController *observer, NSDictionary *info) {
    // Use [observer doSomething] instead of [self doSomething] to avoid retain cycle in block
    // key path     -> info[RJBadgePathKey] : badgeContoller所observe的路径
    // badge status -> info[RJBadgeShowKey] : 当前路径所对应的badge是否处于set状态(是否应该显示小红点)
    // badge count  -> info[RJBadgeCountKey]: 当前路径所对应的badge数值(仅在badge为数值模式下有效)
}];
```

###方案实现

理论talk完了，可以show源码了，完整的小红点解决方案实现源码[RJBadgeKit](https://github.com/RylanJIN/RJBadgeKit)已经发布到GitHub, 可以直接通过cocoapods, pod 'RJBadgeKit'集成使用。我们来看下具体应用示例:

假设我们有个促销页面，该促销有两个商品参与活动，则促销页面的路径可设置为root.promotion，促销页面内两个商品的路径分别设为root.promotion.item1, root.promotion.item2. 现在需要推送小红点消息给用户，在promotion的入口处的button需要显示小红点提示，当用户进入到promotion页面且分别点击了item1和item2后，promotion的小红点提示才消失。

首先我们在RJPromotionViewController里面对promotionButton添加路径的观察者，当该路径被setBadge时候则显示小红点，clearBadge时则隐藏小红点:

```
[self.badgeController observePath:@"root.promotion" 
                        badgeView:promotionButton 
                            block:^(RJPromotionViewController *observer, 
                                    NSDictionary *info) {
    BOOL hasPromotionItem = [info[RJBadgeShowKey] boolValue];
    [observer setPromotionStatus:hasPromotionItem];
}];
```

当网络请求返回时发现有两个促销数据(注意路径的格式)，则调用:

```
[RJBadgeController setBadgeForKeyPath:@"root.promotion.item1"];
[RJBadgeController setBadgeForKeyPath:@"root.promotion.item2"];
```

子路径的小红点状态变化会触发父路径observe的block回调，所以上述两行代码执行后promotionButton会触发显示小红点。当然如果希望promotionButton不显示小红点，而是显示具体的促销数量，则可以直接如下调用:

```
[RJBadgeController setBadgeForKeyPath:@"root.promotion" count:2];
```

如果promotion item下面还有子路径, 则调用:

```
[RJBadgeController setBadgeForKeyPath:@"root.promotion.item1" count:5];
```

在这个情况下，promotionButton上显示的数值(亦即root.promotion路径对应的badge值)为root.promotion.item1和root.promotion.item2及其所有子节点的数值之和。当用户点击查看了item1和item2后，分别调用clearBadeg方法来消除小红点:

```
[RJBadgeController clearBadgeForKeyPath:@"root.promotion.item1"];
[RJBadgeController clearBadgeForKeyPath:@"root.promotion.item2"];
```

这时父节点root.promotion的badge自动clear, promotionButton的小红点会自动隐藏。如果希望在item1被clear后就强制清除root.promotion的badge, 则可以在清除item1后调用:

```
[RJBadgeController clearBadgeForKeyPath:@"root.promotion" force:YES];
```

这样即使子节点的badge尚未全部清除，父节点也会被强制clear. 
> 正常情况下不应该去调用force:YES, 如果非要调用，可能是路径结构设计不合理了

对于小红点的样式, [RJBadgeKit](https://github.com/RylanJIN/RJBadgeKit)可以通过offset来设置显示位置，也可以传入需要展示的自定义红点icon. 如果需要展示的样式非常复杂，那也可以直接传入定制的view用来作为badge展示:
```
promotionButton.badgeOffset = CGPointMake(-50, 0); // 调整小红点的显示位置offset, 相对于右上角

[self.promotionButton setBadgeImage:[UIImage imageNamed:@"badgeNew"]]; // 显示自定义的badge icon

[self.promotionButton setCustomView:self.customBadgeView]; // 显示自定义的badge view
```

下图为[RJBadgeKit](https://github.com/rjinxx/RJBadgeKit)所对应的Example运行效果, 更详细的使用示例及所有支持的接口方法和属性设置可以参考Example工程。

![demo.gif](http://upload-images.jianshu.io/upload_images/268805-43abde9d46c846a0.gif?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

最后再贴一下源码地址: https://github.com/rjinxx/RJBadgeKit, 在使用中有遇到什么问题或者优化建议欢迎留言PR, 如果[RJBadgeKit](https://github.com/rjinxx/RJBadgeKit)的实现方案对你有所帮助和启发，也不妨给个Star鼓励下。