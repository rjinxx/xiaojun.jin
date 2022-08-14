---
title: Hello, 宏定义魔法世界
subtitle: ""

# Summary for listings and search engines
summary: 宏，简单来说就是按预定义的规则来替换相应的文本内容，被替换的文本内容可以是对象也可以是函数。既然是替换，那就需要遵循一定的规则来执行，这里的规则就是本文要讨论的主要内容，希望通过深入浅出和逐层剖析的方式可以让大家对宏定义有更加透彻的理解，继而能够在实际项目中运用并发挥宏定义的魔法。

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
- 宏定义

categories:
- 宏定义
---

宏，简单来说就是按**预定义的规则**来**替换**相应的**文本内容**，被替换的文本内容可以是对象也可以是函数。既然是替换，那就需要遵循一定的规则来执行，这里的规则就是本文要讨论的主要内容，希望通过深入浅出和逐层剖析的方式可以让大家对宏定义有更加透彻的理解，继而能够在实际项目中运用并发挥宏定义的magic.

使用宏定义不仅可以让代码看起来更加简洁易读，更重要的是可以进行编译检查。由于宏定义是在预处理的时候被执行的，因此可以在编译之前就检查出包括参数类型，参数完整性等相关的错误。比如[ReactiveCocoa](https://github.com/ReactiveCocoa)里面的keypath(...)宏，可以接受可变参数：
```keypath(self.observerPath) ```或```keypath(self, observerPath)```
如果self没有observerPath这个成员变量，那么编译器会直接给出错误提示，避免等到运行时才发现路径无效而导致程序异常。
>宏定义分为**对象宏**和**函数宏**，对象宏通常是一些简单的对象替换，比如`#define M_PI 3.1415`, 函数宏（宏名字后加上括号）可以接受参数如函数调用一样在代码中使用，比如 `#define ADD(x,y) x + y`. 函数宏有两点需要注意: a). 括号与宏名之间不能有空格，否则就成对象宏了 b). 函数式宏定义的参数没有类型，预处理器只负责做形式上的替换，而不做参数类型检查，所以传参时要格外小心。 

Okay, 介绍完宏的基本概念之后，让我们正式进入宏定义的魔法世界吧！

###宏定义符号

在宏定义中，有四个特殊的符号：1)`\` 2)`#` 3)`##` 4) `...`，它们分别表示换行，字符串化，连接运算和可变多参数。

####1) 换行符
我们知道在字符串里面可以使用`\n`来实现换行，同样在宏定义里面也可以插入换行符而不影响其含义，只不过在宏里面是用反斜杠`\`来标识换行。
```
#define APP_VERSION() \
[[NSBundle mainBundle] objectForInfoDictionaryKey:@"CFBundleVersion"]
```
等价于：
```
#define APP_VERSION() [[NSBundle mainBundle] objectForInfoDictionaryKey:@"CFBundleVersion"]
```
如果不加`\`换行标识符而直接转行，则第二行的内容就不属于宏定义了, `APP_VERSION()`会被定义成空，也就是调用`APP_VERSION`会不起任何作用。
```
#define SWAP_VALUE(a, b) { \
            typeof(a) _t = a; \
            a =  b; \
            b = _t; \ 
        }
```
一般声明带有参数列表宏定义的时候，如果函数体字符串太长，通常都会使用换行符来增强函数的可读性。
>在宏声明中可以使用`typeof`来引用宏参数的类型。这个示例中typeof会获取参数a的类型，并用这个类型定义中间变量_t来交换a和b的值。因此，该宏可以交换所有基本数据类型的变量（整数，字符，结构等）

####2) 字符串化
单个`#`号的作用是字符串化，简单来说就是在输入值上加上`""`双引号，使其转换为C字符串。如果是在ObjC环境下，则可在头部再加上`@`符号，出来后即是一个`NSString`类型。
```
#define STRINGIZE_(x)  #x
#define STRINGIZE2(x)  STRINGIZE_(x)
#define OCNSSTRING(x) @STRINGIZE2(x)
```
我们带入实际值，比如`OCNSSTRING(3)`，一步步展开看一下：
```
OCNSSTRING(3) => @STRINGIZE2(3) => @STRINGIZE_(3) => @"3" // NSString @"3"
STRINGIZE2(3) =>  STRINGIZE_(3) // 这里多加一层是为了处理宏展开的问题，见后文介绍
STRINGIZE_(3) =>  #3 => "3" // C字符串"3"
```
>字符串化对[空格的处理](http://blog.csdn.net/jiangjingui2011/article/details/6706967)有两种情况: a). 忽略传入参数名前面和后面的空格 e.g. `STRINGIZE_(   abc ) => "abc"` b)当传入参数之间存在空格时，忽略其中多于一个的空格 e.g. `STRINGIZE_(abc /*多个空格*/ def) => "abc def"`

####3) 连接运算
连接符号`##`用来将前后两项合并成一个整体。它的执行分为两步，先是**分隔**，然后它会将合并项之间的空格去除后完成**连接**操作。
```
#define metamacro_concat(A, B) A ## B
NSString *str      = @"This is Ryan!";
NSLog(@"%@", metamacro_concat(st, r));  // This is Ryan!
```
`metamacro_concat`的作用是将`st`和`r`这两项合并成整体`str`，而`str`是字符串`@"This is Ryan!"`的对象，所以最终会打印出来该字符串内容。刚刚这个示例仅仅是演示了**连接**这个动作，那上面所说的**分隔**是什么意思或者说是什么场景呢？我们直接来看下面这个[示例](http://blog.csdn.net/lygoflying/article/details/49871873)：
```
#define A1(name, type) type name_##type##_type 
#define A2(name, type) type name##_##type##_type
```
带入实参`A1(a1, int)`和`A2(a1, int)`，相信你看到这两个宏定义后，会觉得没什么特别的，理解了`##`的含义，再逐个替换之后（`type`替换成`int`，`name`替换成`a1`，`##`前后的项连成整体），很快就能得出答案：
```
A1(a1, int) => int a1_int_int; // 然而并不对！！！
A2(a1, int) => int a1_int_int; // 然而并不对！！！
```
是的，参照后面注释，然而并不对！现在该是讨论`##`分隔操作的时候了，我们先来公布下正确答案，当然我建议你可以找个编译器实际试一下，看看结果到底是什么。
```
A1(a1, int) => int name_int_type; // bingo!
A2(a1, int) => int a1_int_type;   // bingo！
```
看到答案是不是很惊讶，为什么有的替换了，有的没替换，`name`不应该都替换成`a1`，`type`不应该都替换成`int`吗？好了，我们来揭开谜底吧：
>预处理器在解析宏的时候会先做分隔操作，就是把`##`的前后项分隔开。

1. 宏`A1`的`name_##type##_type `会被分隔成`name_`，`type`和`_type`这3段，显然`name` != `name_`;`type` != `_type`，所以第一段`name_`和第三段`_type`不会被宏替换，中间段`type`则被替换成`int`，按这个规则带入后就可以得到最终的结果为`int name_int_type`
2. 宏`A2`的`name##_##type##_type`会被分隔成`name`，`_`，`type`和`_type`这4段，现在就一目了然了，`_type`不会被替换，因此带入参数后得到最终结果为`int a1_int_type`

分隔的作用类似于空格。在宏定义中，预处理器一般把空格解释成分段标志，对于每一段和前面比较，相同的就被替换。而`##`则会把前后项之间的空格都去除，然后再做连接操作。所以`A1`和`A2`的定义也可如下：
```
#define A1(name, type)  type name_     ##    type     ##     _t1ype
#define A2(name, type)  type name   ##   _   ##   type   ##  _t1ype
```
>常见的运算符比如`+`,`-`, `*`, `/`, `++`以及宏定义操作符`#`和`...`也是分隔标志，e.g. `define add(a, b) a+b`和`define add(a, b) a    +    b`结果是一致的，`+`会把`a`和`b`之间的空格去掉后再去做相应运算
####4) 可变多参数
标识符号`...`用来标识该宏可以接收可变的参数数量（零个或多个符号）。在宏体中，使用`__VA_ARGS__ `来表示那些输入的实际参数，也就是`__VA_ARGS__`在预处理中将为实际的参数集所替换。需要注意的是`...`只能放在末尾，代替最后面的宏参数。
>同`...`与`__VA_ARGS__ `配对类似，也可以使用`NAME...`与`NAME`来配对使用表示可变多参数。不同于前者，这里的`NAME`是你任意的参数名，并不是系统保留名: e.g. `format...`和`format` 

我们来看一个实现对ObjC的NSLog打印信息补充和限制只在DEBUG环境下输出的可变参数宏定义示例：
```
#ifdef  DEBUG
#define NSLog(format, ...) NSLog((@"%s [Line %d] " format), __func__, __LINE__, ##__VA_ARGS__);
#else
#define NSLog(format, ...)
#endif
```
在这个宏定义中，如果是非DEBUG环境，那么直接替换为空，也就是NSLog将不起任何作用。我们重点讨论DEBUG环境下的定义，第一个参数`format`将被单独处理，接下来输入的参数则作为一个整体被视为可变参数。比如`NSLog(@"name = %@, age = %d", @"Ryan", 18)`, 这里的`@"name = %@, age = %d"`即对应宏里的`format`, 后面的`@"Ryan"`和`18`则映射为`...`指代为统一的可变参数。 因为我们不确定用户格式化字符串时会输入多少个参数，所以我们指定为可变参数，允许用户输入任意数量的参数。带入具体的实参后替换后的结果为：
```
NSLog((@"%s [Line %d] " "name = %@, age = %d"), __func__, __LINE__, ##@"Ryan", 18);
```
不知道你有没有注意到`__VA_ARGS__`前面的`##`标识符，有了上文的介绍，我们知道它是用来做连接操作的，也就是将`name = %@, age = %d`和前面的参数连接后打印出来。但是`__VA_ARGS__`本来就是顺着`__LINE__`后面写的，应该不需要加`##`吧？YES! 确实不需要加`##`来做"连接"的作用，那为什么还要加呢？

既然是可变多参数，那它是包括一个case的: 参数数量为0，如果我们把`##`去掉，替换后宏就变成如下结果:
```
NSLog((@"%s [Line %d] "), __func__, __LINE__, ); // 注意最后一个逗号
```
有没有发现，当可变参数的个数为0时，最后面会多一个逗号，显然这个情况下编译器会报错的，那怎么才能支持0参数的情况呢？答案就是`##`. 当可变参数的个数为0的时候，`##`会把前面多余的逗号去掉，所以定义可变参数宏需要记得加上`##`来处理这个情况。

###宏定义展开
当宏定义有多层嵌套的情况，即宏定义里面又包含另外的宏定义，这时宏展开（替换）需要遵循一定的规则，总体原则是**每次只解开当前层**的宏，我们直接来看下面这个示例：
```
#define  _ANONYMOUS1(type, var, line) type  var##line
#define  _ANONYMOUS0(type, line)      _ANONYMOUS1(type, _anonymous, line)
#define   ANONYMOUSS(type)            _ANONYMOUS0(type, __LINE__)
```
带入实参`ANONYMOUSS(static int);`即: `static int _anonymous70;` 70表示该行行号。这个宏包含三层，逐一解析：
```
第一层：ANONYMOUSS(static int) –> _ANONYMOUS0(static int, __LINE__)
第二层：                       –> _ANONYMOUS1(static int, _anonymous, 70);
第三层：                       –> static int _anonymous70;
```
由于每次只能解开当前层的宏，`__LINE__`需要等到第二层才能被解开。所以如果我们把中间层`_ANONYMOUS0`去掉，直接由`_ANONYMOUS1`来定义`ANONYMOUSS`：
```
#define  _ANONYMOUS1(type, var, line) type  var##line
#define   ANONYMOUSS(type)            _ANONYMOUS1(type, _anonymous, __LINE__)
```
再次带入实参`ANONYMOUSS(static int);`这个情况下，最终的结果会是`static int _anonymous__LINE__`，预定义宏```__LINE__```并不会被解开！所以当你看一些有嵌套宏定义的时候（包括系统的宏定义），你会发现它们往往会加多一层中间转换宏，加这层宏的用意是把所有宏的参数在这层里全部展开，这个我们在自己实际项目中定义复杂宏的时候也需要特别留意。
>这里用到了预定义宏`__LINE__`，预定义宏的行为是由编译器指定的。`__LINE__`返回展开该宏时在文件中的行数，其他类似的有`__FILE__`返回当前文件的绝对路径；`__func__`是该宏所在scope的函数名称；`__COUNTER__`在编译过程中将从0开始计数，每次被调用时加1。因为唯一性，所以很多时候被用来构造独立的变量名称。

宏展开的另外一个规则是，在展开当前宏函数时，如果形参有`#`或`##`则不进行宏参数的展开，否则先展开宏参数，再展开当前宏。我们来看一道经典的C语言[题目](http://blog.csdn.net/jiabinjlu/article/details/8037003)：
```
#include <stdio.h> 

#define f(a,b) a##b  
#define g(a)   #a  
#define h(a)   g(a)  

int main() {
    printf("%s\n", h(f(1,2))); // => 12
    printf("%s\n", g(f(1,2))); // => f(1,2)
    return 0;
}
```
这道题的正确答案是分别是`12`和`f(1,2)`，后者宏`g`里面的参数`f(1,2)`不会被展开。我们对照上面宏展开的规则来分析下：
第一行`h(f(1,2))`由于`h(a)`非`#`或者`##`所以先展开参数`f(1,2)`即`12`再展开当前宏`h(12)` => `g(12)` => `12`
第二行`g(f(1,2))`由于`g(a)`形参带有`#`所以里面的`f(1,2)`不会被展开，最终结果就是`f(1,2)`

相信你应该发现了，其实`h(a)`在这里充当的就是中间转换宏的角色，目的就是为了让`f(1,2)`先在`h(a)`里面被展开，避免放到`g(a)`里面遇到`#`而无法被替换。好了，了解了宏定义的展开规则，我们再留个小作业给大家：
```
#define VALUE            2 
#define STRINGIZES_(s)   #s 
#define COMBINATION(a,b) int(a##e##b) 

printf("int max: %s\n", STRINGIZES_(INT_MAX)); // => ?
printf("%s\n", COMBINATION(VALUE, VALUE));     // => ?           
```
应该怎样添加转换宏才能分别打印出`int max: 0x7fffffff`和`200`? P.S. `INT_MAX`的十六进制为`0x7fffffff`; `200`则等于`2e2`, `e`为指数表达式，表示`2`乘以`10`的`2`次方。

###宏实例分析
有了上面的介绍，我们可以选一些相对复杂的宏定义来分析了，这边我们还是选取[ReactiveCocoa](https://github.com/ReactiveCocoa)里面的两个宏。大家如果有兴趣，还是强烈推荐去GitHub下载这个库查看下，里面有很多让人叹为观止的宏定义。

####1) 计算参数个数
下面这个宏`metamacro_argcount(...)`用来计算在可变参数的情况下，传入的实参数量。e.g. `int num = metamacro_argcount(a, b, c);`等价于`int num = 3;` 作者提到灵感是来自于[P99](http://p99.gforge.inria.fr). 这里为了方便分析，我们把支持最多参数数量的计算改成10个且做了稍许[简化](http://liuyanwei.jumppo.com/2015/09/20/ios-macro.html)。
```
#define metamacro_argcount(...) metamacro_at(10, __VA_ARGS__,10, 9, 8, 7, 6, 5, 4, 3, 2, 1)
#define metamacro_at(N,...)     metamacro_concat_at##N(__VA_ARGS__)

#define metamacro_concat_at10(_0,_1,_2,_3,_4,_5,_6,_7,_8,_9,...) metamacro_head(__VA_ARGS__)

#define metamacro_head(...)             metamacro_head_first(__VA_ARGS__,0)
#define metamacro_head_first(first,...) first
```
看起来是不是感觉很复杂？没关系，我们一步步来，逐层带入参数来分析。假设我们传入5个参数`metamacro_argcount(a,b,c,d,e)`

#####STEP 1: 带入metamacro_argcount
```
metamacro_argcount(a,b,c,d,e) => metamacro_at(10, a,b,c,d,e,10,9,8,7,6,5,4,3,2,1)
```
这里的`__VA_ARGS__ `替换成前面传入的可变实参`a,b,c,d,e`

#####STEP 2: 带入metamacro_at
```
metamacro_at(10, a,b,c,d,e,10,9,8,7,6,5,4,3,2,1) => metamacro_concat_at10 (a,b,c,d,e,10,9,8,7,6,5,4,3,2,1)
```
第一个参数为`N`, 之后都定义为可变参数。故而`N`为`10`, `__VA_ARGS__` 为` a, b, c, d, e, 10, 9, 8, 7, 6, 5, 4, 3, 2, 1`, 这一步既修改了参数又修改了方法名。

#####STEP 3: 带入metamacro_concat_at10
```
metamacro_concat_at10 (a,b,c,d,e,10,9,8,7,6,5,4,3,2,1) => metamacro_head(5,4,3,2,1)
```
这里把前面十个参数替换成`_0,_1,_2,_3,_4,_5,_6,_7,_8,_9`, 然后之后的参数，也就是`5,4,3,2,1`定义为可变参数，并作为实参传给宏`metamacro_head`. 前10个参数就被drop掉了，所以你不用关心它是被替换成了`_0`还是`___0`, 总之它们不需要继续被使用了。

#####STEP 4: 带入metamacro_head
```
metamacro_head(5,4,3,2,1) => metamacro_head_first(5,4,3,2,1,0)
```
为什么在后面加个`0`呢？还记得前面说过的，可变参数的数量可以为零的吧，在这个场景下就变成`metamacro_head_first()`了，后面再用`metamacro_head_first`取第一个值就出错了，所以需要额外加个`0`, 这样可变参数为空的时候就变成`metamacro_head_first(0) `, 再取第一个值就可以得到参数数量为0了。

#####STEP 5: 带入metamacro_head_first
```
metamacro_head_first(5,4,3,2,1,0) => 5 // 直接获取第一个值，其他的省略
```
是不是很cool很magic? 通过几个宏定义的转换，我们就能轻易的得出传入的实参个数，而且这些结果在预处理阶段就获得了，不必等到运行阶段再去计算。

####2) 参数格式检查
ReactiveCocoa里面还有个非常精妙的宏`keypath(...)`, 可以判断输入的路径参数是否合法，并且给出代码提示。比如输入`keypath(self.path)`, 宏会作出判断`path`是否为`self`的`property`, 如果该`path`不存在，则给出警告，避免误写。而且这个宏支持可变参数，还可以输入格式为`keypath(self, path)`, 同样会对`path`做参数检查。是不是很神奇？让我们来揭开外衣看看它的魔法来源。
```
#define keypath(...) \
        metamacro_if_eq(1, metamacro_argcount(__VA_ARGS__))(keypath1(__VA_ARGS__))(keypath2(__VA_ARGS__))
#define keypath1(PATH) \
        (((void)(NO && ((void)PATH, NO)), strchr(# PATH, '.') + 1))
#define keypath2(OBJ, PATH) \
        (((void)(NO && ((void)OBJ.PATH, NO)), # PATH))
```
#####STEP 1: 带入metamacro_if_eq
```
metamacro_if_eq(1, metamacro_argcount(self,path))(keypath1(__VA_ARGS__))(keypath2(self,path))
```
这边的`metamacro_argcount`上面讨论过，是计算可变参数个数，所以`metamacro_if_eq`的作用就是判断参数个数，如果个数是`1`就执行后面的`keypath1`, 若不是1就执行`keypath2`, 我们来看看`metamacro_if_eq`的定义：
```
/**
 * If A is equal to B, the next argument list is expanded; otherwise, the
 * argument list after that is expanded. A and B must be numbers between zero
 * and twenty, inclusive. Additionally, B must be greater than or equal to A.
 *
 * @code

// expands to true
metamacro_if_eq(0, 0)(true)(false)

// expands to false
metamacro_if_eq(0, 1)(true)(false)

 * @endcode
 *
 * This is primarily useful when dealing with indexes and counts in
 * metaprogramming.
 */
#define metamacro_if_eq(A, B) \
        metamacro_concat(metamacro_if_eq, A)(B)
```
限于篇幅，这边就不展开太多了，只重点分析下`keypath(...)`宏的实现，至于`metamacro_if_eq`我们知道它的作用就可以了，不过我还是建议大家去ReactiveCocoa查看下这个宏的完整定义，并尝试分析下`metamacro_if_eq`的实现原理。我相信，通过本文的介绍再加上一步步的带入替换，应该不难理解它的实现。

#####STEP 2: 带入keypath2
```
keypath2(self,path) (((void)(NO && ((void)self.path, NO)), # path))
```
这个宏整体是一个C语言的逗号表达式，我们来回忆下逗号表达式的格式: e.g. `int a = (b, c);` 逗号表达式取后面的值，故而`a`将被赋值成`c`, 此时`b`在赋值运算中就被忽略了，没有被使用，所以编译器会给出警告，为了消除这个warning我们需要在`b`前面加上`(void)`做个类型强转操作。

逗号表达式的前项和NO进行了与操作，这个主要是为了让编译器忽略第一个值，因为我们真正赋值的是表达式后面的值。预编译的时候看见了NO, 就会很快的跳过判断条件。我猜你看到这儿肯定会奇怪了，既然要忽略，那为啥还要用个逗号表达式呢，直接赋值不就好了？

这里主要是对传入的第一个参数`OBJ`和第二个正要输入的`PATH`做了`.`操作，这也正是为什么输入第二个参数时编辑器会给出正确的代码提示（只要是作为表达式的一部分, Xcode自动会提示）。如果传入的path不是self的属性，那么self.path就不是一个合法的表达式，所以自然编译就不会通过了。

#####STEP 3: 带入keypath1
```
keypath1(self.path) (((void)(NO && ((void)self.path, NO)), strchr(# self.path, '.') + 1))
```
宏`keypath1 `接受1个参数，所以我们直接带入`self.path`. 宏的前半段和上面是一样的，不同的是逗号表达式的后一段`strchr(# self.path, '.') + 1`, 函数`strchar`是C语言中的函数，用来查找某字符在字符串中首次出现的位置，这里用来在`self.path`(注意前面加了`#`字符串化)中查找`.`出现的位置，再加上`1`就是返回`.`后面`path`的地址了。也就是`strchr('self.path', '.')`返回的是一个C字符串，这个字符串从找到`'self.path'`中为`'.'`的字符开始往后，即`'path'`.

按照上面的分析，我们知道`keypath(...)`是返回一个经过检查的合法路径。如果在ObjC环境下，我们需要的是一个`NSString`, 所以我们在调用这个宏的时候，再加上`@`符号就OK了, e.g. `@keypath(self.path) => @"self.path"`. 

>有时定义宏我们会故意加上`@`符号，但不是为了转换`NSString`类型，也不是为了某种特别的作用，只是让调用看起来更原生一些。

我们来看下面这个例子:
```
#define weakObj(obj) __weak typeof(obj) obj##Weak = obj;
```
在ObjC里面的block为了防止循环引用，我们会使用`__weak`关键字，这个宏就是用来实现obj的weak化，调用的时候则是`weakObj(self)`, 但是iOS都是习惯加`@`符号，比如字符串是`@""`, 数组是`@[]`, 就连定义协议都是`@protocol`, 那怎么让我们的`weakObj`也能在前面加上`@`符号呢？

iOS开发的同学应该都记得系统的自动释放池`@autoreleasepool{}`, 这里面就有个`@`符号，所以我们可以在`weakObj`的宏定义里面放一个空的`autoreleasepool{}`, 并且不加`@`符号，让这个`@`符号有外面调用的时候加上，也就是这样的:
```
#define weakObj(obj) autoreleasepool{} __weak typeof(obj) obj##Weak = obj;
```
调用的时候`@weakObj`里的`@`符号就被加到`autoreleasepool{}`上了，其实这个`autoreleasepool{}`是空的，并不起任何实际作用:
```
@weakObj(obj) => @autoreleasepool{} __weak typeof(obj) obj##Weak = obj;
```

###宏知识补充
由于宏定义的实质只是文本替换，所以这个并不智能的替换会在一些环境下发生不可预知的错误。幸运的是，我们的前辈们发现了这些问题，并且提供了很好的解决方案，这也是我们下面要讨论的许多宏定义约定俗成的格式写法。
####1) 使用do{}while(0)语句
对于函数宏，我们一般都是推荐使用`do{}while(0)`语句，把函数体包到`do`后面的`{}`内，为什么要这样呢？我们看一个实例:
```
#difne FOO(a,b) a+b; \
                a++;
```
正常调用是没有问题的，但是如果我们是在`if`条件语句里面调用，并且`if`语句没有`{}`, 像下面这样:
```
if (...)
   FOO(a,b) // 满足了if条件后FOO会被执行
```
展开之后就会变成（显然就不对了）:
```
if (...)
   a+b; // a+b在满足了if条件后会被执行
a++;    // a++不管if条件是否满足都会被执行
```
如果加上`do{}while(0)`语句展开后就是:
```
if (...)
   do {         
        a+b; 
        a++;                                                            
   } while (0);
```
这样就没有问题了，但你肯定会疑惑，这个和直接包一个`{}`不是一样的吗，只要把函数体包成一个整体就可以了。是的，在这个情况下是一样的，但是`do{}while(0)`还有一个功能，会去除多余的分号，我们还是看实例:
```
#difne FOO(a,b) { a+b; \
                  a++; }
if (...)
   FOO;
else
   ...
```
使用`{}`情况下我们展开来看:
```
if (...) {
    a+b; 
    a++; 
}; else // 注意这边多出来的分号，编译直接报错！
   ...
```
如果是`do{}while(0)`的话会直接吸收掉这个分号:
```
if (...) 
   do {
       a+b; 
       a++; 
   } while(0); // 分号被do{}while(0)吸收了
else {
   ...
}
```
这个吸收分号的方法现在已经几乎成了标准写法。而且因为绝大多数的编译器都能够识别`do{}while(0)`这种无用的循环并进行优化，所以不会因为这种方法导致运行效率上的差异。

####2) 使用({...})语句
GNU C里面有个`({...})`形式的[赋值扩展](https://onevcat.com/2014/01/black-magic-in-macro/)。这种形式的语句在顺次执行之后，会将最后一次的表达式的赋值作为返回。
```
#define MIN(A,B) ({ __typeof__(A) __a = (A); __typeof__(B) __b = (B); __a < __b ? __a : __b; })
```
这个宏用来取输入参数中较小的值，并将该值作为返回值返回。这里就是用到了`({...})`语句来实现，函数体中可以做任意的逻辑处理和运算，但最终的返回值则是最后的表达式。所以在定义宏的时候，我们可以用`({...})`语句来定义有返回值的函数宏，这个也是函数宏很常见的写法，大家在实际项目中也可以注意参照使用。

最后简单提下宏和`const`怎么区别使用，一般来说定义常量字符串就用const，定义代码就用宏（可以参见iOS的API相关定义）。如果有任何不清楚的，欢迎留言讨论。PS. 本文参考了很多前辈们的精彩文章，在文中以超链接的方式做了引用，感谢他们的分享，也希望本文能给大家带来一点帮助。