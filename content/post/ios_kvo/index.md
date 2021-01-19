---
title: 读源码涨姿势之优雅KVO实现
subtitle: ""

# Summary for listings and search engines
summary: 如果说书籍是人类进步的阶梯，那么优秀的开源代码就是程序员提升的桥梁。研读源码可以学习其中的框架和模式, 代码技巧, 算法等，然后不断总结运用，最终这些会变成自己的东西，编程水平自然也提高了。

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
- KVO

categories:
- iOS
- KVO
---

如果说书籍是人类进步的阶梯，那么优秀的开源代码就是程序员提升的桥梁。研读源码可以学习其中的框架和模式, 代码技巧, 算法等，然后不断总结运用，最终这些会变成自己的东西，编程水平自然也提高了。

FBKVOController是Facebook开源的接口设计优雅的KVO框架。笔者研读之后确实受益匪浅，本着学以致用的原则，笔者借鉴其接口设计的方式实现了一套完整的小红点(推送消息)解决方案[RJBadgeKit](https://github.com/RylanJIN/RJBadgeKit), 有兴趣的同学可以参考一下。

鉴于目前已经有很多关于FBKVOController源码分析的博文，本文会尝试从另外一个角度，以提炼和分析具体知识点的方式来总结FBKVOController中我们可以借鉴和学习的地方。

###宏定义

通常在添加观察者的时候都需要指定一个观察路径(keyPath), 这个路径是直接以字符串的方式提供的，比如我们有个类`RJPhoto`的对象`photo`, 需要观察它的`name`路径:
```
[self.KVOController observe:photo keyPath:@"name"];
```
如果字符串拼写错误，或者被observe的对象没有`name`这个属性，编译器并不会报错，只有等到运行时才会发现问题。我们来看下FBKVOController是怎么通过宏定义来解决这个问题的:
```
#define FBKVOKeyPath(KEYPATH) \
@(((void)(NO && ((void)KEYPATH, NO)), \
({ const char *fbkvokeypath = strchr(#KEYPATH, '.'); NSCAssert(fbkvokeypath, @"Provided key path is invalid."); fbkvokeypath + 1; })))

#define FBKVOClassKeyPath(CLASS, KEYPATH) \
@(((void)(NO && ((void)((CLASS *)(nil)).KEYPATH, NO)), #KEYPATH))
```
有了这两个宏，被观察者的`keyPath`可以通过宏传入，其好处在于该宏会进行编译检查和代码提示，如果`keyPath`不存在或者拼写错误，会提示错误。
```
[self.KVOController observe:photo keyPath:FBKVOKeyPath(photo.name)];
[self.KVOController observe:photo keyPath:FBKVOClassKeyPath(RJPhoto, name)];
```
上面的宏是怎么做到编译检查和代码提示的呢？我们先分析第二个相对比较复杂的宏`FBKVOClassKeyPath`, 其整体是一个C语言的逗号表达式，逗号表达式的格式: e.g. `int a = (b, c);`逗号表达式取后面的值，故而a将被赋值成c, 此时b在赋值运算中就被忽略了，没有被使用，所以编译器会给出警告，为了消除这个warning我们需要在b前面加上`(void)`做个类型强转操作。

逗号表达式的前项和NO进行了与操作，这个主要是为了让编译器忽略第一个值，因为我们真正赋值的是表达式后面的值。预编译的时候看见了NO, 就会很快的跳过判断条件。我猜你看到这儿肯定会奇怪了，既然要忽略，那为啥还要用个逗号表达式呢，直接赋值不就好了？

这里主要是对传入的第一个参数CLASS的对象`(CLASS *)(nil)`和第二个正要输入的`KEYPATH`做了`.`操作，这也正是为什么输入第二个参数时编辑器会给出正确的代码提示（只要是作为表达式的一部分, Xcode自动会提示）。如果传入的KEYPATH不是CLASS对象的属性，那么`(CLASS *)(nil).KEYPATH`就不是一个合法的表达式，所以自然编译就不会通过了。

宏`FBKVOKeyPath `接受一个参数，前半段和上面是一样的，不同的是逗号表达式的后一段`strchr(# photo.name, '.') + 1`, 函数`strchar`是C语言中的函数，用来查找某字符在字符串中首次出现的位置，这里用来在`photo.name`(注意前面加了`#`字符串化)中查找`.`出现的位置，再加上`1`就是返回`.`后面`keyPath`的地址了。也就是`strchr('photo.name', '.')`返回的是一个C字符串，这个字符串从找到`'photo.name'`中为`'.'`的字符开始往后，即`'name'`.

>这边还用到了断言宏`NSCAssert(x, y)`, `x`为`BOOL`值, `y`为字符串类型, 当`x`为`NO`时产生断言退出并打印`y`字符串内容. 需要注意的是`NSCAssert`在C语言函数下使用, 而`NSAssert `则是Objective-C函数下使用

关于宏定义的详细解释以及上面所述的类似宏定义的应用和分析，可以参考笔者的博文[Hello, 宏定义魔法世界](https://www.jianshu.com/p/4a1531bac39f)。

###自释放

FBKVOController通过自释放的机制来实现observer的自动移除，具体来说就是给observer添加一个FBKVOController的成员变量，比如:

```
#import "RJViewController.h"
#import "KVOController.h"

@interface RJViewController ()

@property (nonatomic, strong) FBKVOController *kvoController;

@end

@implementation RJViewController

- (instancetype)init
{
  self = [super init];
  if (nil != self) {
      _kvoController = [FBKVOController controllerWithObserver:self];
  }
  return self;
}
```
观察者`RJViewController`定义了一个`FBKVOController`的成员变量`kvoController`, 当`RJViewController`释放后，其成员变量`kvoController`也会相应释放，FBKVO自动移除观察者的trick就是在`FBKVOController`的`dealloc`里面做remove observer的操作。

###初始化

我们先来看看FBKVOController是如何提供初始化接口的:
```
+ (instancetype)controllerWithObserver:(nullable id)observer;
- (instancetype)initWithObserver:(nullable id)observer retainObserved:(BOOL)retainObserved NS_DESIGNATED_INITIALIZER;
- (instancetype)initWithObserver:(nullable id)observer;
```
一共有3个初始化函数，其中第一个和第三个为便利初始化函数(convenience initializer), 第二个添加了`NS_DESIGNATED_INITIALIZER`宏，为指定初始化函数(designated initializer). 
> 初始化接口的规则:
**a)** 指定初始化方法必须调用父类的指定初始化方法
**b)** 便利初始化方法必须调用其他的初始化方法，直到最后指向指定初始化方法
**c)** 具有指定初始化方法的子类必须实现所有父类的指定初始化方法

理解了自释放的原理，初始化的规则就很明显了，FBKVOController必须作为`observer`的成员变量存在。那如果使用方忽视了这个规则或者不想如此繁琐，有没有更简单的方式呢？有！我们来看下FBKVO是怎么提供最简化convenience initializer的:
```
@interface NSObject (FBKVOController)

@property (nonatomic, strong) FBKVOController *KVOController;
@property (nonatomic, strong) FBKVOController *KVOControllerNonRetaining;

@end
```
FBKVOController创建了NSObject的Category, 通过AssociateObject给NSObject提供一个Retain和nonRetain的KVOController(这里实际也是在成员变量`KVOController`的Get函数里面调用了`+ controllerWithObserver `方法)。所以任意observer都可以直接调用observer.KVOController来动态生成一个FBKVOController对象，非常方便！

到这儿看起来初始化接口已经很完整了，但好像还有个问题，万一使用方不按套路直接来个系统默认的初始化函数`[[FBKVOController alloc] init]`或者`[FBKVOController new]`那Observer岂不是就没有了。怎么做才能提醒使用方不要调用系统的初始化函数呢？
```
/**
 @abstract Allocates memory and initializes a new instance into it.
 @warning This method is unavaialble. Please use `controllerWithObserver:` instead.
 */
- (instancetype)init NS_UNAVAILABLE;

+ (instancetype)new NS_UNAVAILABLE;
```
答案就是在这两个默认初始化函数后面加上`NS_UNAVAILABLE`宏，这样如果使用方误用了系统默认的初始化函数时会给出警告，提醒他应该使用模块指定的初始化接口方法。

###NSHashTable & NSMapTable

NSHashTable可以理解为更广泛意义上的NSMutableSet, 与后者相比NSMapTable主要有如下特性:

- NSHashTable是可变的, 没有不可变版本
- 可以弱引用所存储的元素, 当元素释放后会自动被移除
- 可以在添加元素的时候复制元素后再存放

与NSMutableSet相同之处(与NSMutableArray不同之处)则是:

- 元素都是无序存放的
- 根据`hash`和`isEqual`来对元素进行比较
- 不会存放相同的元素

关于对比，我们要先区分`==`运算符和`isEqual`方法:
```
UIColor *color1 = [UIColor colorWithRed:0.5 green:0.5 blue:0.5 alpha:1.0];
UIColor *color2 = [UIColor colorWithRed:0.5 green:0.5 blue:0.5 alpha:1.0];
```
在上面的示例中`color1 == color2`返回false, 但`[color1 isEqual:color2]`却是返回true, 原因在于`==`是直接的指针比较，显然color1和color2的地址是不同的，而isEqual则是判断其颜色内容是否相同。

>类似的还包括NSString isEqualToString / NSDate isEqualToDate

回到NSHashTable中来, NSHashTable可以随意的存储指针并且利用指针的唯一性来进行hash同一性检查(检查成员元素是否有重复)和对比操作(isEqual), 当然我们也可以重写hash/isEqual方法来设定元素对比和相等的规则(其实isEqual是NSObject定义的Protocol). 我们来看下面这个[示例](https://www.jianshu.com/p/915356e280fc):

```
@interface Person : NSObject

@property (nonatomic,   copy) NSString *name;
@property (nonatomic, strong) NSDate   *birthday;

+ (instancetype)personWithName:(NSString *)name birthday:(NSDate *)date;

@end
```
我们定义一个Person类，其包括name和birthday两个属性，在我们的正常认知下，如果两个Persion对象的这两个属性是一致的，那么他们就是同一个人，所以类似上面UIColor的例子，我们需要重写下Person的isEqual函数:
```
- (BOOL)isEqual:(id)object
 {
    if (nil == object) {
        return NO;
    }
    if (self == object) {
        return YES;
    }
    if (![object isKindOfClass:[Person class]]) {
        return NO;
    }    
    return [self isEqualToPerson:(Person *)object];
}

- (BOOL)isEqualToPerson:(Person *)person
 {
    if (!person) return NO;
      
    BOOL haveEqualNames     = (!self.name && !person.name) || [self.name isEqualToString:person.name];
    BOOL haveEqualBirthdays = (!self.birthday && !person.birthday) || [self.birthday isEqualToDate:person.birthday];
    
    return haveEqualNames && haveEqualBirthdays;
}
```
这里的isEqual函数的实现分为四步，也是我们推荐的best pratice:
1. 判断对象是否为空
2. 判断是否同一对象（内存地址是否相等）
3. 如果不是同一个class那肯定不是同一对象
4. 判断对象的各属性值是否相等

```
Person *person1 = [Person personWithName:@"Ryan Jin" birthday:self.date];
Person *person2 = [Person personWithName:@"Ryan Jin" birthday:self.date];
```
现在如果判断`[person1 isEqual person2]`就是返回true了，不过怎么感觉缺了点什么, hash好像还没用到呀，难道不需要重写hash方法吗？

答案当然是需要，当成员被加入到NSHashTable(也包括NSSet)中时，会被分配一个hash值，以标识该成员在集合中的位置，通过这个位置标识可以极大的提升成员查找的效率(这也是为什么NSHashTable 查找元素的速度会快于NSArray). 

由于NSHashTable/NSSet在添加元素的时候会就行判等操作，当某个元素已经存在时不会重复添加，这个判等的操作包括两步:
1. 两个成员的hash值是否相等，如不相等则立即判断为不同元素
2. 若hash值相等，则再判断isEqual是否返回一致

只有两个元素的`hash`和`isEqual`都为一致的情况下才判断为相同对象。好了，明白了这个原则，我们来重写下Person的hash方法:
```
- (NSUInteger)hash {
    return [self.name hash] ^ [self.birthday hash]; // best practice
}
```
由于系统的NSString和NSDate在内容相同的情况下会返回相同的hash值，所以这边的最佳实践是返回各属性的位或运算。这边需要注意的是不能简单的返回`[super hash]`, 因为默认的hash值为该对象的内存地址，所以上面的`person1`和`person2`它们的`[super hash]`是不同的，所以会被判定为不同的元素，而我们想要实现的是当Person的各属性一致的时候它们即为同一元素。

> NSHashTable/NSSet在添加元素和判断某个元素是否存在(member:/containsObject:)时会调用hash方法, 另外NSDictionary在查找key时(key为非字符串对象), 也会利用hash值来提高查找效率

我们来看下FBKVO里面用到NSHashTable地方:
```
NSHashTable *infos = [[NSHashTable alloc] initWithOptions:NSPointerFunctionsWeakMemory|NSPointerFunctionsObjectPointerPersonality capacity:0];
```
这里初始化了一个NSHashTable, 存放类型为`NSPointerFunctionsWeakMemory`即弱持有成员元素且当元素释放后自动从NSHashTable移除。另外，判等类型为`NSPointerFunctionsObjectPointerPersonality`即直接使用指针地址是否相等来判断。如果类型设置为`NSPointerFunctionsObjectPersonality`则采用上面所描述hash和isEqual来判断。

FBKVO这边采用直接指针地址进行元素比较的原因是单例`_FBKVOSharedController`的`infos`里面所存放的`_FBKVOInfo`都是从`FBKVOController`传入过来的，已经经过了判等操作，不会出现相同的对象，所以`_infos`处理这些`_FBKVOInfo`元素直接用指针比较就好了，没必要再去调用`hash`和`isEqual`方法。另外`NSPointerFunctionsWeakMemory`设置是为了在`_FBKVOInfo`释放后自动从`_infos`里面移除它, `_FBKVOInfo`都不存在了，放在里面也没意义了。从这边可以看到FBKVO设计的确实很细致。

NSMapTable可以理解为更广泛意义上的NSMutableDictionary, 其各项特性和NSHashTable基本相同:
- NSMapTable是可变的, 没有不可变版本
- 可以弱引用持有keys和values, 当key或value释放后存储的实体会被移除
- NSMapTable可以在添加value的时候对value进行复制

```
NSMapTable *keyToObjectMapping = [NSMapTable mapTableWithKeyOptions:NSMapTableCopyIn
                                                       valueOptions:NSMapTableStrongMemory];
```
如果按上面这么设置NSMapTable会和NSMutableDictionary用起来完全一样: 复制 key, 并对它的object引用计数加一。同样，我们也来看下FBKVO使用NSMapTable的地方:
```
- (instancetype)initWithObserver:(nullable id)observer retainObserved:(BOOL)retainObserved
{
  self = [super init];
  if (nil != self) {
    _observer = observer;
    NSPointerFunctionsOptions keyOptions = retainObserved ? NSPointerFunctionsStrongMemory|NSPointerFunctionsObjectPointerPersonality : NSPointerFunctionsWeakMemory|NSPointerFunctionsObjectPointerPersonality;
    _objectInfosMap = [[NSMapTable alloc] initWithKeyOptions:keyOptions valueOptions:NSPointerFunctionsStrongMemory|NSPointerFunctionsObjectPersonality capacity:0];
  }
  return self;
}
```
这边定义了一个`_objectInfosMap`: key为被观察的对象, value则为存放着`_FBKVOInfo`的NSMutableSet, 这边在初始化的时候增加了retainObserved变量用来标记是否将key强引用，具体调用示例如下:
```
[self.KVOController observe:self.photos                        
                    keyPath:@"count"
                    options:NSKeyValueObservingOptionNew
                      block:^(id observer, id object, NSDictionary *change) {
    // observer -> RJViewController -> __weak self
    // object   -> self.photos    
    // change   -> NSKeyValueChangeKey + FBKVONotificationKeyPathKey    
}];
```
默认情况下对key(被观察的对象)也就是这边的self.photos做强引用，但是如果我们observe的对象为self本身，那么是不能做强引用持有的，否则就循环引用了。所以我们在这个情况下需要retainObserved传入NO, 这也是为什么NSObject+FBKVOController会有一个KVOController和KVOControllerNonRetaining了。

至于`_objectInfoMap`的value, 因为是NSMutableSet所以直接采用默认的强引用持有，这边大家或许有疑问，为什么value值又采用了NSMutableSet而不用NSHashTable呢？原因是这里并不需要弱引用持有各个`_FBKVOInfo`对象，而且大部分情况下使用NSMutableSet更加方便，比如NSHashTable就没有`enumerateObjectsUsingBlock`的枚举方法。而NSSet相比较NSArray的区别是前者无序存放，且使用hash查找元素效率快，但是后者比前者添加元素的速度快很多，所以在选择使用哪个容器的时候需要根据具体情况来选择。

FBKVO对`_FBKVOInfo`的hash和isEqual进行了重写，以keyPath来进行判等。所以这边对于value设置为`NSPointerFunctionsObjectPersonality`以hash/isEqual进行判断，而key值(被观察者)则设置为`NSPointerFunctionsObjectPointerPersonality`直接以指针地址做判断。

最后我们直接引用[Mattt](http://nshipster.com/nshashtable-and-nsmaptable/)大神对于什么时候用NSMapTable什么时候用NSDictionary的说明来结束这一小节:

> As always, it's important to remember that programming is not about being clever: always approach a problem from the highest viable level of abstraction. NSSet and NSDictionary are great classes. For 99% of problems, they are undoubtedly the correct tool for the job. If, however, your problem has any of the particular memory management constraints described above, then NSHashTable & NSMapTable may be worth a look.

###循环引用
我们知道，使用block的时候极易出现循环引用，通常使用方需要在block内部自己声明一个weak化的self来避免这个问题。那有没有办法省去这一步呢？是的，FBKVO在接口设计的时候也考虑到了这个问题，解决方法是在block回调接口增加一个`observer`参数，而这个`observer`在FBKVOController内部做了weak化处理，在上面示例中，`id observer`就是观察者(即`RJViewController *observer`)，也就是初始化接口时传入的self, 所以在block内部直接使用`[observer doSomething]`来代替`[self doSomething]`即可避免循环引用的问题。

FBKVO避免循环引用的设计确实很精致，我们来接着看下面这个情况:

```
[self.KVOController observe:self.photos                        
                    keyPath:@"count"
                    options:NSKeyValueObservingOptionNew
                      block:^(id observer, id object, NSDictionary *change) {
    // observer -> RJViewController -> __weak self
    // object   -> self.photos    
    // [self doSomething] -> [observer doSomething]
    [self.KVOController unobserve:self.photos keyPath:@"count"]
}];
```
这里在block里面用`self`调用了`unobserve`方法，按照我们之前的理解，那这边肯定会出现循环引用了，应该改成：
```
[observer.KVOController unobserve:observer.photos keyPath:@"count"]
```
但事实是在这个情况下，即使用`self`也不会引起循环引用，这是为什么呢？原因是做了`unobserve`操作后，存储KVO信息的`_FBKVOInfo`会被释放掉，这样它所指向的当前这个block也会被置为`nil`, 这样block引用`self`这个链端就被打破了，也就不会出现循环引用的问题了。所以打破循环引用除了在block内使用`weakSelf`外，也可以在事件处理完后将当前的block置为`nil`来实现。

###线程锁
FBKVO使用`pthread_mutex_t`作为线程锁，关于iOS各种线程锁的介绍可以参看[这篇](https://bestswifter.com/ios-lock/)博文。我们直接来看下FBKVO使用的其中一个地方:
```
- (void)_unobserveAll
{
  // lock
  pthread_mutex_lock(&_lock);

  NSMapTable *objectInfoMaps = [_objectInfosMap copy];

  // clear table and map
  [_objectInfosMap removeAllObjects];

  // unlock
  pthread_mutex_unlock(&_lock);

  _FBKVOSharedController *shareController = [_FBKVOSharedController sharedController];

  for (id object in objectInfoMaps) {
    // unobserve each registered object and infos
    NSSet *infos = [objectInfoMaps objectForKey:object];
    [shareController unobserve:object infos:infos];
  }
}
```
锁的基本原则是所有对**公共数据**处理的地方都需要加锁，上面的代码中`_objectInfosMap`为全局的NSMapTable, 对其修改操作(Add/Remove)都需要加锁, FBKVO这边的操作很值得借鉴，先拷贝一份临时变量，然后将`_objectInfosMap`清空，这一步在锁里面操作，之后被拷贝出的那份非全局或者说非共享变量再去做相应的后续操作。

这样的话即使有多个线程访问`_unobserveAll`也不会有任何问题，因为只有第一个线程会访问到`_objectInfosMap `, 第二个线程等解锁后再去访问时`_objectInfosMap`已经为空了，拷贝的对象`objectInfoMaps`也自然为空，所以锁并不需要加满整个`_unobserveAll `函数范围。

如果需要使用互斥类型的`pthread_mutex_t`锁，比如在递归函数中加锁，那需要将`pthread_mutex_t`初始化为互斥类型:
```
pthread_mutexattr_t attr;
pthread_mutexattr_init(&attr);
pthread_mutexattr_settype(&attr, PTHREAD_MUTEX_RECURSIVE);
pthread_mutex_init(&_mutex, &attr);
pthread_mutexattr_destroy(&attr);
```
锁使用完后记得要在dealloc里面销毁掉:
```
- (void)dealloc {
    pthread_mutex_destroy(&_mutex);
}
```

###DEBUG描述
DEBUG描述(debugDescription)其实和description是一样的效果，只是debugDescription是在Xcode控制台里使用po命令的时候调用显示的。如果没有实现debugDescription方法，那么打印该对象的时候仅仅显示内存地址，而不会显示该对象的各个属性值。
```
- (NSString *)debugDescription
{
  NSMutableString *s = [NSMutableString stringWithFormat:@"<%@:%p keyPath:%@", NSStringFromClass([self class]), self, _keyPath];
  if (0 != _options) {
    [s appendFormat:@" options:%@", describe_options(_options)];
  }
  if (NULL != _action) {
    [s appendFormat:@" action:%@", NSStringFromSelector(_action)];
  }
  if (NULL != _context) {
    [s appendFormat:@" context:%p", _context];
  }
  if (NULL != _block) {
    [s appendFormat:@" block:%p", _block];
  }
  [s appendString:@">"];
  return s;
}
```
上面是FBKVO实现的`_FBKVOInfo`的debugDescription, 将各个属性值拼接成字符串显示出来。那如果某个对象有N多个属性，这样一个个拼接会非常繁琐，这种情况下可以采用runtime来动态获取属性并返回:
```
- (NSString *)debugDescription // prefer super class
{ 
    NSMutableDictionary *dictionary = [NSMutableDictionary dictionary];
    
    // fetch class's all properties
    uint count;
    objc_property_t *properties = class_copyPropertyList([self class], &count);
    
    // loop to get each property via KVC
    for (int i = 0; i<count; i++) {
        objc_property_t property = properties[I];
        NSString *name = @(property_getName(property));
        id value = [self valueForKey:name]?:@"nil"; // default nil string
        [dictionary setObject:value forKey:name]; // add to dicionary
    }
    free(properties);
    
    return [NSString stringWithFormat:@"<%@-%p> -- %@",[self class],self,dictionary];
}
```

###数据结构

KVOController是框架的对外接口类，作为KVO的管理者，其持有了当前观察者对象和被观察者的KVO信息。 观察者对象以`weak`属性存储在`_observer`中，而`_objectInfosMap`中将被观察者以key进行存储, value则存储了对应的`_ FBKVOInfo`集合(图片引用自Draveness的[博文](https://www.jianshu.com/p/4c0c36b88db6))。   

![KVOController](http://upload-images.jianshu.io/upload_images/268805-132e5ccc0573a91f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

FBKVO为每一个被observe的对象都生成了一个`_ FBKVOInfo`对象，该对象存储了所有与KVO相关的信息，包括路径，回调等等。

![FBKVOInfo](http://upload-images.jianshu.io/upload_images/268805-4c1e08583227bf0c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

FBKVO的调用流程如下图所示, FBKVOController的作用只是添加相应的被观察者记录，以及生成相应的FBKVOInfo信息，最终会由FBKVOSharedController 这个单例来调用系统KVO方法实现对属性的监听，并且在回调方法中将事件分发给 KVO 的观察者。

![调用流程](http://upload-images.jianshu.io/upload_images/268805-95cb5bd35ae30051.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

FBKVO中还有一个比较有意思的地方是用`_`来区分内部接口和外部接口:
```
- (void)_unobserve:(id)object info:(_FBKVOInfo *)info // -> private method
- (void)unobserve:(nullable id)object keyPath:(NSString *)keyPath // -> public method
```
包括类名也是如此:
```
@interface FBKVOController : NSObject        // -> public 
@interface _FBKVOInfo : NSObject             // -> internal 
@interface _FBKVOSharedController : NSObject // -> internal 
```

FBKVO的代码量虽然不多，但其框架流程，接口设计和代码中使用到的细微技术点确实极具水平，希望本文总结和提炼的各种姿势可以让大家有一些收获，也欢迎大家留言讨论。完。