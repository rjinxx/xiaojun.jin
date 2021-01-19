---
title: 轻量级非侵入式埋点方案
subtitle: 在发展日新月异的移动互联网时代，数据扮演着极其重要的角色。埋点作为一种最简单最直接的用户行为统计方式，能够全面精确的采集用户的使用习惯以及各功能点的迭代反馈等等，有了这些数据才能更好的驱动产品的决策设计和新业务场景的规划。

# Summary for listings and search engines
summary: 在发展日新月异的移动互联网时代，数据扮演着极其重要的角色。埋点作为一种最简单最直接的用户行为统计方式，能够全面精确的采集用户的使用习惯以及各功能点的迭代反馈等等，有了这些数据才能更好的驱动产品的决策设计和新业务场景的规划。

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
- 埋点

categories:
- 移动开发
- 埋点
---

在发展日新月异的移动互联网时代，数据扮演着极其重要的角色。埋点作为一种最简单最直接的用户行为统计方式，能够全面精确的采集用户的使用习惯以及各功能点的迭代反馈等等，有了这些数据才能更好的驱动产品的决策设计和新业务场景的规划。本文旨在提出一种轻量级非侵入式的埋点方案，其主要有以下三方面优势

- 支持动态下发埋点配置
- 物理隔离埋点代码和业务代码
- 插件式的埋点功能实现

该方案通过维护一个`JSON`文件来指定埋点所在的类和方法，继而利用`AOP`的方式在对应的类和方法执行时动态嵌入埋点代码。对于需要逻辑判断来确定埋点值的场景，提供`hook`方法的入参，以及所在类的属性值读取，根据相应的状态值设置不同的埋点

### 埋点配置
埋点配置`JSON`表中包含需要`hook`的类名`class`和具体的事件`event`信息，`event`中包括`hook`的方法和对应的埋点值。如下所示
```
{
    "version": "0.1.0",
    "tracking": [
        {
            "class": "RJMainViewController",
            "event": {
                "rj_main_tracking": [
                    "tripTypeViewChangedWithIndex:",
                    "tripLabClickWithLabKey:"
                ],
                "user_fp_slide_click": "clickNavLeftBtn",
                "user_fp_reflocate_click": "clickLocationBtn"
            }
        },
        {
            "class": "RJTripHistoryViewModel",
            "event": {
                "user_mytrip_show": "tableView:didSelectRowAtIndexPath:"
            }
        },
        {
		    "class": "RJTripViewController",
            "event": {
                "rj_trip_tracking": "callServiceEvent"
            }
        }
    ]
}
```
简单来说就是本来埋点需要手动在该方法写入埋点代码来记录埋点值，现在通过`AOP`的方式物理隔离埋点代码和业务代码，避免埋点的逻辑侵入污染业务逻辑。埋点包括固定埋点和需要逻辑判断的场景化埋点，固定埋点如下所示
```
{
    "class": "RJTripHistoryViewModel",
    "event": {
        "user_mytrip_show": "tableView:didSelectRowAtIndexPath:"
    }
}
```
`RJTripHistoryViewModel`为类名，`tableView:didSelectRowAtIndexPath:`为需要`hook`的该类中的方法，而`user_mytrip_show`则是具体的埋点值，也就是当`RJTripHistoryViewModel`中的`tableView:didSelectRowAtIndexPath:`方法执行的时候记录埋点值`user_mytrip_show`

```
{
    "class": "RJTripViewController",
    "event": {
        "rj_trip_tracking": "callServiceEvent"
    }
},
```
对于场景化埋点，则需要提供一个`impl`类来提供相应的逻辑判断。比如上述配置表中的`rj_trip_tracking`为场景埋点的实现类，在该类中根据状态量返回对应的埋点值，即当`callServiceEvent`方法执行时会去找`rj_trip_tracking`这个埋点`impl`同名类，取该类返回的埋点值记录埋点。需要注意到是`event`中的`key`值既可以作为埋点值也可以作为`impl`的类名，埋点库会首先判断是否存在对应的类，存在即认为是`impl`实现类，从该类中取具体的埋点值。反之，则认为是固定埋点值

> 配置表中的类名和方法名需要对应，在`hook`的时候会去匹配，如果发现类中不存在对应的方法，则会自动触发断言

### 固定埋点

对于固定的埋点，只需要在对应的方法执行时直接记录埋点，利用[Aspects](https://github.com/steipete/Aspects)来`hook`指定的类和方法，代码如下所示
```
[class aspect_hookSelector:sel withOptions:AspectPositionBefore usingBlock:^(id<AspectInfo> info) {
    [events enumerateObjectsUsingBlock:^(NSString *name, NSUInteger idx, BOOL *stop) {
        NSLog(@"<RJEventTracking> - %@", ename);
    }];
} error:&error];
```
为了便于检测无效的埋点，还需对`hook`的类和方法进行匹配校验，若类中没有对应的方法，则抛出断言
```
+ (void)checkValidWithClass:(NSString *)class method:(NSString *)method {
    SEL sel       = NSSelectorFromString(method);
    Class c       = NSClassFromString(class);
    BOOL respond  = [c respondsToSelector:sel] || [c instancesRespondToSelector:sel];
    NSString *err = [NSString stringWithFormat:@"<RJEventTracking> - no specified method: %@ found on class: %@, please check", method, class];
    
    NSAssert(respond, err);
}
```

### 场景埋点
场景化埋点主要为同一事件但是在多种状态或逻辑下不同埋点的情况，比如同是联系客服的操作，在各种订单类型以及订单状态下所设置的埋点是不同的。这个情况下，埋点库通过提供一个`protocol`由埋点`impl`类来实现，根据不同的逻辑判断，返回对应的埋点值

```
@protocol RJEventTracking <NSObject>

- (NSString *)trackingMethod:(NSString *)method instance:(id)instance arguments:(NSArray *)arguments;

@end
```

比如上文的`rj_trip_tracking`类需要遵循`RJEventTracking`协议，并根据相关逻辑判断返回对应的埋点值

> 埋点实现类的类名需要与埋点配置`JSON`中的`event`里的`key`保持一致，因为埋点库会通过检测是否有同名的类来实现插件式的埋点规则。另外，一个`impl`可以对应多个`method`方法

##### 状态判断
根据状态量来确定埋点值。还是联系客服埋点的例子，根据订单种类和订单状态来返回对应的埋点值，首先定义`JSON`表中同名的`impl`类，并遵循`RJEventTracking`协议

```
#import "RJEventTracking.h"
 
NS_ASSUME_NONNULL_BEGIN
 
@interface rj_trip_tracking : NSObject <RJEventTracking>
 
@end
 
NS_ASSUME_NONNULL_END
```

在.m文件中实现自定义埋点的协议方法`trackingMethod:instance:arguments: `

```
#import "rj_trip_tracking.h"
 
@implementation rj_trip_tracking
 
- (NSString *)trackingMethod:(NSString *)method instance:(id)instance arguments:(NSArray *)arguments {
    id dataManager        = [instance property:@"dataManager"];
    NSInteger orderStatus = [[dataManager property:@"orderStatus"] integerValue];
    NSInteger orderType   = [[dataManager property:@"orderType"] integerValue];
 
    if ([method isEqualToString:@"callServiceEvent"]) {
        if (orderType == 1) {       
            if (orderStatus == 1) { 
                return @"user_inbook_psgservice_click";
            } else if (orderStatus == 2) {
                return @"user_finishbook_psgservice_click";
            }
        } else {
            return @"user_psgservice_click";
        }
    }
    return nil;
}
 
@end
```
在协议方法中，可以获取当前的实例（在这个示例下为`RJTripViewController`）和入参数组。订单的类型和状态是存储在`RJTripViewController`中的`dataManager`属性中的，所以可以通过埋点库封装好的`property:`方法来获取属性值，并根据属性值返回对应的埋点名称 

```
@interface NSObject (RJEventTracking)
 
- (id)property:(NSString *)property;
 
@end
```

属性值读取的实现为
```
- (id)property:(NSString *)property {
    return [NSObject runMethodWithObject:self selector:property arguments:nil];
}
```

其中的原理很简单，就是将`getter`方法封装到`NSInvocation`中并`invoke`读取返回值即可

```
+ (id)runMethodWithObject:(id)object selector:(NSString *)selector arguments:(NSArray *)arguments {
    if (!object) return nil;
    
    if (arguments && [arguments isKindOfClass:NSArray.class] == NO) {
        arguments = @[arguments];
    }
    SEL sel = NSSelectorFromString(selector);
        
    NSMethodSignature *signature = [object methodSignatureForSelector:sel];
    if (!signature) {
        return nil;
    }
    NSInvocation *invocation = [NSInvocation invocationWithMethodSignature:signature];
    invocation.selector      = sel;
    invocation.arguments     = arguments;
    [invocation invokeWithTarget:object];
    
    return invocation.returnValue_obj;
}
```

##### 入参判断

需要根据`JSON`中设置的所`hook`方法的入参来确定埋点名称的情况。比如在订单列表中点击全部，进行中，待支付，待评价，已完成等菜单项时分别埋点。被`hook`的方法为`tripLabClickWithLabKey:`其参数为`UILabel`，原先代码中通过`Label`的`tag`判断是点击的哪个子项，同样，我们也可以获取到`Label`的入参然后据此判断。由于参数只有一个，所以可以直接取`arguments`第一个值 

```
#import "rj_main_tracking.h"
#import <UIKit/UIKit.h>
 
static NSString *order_types[5] = { @"user_order_all_click",   @"user_order_ongoing_click",
                                    @"user_order_unpay_click", @"user_order_unmark_click",
                                    @"user_order_finish_click" };
@implementation rj_main_tracking
 
- (NSString *)trackingMethod:(NSString *)method instance:(id)instance arguments:(NSArray *)arguments {
    if ([method isEqualToString:@"tripLabClickWithLabKey:"]) {
        UILabel *label = arguments[0];
        if (!label || label.tag > 4) {
            return nil;
        }
        return order_types[label.tag];
    } else if ([method isEqualToString:@"tripTypeViewChangedWithIndex:"]) {
        return @"xx_ryan_jin";
    }
}
 
@end
```

通过`AOP`来`hook`方法时，可以获取到当前`hook`方法所对应的实例对象和入参，在调用协议方法时，直接传给协议实现类

##### 方法调用

和读取属性值类似，也是在不同场景下同一事件不同埋点名称的情况，但获取的状态量不是当前实例对象的，而是某个方法的返回值，这种情况下可以通过埋点库提供的方法调用函数来实现

```
@interface NSObject (RJEventTracking)
 
- (id)performSelector:(NSString *)selector arguments:(nullable NSArray *)arguments;
 
@end
```

比如获取某个页面的视图类型，而这个视图类型存储于单例对象中

```
[RJViewTypeModel sharedInstance].viewType
```

该场景下则根据viewType的类型，来返回相应的埋点名称 

```
- (NSString *)trackingMethod:(NSString *)method instance:(id)instance arguments:(NSArray *)arguments {
    NSString *labKey   = [instance property:@"labKey"];
    id viewTypeModel   = [NSClassFromString(@"RJViewTypeModel") performSelector:@"sharedInstance"
                                                                      arguments:nil];
    NSInteger viewType = [[viewTypeModel property:@"viewType"] integerValue];
     
    if (viewType == 0) { 
        if ([labKey isEqualToString:@"rj_view_begin_add"]) {
            return @"user_fp_book_on_click";
        }
        if ([labKey isEqualToString:@"rj_view_end_add"]) {
            return @"user_fp_book_off_click";
        }
    }
    if (viewType == 1) { 
        if ([labKey isEqualToString:@"rj_view_begin_add"]) {
            return @"user_fr_on_click";
        }
        if ([labKey isEqualToString:@"rj_view_end_add"]) {
            return @"user_fr_off_click";
        }
    }
    return nil;
}
```

##### 逻辑判断

需要额外添加逻辑判断的场景，比如在订单详情页需要统计用户进入页面的查看行为，但是详情页的类型需要在网络请求后才能获取，而且该网络请求会定时触发，所以埋点`hook`的方法会走多次，该情况下，需要添加一个属性用来标记是否已记录埋点 。故而埋点库需要提供动态添加属性的功能

```
@interface NSObject (RJEventTracking)
 
- (id)extraProperty:(NSString *)property;
 
- (void)addExtraProperty:(NSString *)property defaultValue:(id)value;
 
@end
```

在埋点实现`impl`类里面，添加额外的属性来标记是否已记录过埋点

```
@implementation user_orderdetail_show
 
- (NSString *)trackingMethod:(NSString *)method instance:(id)instance arguments:(NSArray *)arguments {
    if ([instance extraProperty:@"isRecorded"]) {
        return nil;
    }
    [instance addExtraProperty:@"isRecorded" defaultValue:@(YES)];
     
    return @"user_orderdetail_show";
}
 
@end
```

使用`addExtraProperty:defaultValue:`来给当前实例动态添加属性，而`extraProperty:`方法则用来获取实例的某个额外属性。如果`isRecorded`返回`YES`代表已经记录过该埋点，返回`nil`值来忽略该次埋点

> 上面示例中添加的isRecorded属性是因为埋点的需求，和业务逻辑无关，所以比较合理的方式是在埋点的插件`impl`类中添加，避免影响业务代码

埋点库动态添加属性的原理也很简单，利用`runtime`的`objc_setAssociatedObject`和`objc_getAssociatedObject`方法来绑定属性到实例对象

```
- (id)extraProperty:(NSString *)property {
    return objc_getAssociatedObject(self, NSSelectorFromString(property));
}

- (void)addExtraProperty:(NSString *)property defaultValue:(id)value {
    objc_setAssociatedObject(self, NSSelectorFromString(property), value, OBJC_ASSOCIATION_RETAIN_NONATOMIC);
}
```

### 动态下发

埋点`JSON`配置表可以由服务器提供接口，客户端在每次启动时通过接口获取最新埋点配置表，从而达到动态下发的目的，客户端拿到`JSON`后，读取埋点信息并生效

```
[RJEventTracking loadConfiguration:[[NSBundle mainBundle] pathForResource:@"RJUserTracking" ofType:@"json"]];
```

读取的代码如下所示，主要逻辑为遍历埋点中的类和`hook`的方法，并检测是固定埋点还是场景化埋点，对于场景化埋点的情况查询是否有对应的埋点`impl`实现类。当然，还需检测`JSON`配置表的合法性，每个类和其中的方法是否匹配

```
+ (void)loadConfiguration:(NSString *)path {
    NSData *data = [NSData dataWithContentsOfFile:path];
    if (!data) {
        return;
    }
    NSDictionary *dict = [NSJSONSerialization JSONObjectWithData:data options:NSJSONReadingMutableLeaves error:nil];
    NSString *version  = dict[@"version"];
    NSArray *ts        = dict[@"tracking"];
    [ts enumerateObjectsUsingBlock:^(NSDictionary *obj, NSUInteger idx, BOOL *stop) {
        Class class              = NSClassFromString(obj[@"class"]);
        NSDictionary *ed         = obj[@"event"];
        NSMutableDictionary *td  = [NSMutableDictionary dictionaryWithCapacity:0];
        [ed enumerateKeysAndObjectsUsingBlock:^(NSString *key, id obj, BOOL *stop) {
            NSMutableArray *tArr = [NSMutableArray arrayWithCapacity:0];
            [tArr addObjectsFromArray:[obj isKindOfClass:[NSArray class]] ? obj : @[obj]];
            [tArr enumerateObjectsUsingBlock:^(NSString *m, NSUInteger idx, BOOL *stop) {
                if ([td.allKeys containsObject:m]) {
                    NSMutableArray *ms         = [td[m] mutableCopy];
                    if (![ms containsObject:key]) [ms addObject:key];
                    td[m] = ms;
                } else {
                    td[m] = @[key];
                }
            }];
        }];
        [td enumerateKeysAndObjectsUsingBlock:^(NSString *kmethod, NSArray <NSString *> *tArr, BOOL *stop) {
            SEL sel        = NSSelectorFromString(kmethod);
            NSError *error = nil;
            [self checkValidWithClass:obj[@"class"] method:kmethod];
            [class aspect_hookSelector:sel withOptions:AspectPositionBefore usingBlock:^(id<AspectInfo> info) {
                [tArr enumerateObjectsUsingBlock:^(NSString *name, NSUInteger idx, BOOL *stop) {
                    NSString *ename       = name;
                    id<RJEventTracking> t = [NSClassFromString(name) new];
                    if (t && [t respondsToSelector:@selector(trackingMethod:instance:arguments:)]) {
                        ename = [t trackingMethod:kmethod instance:info.instance
                                                         arguments:info.arguments];
                    }
                    if ([ename length]) {
                        NSLog(@"<RJEventTracking> - %@", ename);
                    }
                }];
            } error:&error];
            [self checkHookStatusWithClass:obj[@"class"] method:kmethod error:error];
        }];
    }];
}
```

最后附上源码地址: https://github.com/rjinxx/RJEventTracking, 
```
pod 'RJEventTracking'
```
在使用[RJEventTracking](https://github.com/rjinxx/RJEventTracking)的过程中中有遇到什么问题或者优化建议欢迎留言PR，谢谢。