---
title: Ivar和类实例的内存分布
---

​	ivar是类中的成员变量，@propery声明一个属性的时候就会产生一个成员变量。

#### 获取类的成员变量

​	在runtime中，通过class_copyIvarList(Class cls, unsigned int * outCount)函数可以获取类的成员变量列表。在这里需要注意的是，class_copyIvarList函数获取的是当前类声明的成员变量，它的父类声明的 成员变量并不会返回。<!--more-->

```	c++
/***********************************************************************
* class_copyIvarList
* fixme
* Locking: read-locks runtimeLock
**********************************************************************/
Ivar *
class_copyIvarList(Class cls, unsigned int *outCount)
{
    const ivar_list_t *ivars;
    Ivar *result = nil;
    unsigned int count = 0;

    if (!cls) {
        if (outCount) *outCount = 0;
        return nil;
    }

    mutex_locker_t lock(runtimeLock);

    ASSERT(cls->isRealized());
    
    if ((ivars = cls->data()->ro->ivars)  &&  ivars->count) {
        result = (Ivar *)malloc((ivars->count+1) * sizeof(Ivar));
        
        for (auto& ivar : *ivars) {
            if (!ivar.offset) continue;  // anonymous bitfield
            result[count++] = &ivar;
        }
        result[count] = nil;
    }
    
    if (outCount) *outCount = count;
    return result;
}
```

​	从runtime源码中可以看到，成员变量列表是从*cls->data()->ro->ivars*中获得。	runtime源码中可以看到class在remap时从DATA Segment，__objc_classlist Section中读取来的，在class初始化之前的data返回的是ro，初始化完成之后才重新开辟一块内存作为rw，ro涵盖了类的所有初始信息初始化信息，类的成员变量、方法列表、协议等等。所以ivars在编译结束之后就是确定的。

```c++
GETSECT(_getObjc2ClassList,           classref_t const,      "__objc_classlist");
```

​	再来看一下ro的结构：

```c++
struct class_ro_t {
    uint32_t flags;
    uint32_t instanceStart;
    uint32_t instanceSize;
#ifdef __LP64__
    uint32_t reserved;
#endif

    const uint8_t * ivarLayout;
    
    const char * name;
    method_list_t * baseMethodList;
    protocol_list_t * baseProtocols;
    const ivar_list_t * ivars;

    const uint8_t * weakIvarLayout;
    property_list_t *baseProperties;
		...
};
```


#### Ivar结构

​	再回到成员变量这里，先看一下ivar的结构，offset表示的是成员变量在实例中的偏移量，通过计算偏移量就能找到类实例中成员变量的位置，继而读取变量的内容。name、type、size都表示着变量的基本信息。

```c++
struct ivar_t {
    int32_t *offset;
    const char *name;
    const char *type;
    // alignment is sometimes -1; use alignment() instead
    uint32_t alignment_raw;
    uint32_t size;

    uint32_t alignment() const {
        if (alignment_raw == ~(uint32_t)0) return 1U << WORD_SHIFT;
        return 1 << alignment_raw;
    }
};
```

​	因为class_copyIvarList只返回了当前类声明的成员变量，那父类的成员变量又如何访问？上面说明了如何通过ivar的偏移量来访问类实例的变量。猜测类实例的变量排布大概是先排列父类的变量，然后子类的变量从父类之后排列。由此也可以理解子类的ivars中不会有父类的成员变量。父类的成员变量的偏移量是确定的，子类的成员变量继父类之后排列，也就不会出现内存重叠的问题，接下来在实例中打印对象的内存。

#### 类实例的内存分布

&emsp;编译器会自动为我们的类进行内存对齐的优化。减少实例的内存占用空间。

> *内存对齐原则其实可以简单理解为**min(m,n)**——m为当前开始的位置，n为所占位数。当m是n的整数倍时，条件满足；否则m位空余，m+1，继续min算法。*

```objc
// Animal类
@interface Animal : NSObject
@property (assign, nonatomic) NSInteger identify;
@property (assign, nonatomic) uint8 code;
@end
```

&emsp;Animal类实例化，`identify=0x1122`，`code=17`，然后打印内存：

```objc
(lldb) x/4g 0x10313db50
0x10313db50: 0x001d800100002649 0x0000000000000011
0x10313db60: 0x0000000000001122 0x0000000000000000
```

`0x001d800100002649`是isa指针，
`0x0000000000000011`中11是code的值，剩余的补0的内容是为了内存对齐。
`0x0000000000001122`就是identify值。

```objc
//Dog类
@interface Dog : NSObject
@property (assign, nonatomic) NSInteger age;
@property (assign, nonatomic) char tag;
- (void)foo;
@end
```

&emsp;Dog类实例化，赋值同样的identify、code，`age=16`，`tag='a'`后打印内存：

```objc
(lldb) x/6g 0x10313dc70
0x10313dc70: 0x001d800100002699 0x0000000000000011
0x10313dc80: 0x0000000000001122 0x0000000000000061
0x10313dc90: 0x0000000000000010 0x0000000000000000
```

0x001d800100002699是类实例的isa，指向类对象

```objc
(lldb) po (Class)0x001d800100002699
Dog
```

`0x0000000000000011 0x0000000000001122 `，值就是父类ivars占的内存。
`0x0000000000000061`，因为char占一个字节，为了字节对齐，所以剩余的7个字节0补齐。
`0x0000000000000010`, 就是age的值——16。NSInteger占8个字节。

#### .m中直接访问成员变量，编译器是如何处理的

​	在代码实现中，类中声明的成员变量，我们可以直接通过下划线来访问变量。那系统是如何实现变量访问的呢？（以下为clang重写后的代码）

```c++
/*
	在.m文件中重写了name的setter，简单的_name=name；
*/
static void _I_Human_setName_(Human * self, SEL _cmd, NSString * _Nonnull name) {
    (*(NSString * _Nonnull *)((char *)self + OBJC_IVAR_$_Human$_name)) = name;
}
```

​	从代码中可以看到，预处理后的成员变量的访问直接通过偏移量计算来实现。