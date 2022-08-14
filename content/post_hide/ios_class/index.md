---
title: 格物致知iOS类与对象
subtitle: ""

# Summary for listings and search engines
summary: 在编程中我们接触最多的也是最基本的就是类和对象，当我们在创建类或者实例化对象时，是否考虑过类和对象到底是什么？理解其本质才能真正掌握一门语言。本文将从结构类型角度并结合实际应用探讨下Objective-C的类和对象。

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
- 类
- 对象

categories:
- iOS
- 类
- 对象
---

>欲诚其意者，先致其知；致知在格物。物格而后知至，知至而后意诚。现代汉语词典中将格物致知解释为: "推究事物的原理，从而获得知识"。

在编程中我们接触最多的也是最基本的就是类和对象，当我们在创建类或者实例化对象时，是否考虑过类和对象到底是什么？理解其本质才能真正掌握一门语言。本文将从结构类型角度并结合实际应用探讨下Objective-C的类和对象。

在Objective-C中，对象是广义的概念，类也是对象，所以严谨的说法应该是类对象和实例对象。既然实例对象所属的类称为类对象，那类对象有所属的类吗？有，称之为元类(Metaclass)。

### 类对象

类对象(Class)是由程序员定义并在运行时由编译器创建的，它没有自己的实例变量，这里需要注意的是类的成员变量和实例方法列表是属于实例对象的，但其存储于类对象当中的。我们在`/usr/include/objc/objc.h`下看看`Class`的定义:

```objc
/// An opaque type that represents an Objective-C class.
typedef struct objc_class *Class;
```
可以看到类是由`Class`类型来表示的，它是一个`objc_class`结构类型的指针。我们接着来看`objc_class`结构体的定义:
```objc
struct objc_class {
    Class                      isa;           // 指向所属类的指针(_Nonnull)
    Class                      super_class;   // 父类                  
    const char                *name;          // 类名(_Nonnull)
    long                       version;       // 类的版本信息(默认为0)
    long                       info;          // 类信息(供运行期使用的一些位标识)
    long                       instance_size; // 该类的实例变量大小
    struct objc_ivar_list     *ivars;         // 该类的成员变量链表
    struct objc_method_list * *methodLists;   // 方法定义的链表
    struct objc_cache         *cache;         // 方法缓存
    struct objc_protocol_list *protocols;     // 协议链表
};
```

- isa指针是和`Class`同类型的`objc_class`结构指针，类对象的指针指向其所属的类，即元类。元类中存储着类对象的类方法，当访问某个类的类方法时会通过该isa指针从元类中寻找方法对应的函数指针

- super_class为该类所继承的父类对象，如果该类已经是最顶层的根类(如`NSObject`或`NSProxy`), 则 super_class为`NULL`

- ivars是一个指向`objc_ivar_list`类型的指针，用来存储每一个实例变量的地址

- info为运行期使用的一些位标识，比如:
`CLS_CLASS (0x1L)`表示该类为普通类, `CLS_META (0x2L)`则表示该类为元类

- methodLists用来存放方法列表，根据info中的标识信息，当该类为普通类时，存储的方法为实例方法；如果是元类则存储的类方法

- cache用于缓存最近使用的方法。系统在调用方法时会先去cache中查找，在没有查找到时才会去methodLists中遍历获取需要的方法

### 实例对象

实例对象是我们对类对象`alloc`或者`new`操作时所创建的，在这个过程中会拷贝实例所属的类的成员变量，但并不拷贝类定义的方法。调用实例方法时，系统会根据实例的isa指针去类的方法列表及父类的方法列表中寻找与消息对应的selector指向的方法。同样的，我们也来看下其定义:

```objc
/// Represents an instance of a class.
struct objc_object {
    Class _Nonnull isa  OBJC_ISA_AVAILABILITY;
};
```

可以看到，这个结构体只有一个isa变量，指向实例对象所属的类。任何带有以指针开始并指向类结构的结构都可以被视作`objc_object`, 对象最重要的特点是可以给其发送消息. NSObject类的`alloc`和`allocWithZone:`方法使用函数`class_createInstance`来创建`objc_object`数据结构。

另外我们常见的id类型，它是一个`objc_object`结构类型的指针。该类型的对象可以转换为任何一种对象，类似于C语言中`void *`指针类型的作用。其定义如下所示:

```objc
/// A pointer to an instance of a class.
typedef struct objc_object *id;
```

### 元类对象

元类(Metaclass)就是类对象的类，每个类都有自己的元类，也就是`objc_class`结构体里面isa指针所指向的类. Objective-C的类方法是使用元类的根本原因，因为其中存储着对应的类对象调用的方法即类方法。

![类存储示意图.png](http://upload-images.jianshu.io/upload_images/268805-97f49657e1039185.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

所以由上图可以看到，在给实例对象或类对象发送消息时，寻找方法列表的规则为:

- 当发送消息给实例对象时，消息是在寻找这个对象的类的方法列表(实例方法)
- 当发送消息给类对象时，消息是在寻找这个类的元类的方法列表(类方法)

元类，就像之前的类一样，它也是一个对象，也可以调用它的方法。所以这就意味着它必须也有一个类。所有的元类都使用根元类作为他们的类。比如所有NSObject的子类的元类都会以NSObject的元类作为他们的类。

根据这个规则，所有的元类使用根元类作为他们的类，根元类的元类则就是它自己。也就是说基类的元类的isa指针指向他自己。

***

我们可以通过代码来实际验证下, Runtime提供了`object_getClass`函数:
```objc
Class _Nullable object_getClass(id _Nullable obj) 
```
来获取对象所属的类，看到这个函数你也许会好奇这个和我们平常接触的NSObject的`[obj class]`有什么区别？
```objc
// NSObject.h
- (Class)class;
+ (Class)class;
```
我们继续从runtime的[源码](https://opensource.apple.com//source/objc4/objc4-680/runtime/)里面寻找答案:
```objc
Class object_getClass(id obj) {
    return _object_getClass(obj);
}
```
`object_getClass`实际调用的是`_object_getClass `函数，我们接着看其实现:
```objc
static inline Class _object_getClass(id obj) {
    #if SUPPORT_TAGGED_POINTERS
    if （OBJ_IS_TAGGED_PTR(obj)）{
        uint8_t slotNumber = ((uint8_t)(uint64_t) obj) & 0x0F;
        Class isa = _objc_tagged_isa_table[slotNumber];
        return isa;
    }
    #endif
        if (obj) return obj->isa;
        else return Nil;
}
```
显然`_object_getClass`函数就是返回对象的isa指针，也就是返回该对象所指向的所属类。我们接着看`[obj class]`的具体实现(包括类方法和实例方法两种):
```objc
+ (Class)class {
    return self; // 返回自身指针
}

- (Class)class {
    return object_getClass(self); // 调用'object_getClass'返回isa指针
}
```
从代码中可以看出`+ (Class)class`返回的是其本身，而`- (Class)class`则等价于`object_getClass `函数。

我们来写个测试代码，看看这些函数的实际返回值是否和上面的所述保持一致，比如我们有个RJObject继承与NSObject：
```objc
RJObject *obj = [RJObject new];

Class clsClass0 = [RJObject class];     // 返回RJObject类对象的本身的地址
Class objClass0 = [obj class];          // isa指向的RJObject类对象的地址
Class ogcClass0 = object_getClass(obj); // isa指向的RJObject类对象的地址

NSLog(@"clsClass0 -> %p", clsClass0); // -> 0x10fb22068
NSLog(@"objClass0 -> %p", objClass0); // -> 0x10fb22068
NSLog(@"ogcClass0 -> %p", ogcClass0); // -> 0x10fb22068
```
打印结果可以看出，当obj为实例变量时, `object_getClass(obj)`与`[obj class]`输出结果一致，均返回该对象的isa指针，即指向RJObject类对象的指针。而`[RJObject class]`则直接返回RJObject类对象本身的地址，所以与前面两者返回的地址相同。

```objc
// 'objClass0'为RJObject类对象(RJObject Class)
Class objClass1 = [objClass0 class];          // 返回RJObject类对象本身的地址
Class ogcClass1 = object_getClass(objClass0); // isa指向的RJObject元类的地址

NSLog(@"objClass1 -> %p", objClass1); // -> 0x10fb22068
NSLog(@"ogcClass1 -> %p", ogcClass1); // -> 0x10fb22040
```
此时`objClass0`为RJObject的类对象，所以类方法`[objClass0 class]`返回的`objClass1`为`self`, 即RJObject类对象本身的地址，故结果与上面的地址相同。而`ogcClass1`返回的为RJObject元类的地址。
```objc
// 'ogcClass1'为RJObject的元类(RJObject metaClass)
Class objClass2 = [ogcClass1 class];          // 返回RJObject元类对象的本身的地址
Class ogcClass2 = object_getClass(ogcClass1); // isa指向的RJObject元类的元类地址

NSLog(@"objClass2 -> %p", objClass2); // -> 0x10fb22040
NSLog(@"ogcClass2 -> %p", ogcClass2); // -> 0x110ad9e58
```
同理，这边`ogcClass2 `为RJObject元类的元类的地址，那问题来了，某个类它的元类的元类的是什么类呢？这样下去岂不是元类无穷尽了？擒贼先擒王，我们先来看看根类NSObject的元类和它元类的元类分别是什么:
```objc
Class rootMetaCls0 = object_getClass([NSObject class]); // 返回NSObject元类(根元类)的地址
Class rootMetaCls1 = object_getClass(rootMetaCls0);     // 返回NSObject元类(根元类)的元类地址

NSLog(@"rootMetaCls0 -> %p", rootMetaCls0); // -> 0x110ad9e58
NSLog(@"rootMetaCls1 -> %p", rootMetaCls1); // -> 0x110ad9e58
```
看到结果就一目了然了，根元类的isa指针指向自己，也就是根元类的元类即其本身。另外，可以发现`ogcClass2`的地址和根元类isa的地址相同，说明任意元类的isa指针都指向根元类，这样就构成一个封闭的循环。

另外，我们可以通过`class_isMetaClass`函数来判断某个类是否是元类，比如:
```objc
NSLog(@"ogcClass0 is metaClass: %@", class_isMetaClass(objClass0) ? @"YES" : @"NO");
NSLog(@"ogcClass1 is metaClass: %@", class_isMetaClass(ogcClass1) ? @"YES" : @"NO");
```
输出结果为:
```objc
LearningClass[58516:3424874] ogcClass0 is metaClass: NO
LearningClass[58516:3424874] ogcClass1 is metaClass: YES
```
日志表明`ogcClass0 `为类对象，而`ogcClass1 `则为元类对象，这与我们上面的分析是一致的。

> 类和元类的父类指向情况也可以参照上面的步骤，通过`class_getSuperclass`或者`[obj superClass]`函数来获取分析，这边就不再赘述了。

***

除了isa声明了实例与所属类的关系，还有superClass表明了类和元类的继承关系，类对象和元类对象都有父类。同样，为了形成一个闭环，根类的父类为`nil`, 根元类的父类则指向其根类。我们可以通过一张示意图来看下三种对象之间的连接关系:

 ![类关系示意图](http://upload-images.jianshu.io/upload_images/268805-196560ee064edb09..jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

总结一下实例对象，类对象以及元类对象之间的isa指向和继承关系的规则为:

**规则一**: 实例对象的isa指向该类，类的isa指向元类(metaClass)

**规则二**: 类的superClass指向其父类，如果该类为根类则值为nil

**规则三**: 元类的isa指向根元类，如果该元类是根元类则指向自身

**规则四**: 元类的superClass指向父元类，若根元类则指向该根类

###动态创建类
Objective-C作为动态语言的优势在于它能在运行时创建类和对象，并向类中增加方法和实例变量。具体示例如下:

```objc
Class newClass = objc_allocateClassPair([NSObject class], "RJInfo", 0);

if (!class_addMethod(newClass, @selector(report), (IMP)ReportFunction, "v@:")) {
    NSLog(@"Add method 'report' failed!");
}
if (!class_addIvar(newClass, "_name", sizeof(NSString *), log2(sizeof(NSString *)), @encode(NSString *))) {
    NSLog(@"Add ivar '_name' failed!");
}

objc_registerClassPair(newClass);
```
上面代码创建了一个`RJInfo`的类，并分别添加了`_name`成员变量和`report`实例方法。需要注意的是，方法和变量必须在`objc_allocateClassPair`和`objc_registerClassPair`之间进行添加。所以，在运行时创建一个类只需要3个步骤:

首先是调用`objc_allocateClassPair`为新建的类分配内存，三个参数依次为newClass的父类，newClass的名称，第三个参数通常为0, 从这个函数名字可以看到新建的类是一个pair, 也就是成对的类，那为什么新建一个类会出现一对类呢？是的，元类！类和元类是成对出现的，每个类都有自己所属的元类，所以新建一个类需要同时创建类以及它的元类。

然后就可以向newClass中添加变量及方法了，注意若要添加类方法，需用`objc_getClass(newClass)`获取元类，然后向元类中添加类方法。因为示例方法是存储在类中的，而类方法则是存储在元类中。最后必须把newClass注册到运行时系统，否则系统是不能识别这个类的。

上面的代码中添加了一个成员变量`_name`, 我们来看下实际应用中如何获取和使用这个变量:
```objc
unsigned int varCount;

Ivar *varList = class_copyIvarList(newClass, &varCount);

for (int i = 0; i < varCount; i++) {
    NSLog(@"var name: %s", ivar_getName(varList[i]));
}

free(varList);

id infoInstance = [[newClass alloc] init];
Ivar nameIvar   = class_getInstanceVariable(newClass, "_name");

object_setIvar(infoInstance, nameIvar, @"Ryan Jin");

NSLog(@"var value: %@",object_getIvar(infoInstance, nameIvar));
```
我们可以通过`class_copyIvarList `来查看实例变量列表，注意获取的`varList`列表需要调用`free()`函数释放。当前只添加了一个变量，所以`varCount `为`1`, 在调用`ivar_getName`打印出变量的名字。如若对`_name `赋值，则需要先实例化newClass对象，并取出对象的该变量后调用`object_setIvar`进行赋值操作。示例代码的输出结果为:
```objc
LearningClass[58516:3424874] var name: _name
LearningClass[58516:3424874] var value: Ryan Jin
```
好了，验证完变量的添加，继续看方法的添加和使用。上文的示例中添加了`report`方法，但仅仅是做了`SEL`方法名的声明，我们来接着完成其`IMP`所指向函数`ReportFunction`的具体实现:
```objc
void ReportFunction(id self, SEL _cmd) {
    Class currentClass = [self class];
    Class metaClass    = objc_getMetaClass(class_getName(currentClass));
    
    NSLog(@"Class is %@, and super - %@.", currentClass, [self superclass]);
    NSLog(@"%@'s meta class is %p.", NSStringFromClass(currentClass), metaClass);
}
```
在函数实现中我们打印了类，父类以及元类的相关信息，为了运行`ReportFunction`, 我们需要创建一个动态实例来创建类的实例对象并调用`report`方法:
```objc
id instanceOfNewClass = [[newClass alloc] init];
    
[instanceOfNewClass performSelector:@selector(report)];
```
输出结果:
```objc
LearningClass[58516:3424874] Class is RJInfo, and super - NSObject.
LearningClass[58516:3424874] RJInfo's meta class is 0x600000253920.
```

除了给类添加方法，我们同样也可以动态修改已存在方法的实现，比如:
```
class_replaceMethod(newClass, @selector(report), (IMP)ReportReplacedFunction, "v@:");
```
这样就将`report`这个`SEL`所指向的`IMP`实现换成了`ReportReplacedFunction `. 如果类中不存在`name`指定的方法, `class_replaceMethod`则类似于class_addMethod函数一样会添加方法；如果类中已存在`name`指定的方法，则类似于`method_setImplementation`一样替代原方法的实现。

> 看到`class_replaceMethod`的解释，相信你已经发现了，这不就是Method Swizzling吗？没错，所谓的黑魔法，其实就是底层原理的应用而已！

### 本质探究
知其然亦知其所以然才是获取知识的正确方式，理解了类和对象的本质后，我们来看看格物致知后的理论可以引导出哪些应用和认识:

##### 属性
在Objective-C中，属性(property)和成员变量是不同的。那么，属性的本质是什么？它和成员变量之间有什么区别？简单来说属性是添加了存取方法的成员变量，也就是:
```objc
@property = ivar + getter + setter;
```
因此，我们每定义一个`@property`都会添加对应的`ivar`, `getter`和`setter`到类结构体`objc_class`中。具体来说，系统会在`objc_ivar_list`中添加一个成员变量的描述，然后在`methodLists`中分别添加`setter`和`getter`方法的描述。

#### 方法调用
如上文所述，方法调用是通过查询对象的isa指针所指向归属类中的`methodLists`来完成。这里我们通过[孙源](http://v.youku.com/v_show/id_XODIzMzI3NjAw.html)在runtime分享会上的一道题目来理解下。假设我们有一个类`RJSark`定义如下:
```objc
@interface RJSark : NSObject

- (void)speak;

@end
```
然后通过如下方式调用`speak`方法:
```objc
@implementation RJViewController

- (void)viewDidLoad 
{
    [super viewDidLoad];
    
    id cls    = [RJSark class];
    void *obj = &cls;
    
    [(__bridge id)obj speak];
}

@end
```
这里会正常完成调用，并不会导致程序crash. 这又是为什么呢？我们先来看下`cls`. 显然，它是`RJSark`的类对象，经过`void *obj = &cls`赋值后`obj`为指向`cls`的指针，再通过`(__bridge id)`将其转换为`id`对象。上文中我们提到`id`其实是一个`objc_object`结构体，里面存放了指向所属类的isa指针，所以调用`[obj speak]`能够找到它的isa所指向的类对象(也就是`RJSark`类)的方法列表并完成调用，但其实`obj`并不是`RJSark`的实例对象，它仅仅拥有和`RJSark`实例对象一样的isa指针而已。

空说无凭，我们将上面的代码稍微修改后验证下:
```objc
id cls       = [RJSark class];
RJSark *sark = [[cls alloc] init];
void *obj    = &cls;

NSLog(@"cls  = %p", cls);
NSLog(@"sark = %p", objc_getClass(object_getClassName(sark)));
NSLog(@"obj  = %p", objc_getClass(object_getClassName((__bridge id)obj)));
```
输出结果为:
```objc
LearningClass[58516:3424874] cls  = 0x10fbd02d0
LearningClass[58516:3424874] sark = 0x10fbd02d0
LearningClass[58516:3424874] obj  = 0x10fbd02d0
```
可以发现`obj`和`sark`的isa指针所指向的地址相同且与`cls`的地址一致，也就是它们都指向`cls`类对象。

##### 父类对象
我们直接来看一个面试题, `Father`继承与`NSObject`, `Son`则继承于`Father`类，分别调用`[self class]`和`[super class]`, 输出结果是？
```objc
@implementation Son : Father

- (instancetype)init
{
    self = [super init];
    if (self) {
        NSLog(@"%@", NSStringFromClass([self class]));
        NSLog(@"%@", NSStringFromClass([super class]));
    }
    return self;
}
@end
```
输出结果都为`Son`, 为什么`[super class]`的结果不是`Father`? 我们简单分析下就明白了。实例对象的方法列表是存放在isa所指向的类对象中的，所以调用`[self class]`的时候会去`self`的isa所指向的`Son`类对象中寻找该方法，在没有重载`[obj class]`的情况下, `Son`类对象是没有这个方法的，此时会接着在父类对象的方法列表中查找，最终会发现`NSObject`存储了该方法，所以`[self class]`会返回**实例对象(self)**所属的`Son`这个**类对象**。

而`[super class]`则指定从父类`Father`的方法列表开始去查找`- (Class)class`这个方法，显然`Father`没有这个方法，最终还是要查找到`NSObject`类对象的方法列表中，需要注意的是不管是`[self class]`还是`[super class]`, 它们都是调用的实例对象的`- (Class)class`方法，虽然其指向的类对象不同，但实例对象都是`self`本身，再强调下区分开实例对象和类对象！因而返回的也是当前`self`的isa所指向的`Son`类。

> 其实`super`是`objc_super`类型的结构体，它包含了当前的实例对象`self`以及父类的类对象。更详细的解答可以参考[@iOS程序犭袁](https://github.com/ChenYilong/iOSInterviewQuestions/blob/master/01《招聘一个靠谱的iOS》面试题参考答案/《招聘一个靠谱的iOS》面试题参考答案（上）.md#21-下面的代码输出什么)的博文。

***
除了用`super`来指向父类外，我们还可以用`isKindOfClass`和`isMemberOfClass`来判断对象的继承关系。这两个函数有什么区别呢？同样，先来看一个测试题:
```objc
BOOL r1 = [[NSObject class] isKindOfClass:[NSObject class]]; // -> YES
BOOL r2 = [[RJObject class] isKindOfClass:[RJObject class]]; // -> NO

BOOL r3 = [[NSObject class] isMemberOfClass:[NSObject class]]; // -> NO
BOOL r4 = [[RJObject class] isMemberOfClass:[RJObject class]]; // -> NO
```
为什么只有`r1`是`YES`? 实际上`isKindOfClass`是判断对象是否为`Class`的实例或子类，而`isMemberOfClass`则是判断对象是否为`Class`的实例。还是不明白？没关系，我们直接来看看这两个函数的源码实现，看看它们本质上是以什么作为判断标准的:

```objc
+ (BOOL)isKindOfClass:(Class)cls
{
    for (Class tcls = object_getClass((id)self); tcls; tcls = tcls->superclass) {
        if (tcls == cls) return YES;
    }
    return NO;
}

- (BOOL)isKindOfClass:(Class)cls 
{
    for (Class tcls = [self class]; tcls; tcls = tcls->superclass) {
        if (tcls == cls) return YES;
    }
    return NO;
}

+ (BOOL)isMemberOfClass:(Class)cls {
    return object_getClass((id)self) == cls;  
}

- (BOOL)isMemberOfClass:(Class)cls {
    return [self class] == cls; 
}
```
注意上面的题目是调用的类方法，所以我们分析下类方法的实现，至于实例方法也是类似的。可以看到`isMemberOfClass`的判断是先调用`object_getClass`获取isa所指向的归属类，也就是元类，然后直接判断`cls`是否就是被比较的对象的元类。而`[NSObject class]`的元类是根元类，显然不等于`[NSObject class]`本身，所以`r3`返回`NO`, `r4`也是同理。

而`isKindOfClass`也是先获取当前对象的元类，但是会循环获取其isa所指向类的父类进行比较，只要该元类或者元类的父类与`cls`相对则返回`YES`. `RJObject`的元类，以及父元类(最终指向根元类)都不等于`RJObject`对象，所以`r2`返回`NO`. 那为什么`r1`返回`YES`呢？还记得上文所说的闭环吗？根元类的父类指向根类本身！显然, `r1`符合了`isKindOfClass`的判断标准。

###学以致用
到这里理论部分就结束了。那么，问题来了，理解了类和对象的本质原理有什么实际应用价值吗？可以让我们更优雅的解决项目中遇到的问题和需求吗？Talk is cheap, show me the code:

比如App常见的记录用户行为的数据统计需求，俗称埋点。具体来说假设我们需要记录用户对按钮的点击。通常情况下，我们会在按钮的点击事件里面直接加上数据统计的代码，但这样做的问题在于会对业务代码进行侵入，且统计的代码散落各处，难以维护。

当然，我们还可以创建一个UIButton的子类，在子类中重载点击事件的响应函数，并在其中加上统计数据部分的代码:
```objc
-(void)sendAction:(SEL)action to:(id)target forEvent:(UIEvent *)event
```
这样做是可以的，但是现有工程中所有需要支持数据统计的按钮都必须替换成该子类，而且如果哪天不需要支持埋点功能了并需要迁移复用业务代码，那还得一个个再改回去。所以，我们需要一个更优雅的实现。

我们可以利用动态创建类并添加方法的思路来实现这个需求，这边只是以埋点作为示例，你也可以利用该思路扩展任意需要处理的需求和功能。简单来说就是我们创建一个UIButton的Category, 然后在需要埋点的情况下动态生成一个新的UIButton子类，并给其添加一个可以记录数据的事件响应方法来替代默认的方法，如下所示:

```objc
//
//  UIButton+Tracking.m
//  LearningClass
//
//  Created by Ryan Jin on 07/03/2018.
//  Copyright © 2018 ArcSoft. All rights reserved.
//

#import "UIButton+Tracking.h"
#import <objc/runtime.h>
#import <objc/message.h>

@implementation UIButton (Tracking)

- (void)enableEventTracking
{
    NSString *className = [NSString stringWithFormat:@"EventTracking_%@",self.class];
    Class kClass        = objc_getClass([className UTF8String]);
    
    if (!kClass) {
        kClass = objc_allocateClassPair([self class], [className UTF8String], 0);
    }
    SEL setterSelector  = NSSelectorFromString(@"sendAction:to:forEvent:");
    Method setterMethod = class_getInstanceMethod([self class], setterSelector);
    
    object_setClass(self, kClass); // 转换当前类从UIButton到新建的EventTracking_UIButton类
    
    const char *types   = method_getTypeEncoding(setterMethod);
    
    class_addMethod(kClass, setterSelector, (IMP)eventTracking_SendAction, types);
    
    objc_registerClassPair(kClass);
}

static void eventTracking_SendAction(id self, SEL _cmd, SEL action ,id target , UIEvent *event) {
    struct objc_super superclass = {
        .receiver    = self,
        .super_class = class_getSuperclass(object_getClass(self))
    };
    void (*objc_msgSendSuperCasted)(const void *, SEL, SEL, id, UIEvent *) = (void *)objc_msgSendSuper;
    
    // to do event tracking...
    NSLog(@"Click event record: target = %@, action = %@, event = %ld", target, NSStringFromSelector(action), (long)event.type);
    
    objc_msgSendSuperCasted(&superclass, _cmd, action, target, event);
}

@end
```

然后在添加按钮的地方，如果需要数据统计功能，则调用`enableEventTracking`函数来内嵌打点功能。使用示例如下:
```objc
- (void)viewDidLoad
{
    [super viewDidLoad];
    
    UIButton *button = [[UIButton alloc] initWithFrame:CGRectMake(0, 0, 50, 30)];
    
    button.layer.borderColor   = [[UIColor redColor] CGColor];
    button.layer.borderWidth   = 1.0f;
    button.layer.cornerRadius  = 4.0f;
    button.layer.masksToBounds = YES;
    
    [button addTarget:self action:@selector(trackingButtonAction:)
                 forControlEvents:UIControlEventTouchUpInside];
    
    [self.view addSubview:button];

    [button enableEventTracking];
}

- (void)trackingButtonAction:(UIButton *)sender
{
    // to do whatever you want...
    NSLog(@"%s", __func__);
}
```
打印输出信息为:
```objc
LearningClass[58516:3424874] Click event record: target = <ViewController: 0x7f97a5d0cb80>, action = trackingButtonAction:, event = 0
LearningClass[58516:3424874] -[ViewController trackingButtonAction:]
```

>浮于表面探究问题不失为一种方法，但是弄清楚本质才是真正意义上的解决疑惑。

###参考文章
1. [清晰理解Objective-C元类](http://blog.csdn.net/beclosedtomyheart/article/details/50164353)
2. [Objective-C Runtime 运行时之一: 类与对象](http://southpeak.github.io/2014/10/25/objective-c-runtime-1/)
3. [What is a meta-class in Objective-C?](http://www.cocoawithlove.com/2010/01/what-is-meta-class-in-objective-c.html)
4. [一只iOS魔法师的土系魔法讲义](https://juejin.im/post/5a951a1c6fb9a0633f0e471e)