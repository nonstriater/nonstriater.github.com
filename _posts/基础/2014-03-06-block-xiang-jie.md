---
layout: post
title: "Block详解"
description: ""
category: "default"
tags: []
---

###Block是什么

>block 是“带有自动变量值的匿名函数”。所谓匿名函数也就是不带名称的函数。

  
感觉有点空，先看看《代码的未来》作者松本行弘 对“闭包”的解释：
  
  
>先理解函数对象，顾名思义，就是作为对象来使用的函数(不是面向对象中指的对象)。在C语言中，函数对象就是值函数指针。

>函数对象，也就是将函数作为值来利用的方法。那函数对象对我们编程有什么用？其最大的用途就是*高阶函数*。高阶函数就是把一个另一个函数作为参数或返回值的函数。

>设想对数组排序：

>    void sort(int *a , size_t   size)   


>这个函数有几个缺点：1 只能对整数排序 ;  2 排序条件无法从外部指定

>下面是C标准库中的排序函数：  

>    void qsort( void *ptr, size_t count, size_t size,int (*comp)(const void *, const void *) );

>qsort 就是通过将另一个函数作为参数使用，来实现通用排序的功能。高阶函数这样的方式，将一部分处理以函数对象的形式转移到外部。

>那么函数指针有哪些局限呢？

>  1. 函数指针的函数体要远离使用函数指针的地方单独定义
>  2. 无法访问外部的局部变量


>而”闭包“就是要解决函数指针的局限性。


故事到此为止，我们来看看Objective-C中的block是怎么解决C函数指针的局限性的。
  
  

### Block基本语法

先来熟悉下block的使用

#### block变量

```OC
int (^block)(int,int) // 声明block变量
int (^block(int,int)) = ^(int a,int b){return a+b;};  // block变量赋值 
```

#### block变量作为函数参数
```OC
void func(int(^block)(int));
```

#### block调用
```OC
block(1,2);
```

#### typedef定义一个block类型

```OC
typedef  int  (^SLArrayBlock)(NSArray *array,BOOL hasNextPage);
```


#### 使用C语言数组时，要使用其指针

这样使用会出现编译错误
```OC
const char text[] = "hello";
void (^blk)(void) = ^{printf("%c\n",text[2]);};
```

下面这样就没有问题
```OC
const char *text = "hello";
void (^blk)(void) = ^{printf("%c\n",text[2]);};
```


### Block 原理

支持block的编译器，首先将block语法转换为C语言编译器能编译的代码，然后再编译。

clang 编译器 -rewrite-objc选项，将源block语法转换为C源代码。


     clang -rewrite-objc 源文件   // (得到一个cpp文件)


通过源码，我们来看看block是如何实现：

1.  截获局部变量: 包括基本类型和对象,static和全局变量
2.  使用__block关键字完成对局部变量的修改
3.  block 内存管理
4.  block 的 循环引用以及解决方案



#### 测试1  了解block 的本质

```C
int main(){

void (^blk)(void) = ^{printf("block\n");};
blk();

return 0;
}
```


看main函数：

```C
int main(){

void (*blk)(void) = (void (*)())&__main_block_impl_0((void *)__main_block_func_0, &__main_block_desc_0_DATA);
((void (*)(__block_impl *))((__block_impl *)blk)->FuncPtr)((__block_impl *)blk);

return 0;
}
```

block 变量blk被编译器转换成 函数指针blk，并被强制转换后赋值为 __main_block_impl_0(这里是结构体的构造函数)，也就是说Block被作为C函数指针来处理的，函数指针; 


我们接着看下__main_block_impl_0定义

```C
struct __main_block_impl_0 {
struct __block_impl impl;
struct __main_block_desc_0* Desc;
__main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, int flags=0) {
impl.isa = &_NSConcreteStackBlock;
impl.Flags = flags;
impl.FuncPtr = fp;
Desc = desc;
}
};
```

我们看到__main_block_impl_0是一个结构体，我们逐步来解析它的结构:


1） 首先，它有一个构造方法：
```C
__main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, int flags=0)
```
第一个参数是函数指针，赋值为__main_block_func_0，我们在翻译中间代码里可以找到函数__main_block_func_0的定义
```C
static void __main_block_func_0(struct __main_block_impl_0 *__cself) {
printf("block\n");}
```
也就是说，这个指针就是指向block块的实现。注意这里函数的参数__cself是一个执行自身的struct __main_block_impl_0类型，也就是说，通过__cself, __main_block_func_0这个函数里面可以访问到__main_block_impl_0的所有信息，这在下面测试的截获局部变量至关重要。


第二个参数是一个__main_block_desc_0结构体类型变量的指针，看看__main_block_desc_0结构体类型定义：
```C
static struct __main_block_desc_0 {
size_t reserved;
size_t Block_size;
} __main_block_desc_0_DATA = { 0, sizeof(struct __main_block_impl_0)};
```

这个结构体基本上就是保存__main_block_impl_0结构体的大小信息。(__main_block_impl_0结构体类型的定义看上面)


通过上面的构造函数，完成了成员impl(struct __block_impl)和Desc指针(__main_block_desc_0类型)的赋值

2） 结构体__block_impl ，这个是重点哦~

```C
struct __block_impl {
void *isa; // 
int Flags;  
int Reserved;
void *FuncPtr;  // 函数指针
};
```


isa = &_NSConcreteStackBlock;

_NSConcreteStackBlock 相当于class_t结构体实例，class_t结构体描写了一个类的信息。在将Block作为OC对象处理时，关于类的信息放在_NSConcreteStackBlock实例中。

Block的实质，即为Objective-C对象，也可以说是一个结构体实例(对象的本质也是一个结构体实例)


#### 测试二： 截获局部变量

我们分别来看看基本类型和Objective-C对象类型是如何被截获处理的。

测试代码
```C
int main(){

char c = 'a'; 
void (^blk)(void) = ^{printf("block: %c\n",c);};
blk();

return 0;
}
```


看中间代码，可以看到：结构体__main_block_impl_0还是原来的结构体，
```C
struct __main_block_impl_0 {
struct __block_impl impl;
struct __main_block_desc_0* Desc;
char c;
__main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, char _c, int flags=0) : c(_c) {
impl.isa = &_NSConcreteStackBlock;
impl.Flags = flags;
impl.FuncPtr = fp;
Desc = desc;
}
};
```

不对，多了一个char c的成员。__main_block_desc_0也没有变化,恩~基本还是原来的味道。


```C
void (*blk)(void) = (void (*)())&__main_block_impl_0((void *)__main_block_func_0, &__main_block_desc_0_DATA, c);
```

只是构造函数__main_block_impl_0多了一个参数c，通过*测试一*我们知道，第三个参数c是通过__main_block_impl_0()构造函数传递给了struct __block_impl中的flags成员。



我们再来看下__main_block_func_0()函数中的内容：
```C
static void __main_block_func_0(struct __main_block_impl_0 *__cself) {
char c = __cself->c; // bound by copy
printf("block: %c\n",c);}
```

我又尝试将char类型换成char*,int,int *,unsigned int,float,double,都是在结构体__main_block_impl_0中多了一个对应类型的成员。

到这里，我们明白了: **block对于基本类型的捕获，是直接通过值传递完成的。或者叫做栈拷贝**




#### 测试三： block中无法截获C语言数组


block对于C语言数组呢？我将代码做了点小改造

int a[10] = {1}; // const char text[] = "hellow";  也是一样

编译报错：

”cannot refer to declaration with an array type inside block“

这是因为block并没有实现对C语言数组的截获，可以通过指针完成在block中对数组的访问。



**由于Objective-C对象类型其实质也是一个结构体变量，我们可以推测block对OC对象的捕获也是通过栈拷贝的方式完成的。**



#### Block中修改局部变量

我们知道，block中是无法修改局部变量的值的，否则会报编译错误。要向在block中访问局部变量有2种方式：

1. 通过加static，将局部变量变为静态数据。这种方式改变了变量的存储位置。
2.  加 __block 修饰符


**测试四  block中修改static局部变量**


测试代码
```C
int main(){

static int a = 10;
void (^blk)(void) = ^{ a = 4;printf("block: %d\n", a);};
blk();

return 0;
}
```

翻译后的__main_block_impl_0结构如下：
```C
struct __main_block_impl_0 {
struct __block_impl impl;
struct __main_block_desc_0* Desc;
int *a;
__main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, int *_a, int flags=0) : a(_a) {
impl.isa = &_NSConcreteStackBlock;
impl.Flags = flags;
impl.FuncPtr = fp;
Desc = desc;
}
};
```

可以看到，静态类型变为了一个指针类型(int *a)。因此我们大致可以猜出：Block对静态数据的修改是通过传递变量指针完成的。事实上也确是如此。


**测试五  block中修改带"__block"局部变量**

__block说明符同C语言中的static,auto,register说明符类似，用于指定将变量值设置到那个存储区域中。__block用来指定Block中想要变更的局部变量。

测试代码
```C
int main(){

__block int a = 10;
void (^blk)(void) = ^{ a = 4;printf("block: %d\n", a);};
blk();

return 0;
}
```

感觉去看下中间代码，发现，这个时候我们代码发生了很多的变化，出现了很多陌生的关键字:__Block_byref_a_0,__attribute__,__blocks___Block_object_dispose..... wait!wait! 先深吸口气，表要紧张，已经到了问题的核心点了，过了这道坎，你就要攻克一大难点了。我们一步一步来分析。


首先，我们发现
```C
__block int a = 10;
```
翻译成了
```C
 __Block_byref_a_0 a = {(void*)0,(__Block_byref_a_0 *)&a, 0, sizeof(__Block_byref_a_0), 10};
```

也就是说__block 类型的局部变量会被转换成  __Block_byref_a_0类型的局部变量来处理。也就是说，__block修饰的变量会用一个__Block_byref_a_0类型来存储。


同事，我们发现__main_block_impl_0结构体中多了下面一句：
```c
__Block_byref_a_0 *a; // by ref
```
 来看看__Block_byref_a_0 类型的定义：
```C
struct __Block_byref_a_0 {
void *__isa;
__Block_byref_a_0 *__forwarding;
int __flags;
int __size;
int a;
};
```


看到 void *__isa 你应该条件反射知道: “这特么也是一个类啊!” 。这个测试中被置为 (void *)0

__forwarding 是一个指向同样类型对象的指针，这个测试中置为指向结构体a的指针，我们还不知道有什么用，先不管

int a: 这个就是我们block变量的值，存储在这里。

其他几个字段就不介绍了。



__main_block_impl_0() 构造函数也有变化：

```C

 __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, __Block_byref_a_0 *_a, int flags=0) : a(_a->__forwarding)

```



那么我们的__block局部边是如何被修改的呢？接着看__main_block_func_0()函数定义：

```C
static void __main_block_func_0(struct __main_block_impl_0 *__cself) {
__Block_byref_a_0 *a = __cself->a; // bound by ref
(a->__forwarding->a) = 4;printf("block: %d\n", (a->__forwarding->a));}
```


**基本上，也是通过指针完成的修改，只不过先是用结构体__Block_byref_a_0包装了局部变量数据，然后通过__forwarding指针来修改值。**


另外，__main_block_desc_0结构体中还新增了copy 和 dispose 方法，来修改变量的引用计数

```C
static struct __main_block_desc_0 {
size_t reserved;
size_t Block_size;
void (*copy)(struct __main_block_impl_0*, struct __main_block_impl_0*);
void (*dispose)(struct __main_block_impl_0*);
} __main_block_desc_0_DATA = { 0, sizeof(struct __main_block_impl_0), __main_block_copy_0, __main_block_dispose_0};
```


__main_block_copy_0 调用了block runtime _Block_object_assign()方法

__main_block_dispose_0 调用了_Block_object_dispose()方法


  
那为什么搞这么麻烦？为什么要有一个__forwarding指针呢？


### Block 内存管理前传


Block对象在内存中的位置分为3中类型：

* _NSConcreteStackBlock
* _NSConcreteGlobalBlock 
* _NSConcreteMallocBlock


_NSConcreteGlobalBlock:  在存储全局变量的地方使用block时，Block对应的类对象即为_NSConcreteGlobalBlock。

来一个栗子：
```C
void (^blk)(void) = ^{ printf("hellow，world");};

int main(){

//__block int a = 10;

blk();

return 0;
}
```

blk变量是全部变量时，这时候Block结构体成员变量isa值就是_NSConcreteGlobalBlock，也就是Block对象存储在数据区中
```OC
impl.isa = &_NSConcreteGlobalBlock;
```
**有必要提下得是，当block里面没有局部变量的时候会block会变为_NSConcreteGlobalBlock类型，且这只在ARC有效是才是这样。**




_NSConcreteMallocBlock：将Block复制copy到堆上时，就会给Block接头体成员变量isa赋值为_NSConcreteMallocBlock，即

```OC
impl.isa = &_NSConcreteMallocBlock;
```


为什么需要 在 block超出变量作用域 任然存在？

如需要返回一个blk变量，这种情况下，是通过将block赋值到堆上来完成的。这个时候栈上的block超出作用域释放，但认可访问堆上的block对象。而堆上的__block变量可以通过__forwording指针完成。



如何将block从栈上赋值到堆上呢？

block中提供了专门的方法来完成这一任务：
```OC
_Block_copy()
```



### block内存管理


#### 非ARC下block得内存管理

1. 在非ARC下，向外传递block的时候一定也要做到，传给外面一个在堆上的，autorelease的对象
2. block是一个特殊的对象，copy会将block从栈上放到堆上

```OC
return [[stackBlock copy] autorelease];
```




#### ARC下block得内存管理

开启ARC时，将只会有 NSConcreteGlobalBlock和 NSConcreteMallocBlock类型的block。原本的NSConcreteStackBlock的block会被NSConcreteMallocBlock类型的block替代。


1. 在ARC下，strong指针指向的block放到堆上。
2. arc下，weak指针指向的block在栈上
3. 在ARC环境下，当block作为函数返回值的时候，block会自动被移到堆上，并返回一个注册到autoreleasepool的对象。



### Block的循环引用


>在我们阅读一些开源项目时，常看到这样的写法：
>    __weak TMCache *weakSelf = strongSelf;



在block中使用附有__strong修饰符的对象类型(arc下,没有修饰符时默认就是__strong)的局部变量, 此时，block对象持有该对象，这时很容易出现循环引用。
```OC
@interface MyObject: NSObject{
    void(^)() blk;
    id  obj_;
}

- (id)init{
    self = [super init];
    blk = ^{NSLog(@"%@",self);}; // blk对象持有self对象，而self又持有blk对象，造成循环引用
    return self;
}
```

为避免循环引用，可以声明附有__weak修饰符的变量

```OC
- (id)init{
    self = [super init];
    id __weak weakSelf = self;
    blk = ^{NSLog(@"%@",weakSelf);}; 
    
}
```

以下代码虽然没有使用self，但同样截获了self：
```OC
- (id)init{
    self = [super init];
    blk = ^{NSLog(@"%@",obj_);}; // 
      return self;   
}
```

因为对编译器来说，obj_只不过是结构体的一个成员变量。等价于：

     blk = ^{NSLog(@"%@",self ->obj_);};

解除循环引用的方法跟上面一样。


再比如

```C
ASIHTTPRequest *request = [ASIHTTPRequest requestWithURL:url];
[request setCompletionBlock:^{ NSString* string = [request responseString]; }];
``` 

也是request对象和block对象之间相互循环引用了。


另外，在GCD中，我们不需要为考虑循环引用的问题。

因为将Block作为参数传给GCD时，系统会将Block拷贝到堆上，如果Block中使用了实例变量，还将retain self，因为dispatch_async并不知道self会在什么时候被释放，为了确保系统调度执行Block中的任务时self没有被意外释放掉，dispatch_async必须自己retain一次self，任务完成后再release self。





### 什么时候使用block？

block的好处：
1. 一部分逻辑由外部指定，增加算法的通用性
2. 截获局部变量，简化代码，优化代码结构

因此，在能利用上block这些优势的地方都可以使用block。集合遍历，动画，网络状态回调等等。



### 参考

《Objective-C高级编程 iOS与OS X多线程和内存管理》

<http://tanqisen.github.io/blog/2013/04/19/gcd-block-cycle-retain/>

<http://www.cnblogs.com/biosli/archive/2013/05/29/iOS_Objective-C_Block.html>

 




