# Mach-O混淆原理

## 混淆流程

#### 1. 获取Mach-O

解压ipa包，获取到可执行文件即Mach-O文件。

#### 2. 修改类名

修改Mach-O文件中的__objc_classname section中的类名字符串为等长度的替换字符串; 为了保证使用NSClassFromString函数可以正确的获取到原始类，这里记录修改以前的字符映射关系后续使用。

#### 3. 重签名

使用codesign，将ipa重签名



## 原理解析

这里拿最简单的一个例子做介绍，源码如下：

```objc
#import <Foundation/Foundation.h>
#import "Animal.h"
#import "Man.h"

void foo(void);

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        [[Man alloc] init];
        
        [[Animal alloc] init];
        
        foo();
    }
    return 0;
}

void foo() {
    Class cls = NSClassFromString(@"Man");
}
```

代码中声明了两个类Animal和Man。然后main文件中调用了两个类个构造方法。很简单是吧~

接下来，将项目打包，就可以获取到可执行文件了。

### 类名的作用

想想程序运行中什么时候会用到类名？

程序在装载的时候构建了一个NXMapTable——*gdb_objc_realized_classes*，用于存放类，key是类名。runtime中objc_getClass函数就是从这个表中获取到对应名称的类。

那么代码` [[Man alloc] init];`到底执行了什么？

## Disassembly

使用hopper查看反编译后的汇编代码：

```assembly
0000000100003eaf         push       rbp
0000000100003eb0         mov        rbp, rsp
0000000100003eb3         push       r14
0000000100003eb5         push       rbx
0000000100003eb6         call       imp___stubs__objc_autoreleasePoolPush
0000000100003ebb         mov        r14, rax
0000000100003ebe         mov        rdi, qword [objc_cls_ref_Man]
0000000100003ec5         call       imp___stubs__objc_alloc_init
0000000100003eca         mov        rbx, qword [_objc_release_100004000]
0000000100003ed1         mov        rdi, rax                                    ; argument "instance" for method _objc_release
0000000100003ed4         call       rbx                                         ; _objc_release
0000000100003ed6         mov        rdi, qword [objc_cls_ref_Animal]
0000000100003edd         call       imp___stubs__objc_alloc_init
0000000100003ee2         mov        rdi, rax                                    ; argument "instance" for method _objc_release
0000000100003ee5         call       rbx                                         ; _objc_release
0000000100003ee7         call       sub_100003efb
0000000100003eec         mov        rdi, r14                                    ; argument "pool" for method imp___stubs__objc_autoreleasePoolPop
0000000100003eef         call       imp___stubs__objc_autoreleasePoolPop
0000000100003ef4         xor        eax, eax
0000000100003ef6         pop        rbx
0000000100003ef7         pop        r14
0000000100003ef9         pop        rbp
0000000100003efa         ret
                        ; endp
```

可以看到在使用`alloc] init]`创建一个Man对象的时候，`__objc_alloc_init`函数的入参是`objc_cls_ref_Man`——类的引用。同命令行使用MachOView看到的内容如下：

```assembly
100003ebe    movq 0x42f3(%rip), %rdi
100003ec5    callq "[0x100003f26->__objc_alloc_init]"
```

根据相对寻址得出rdi寄存器的内容为0x10003ec5 + 0x42f3 = 0x100082b8。该地址的内容位于`__objc_classrefs` section, 如下：

```assembly
1000081b8 0x1000081f8
```

指向的是`__objc_data` section，如下：

```assembly
00000001000081d0         struct __objc_class {                                  ; DATA XREF=__objc_class_Man_class
                             _OBJC_METACLASS_$_NSObject,          // metaclass
                             _OBJC_METACLASS_$_NSObject,          // superclass
                             __objc_empty_cache,                  // cache
                             0x0,                                 // vtable
                             __objc_metaclass_Man_data            // data
                         }
                             ; 
                             ; @class Man : NSObject {
                             ; }
                     __objc_class_Man_class:
00000001000081f8         struct __objc_class {                                  ; DATA XREF=0x100004030, objc_cls_ref_Man
                             __objc_metaclass_Man_metaclass,      // metaclass
                             _OBJC_CLASS_$_NSObject,              // superclass
                             __objc_empty_cache,                  // cache
                             0x0,                                 // vtable
                             __objc_class_Man_data                // data
                         }
                             ; 
                             ; @metaclass Animal  {
                             ; }
```

到这里已经是Man类本身了。由此可见，类的创建跟classname无关（旧版本的sdk编译的时候alloc，init也是通过objc_msgsend调用的）。

那么类名是什么时候被使用呢，然后在看下`__objc_class_Man_data`：

```assembly
 __objc_class_Man_data:
0000000100008068         struct __objc_data {                                   ; "Man", DATA XREF=__objc_class_Man_class
                             0x90,                                // flags
                             8,                                   // instance start
                             8,                                   // instance size
                             0x0,
                             0x0,                                 // ivar layout
                             0x100003f70,                         // name
                             0x0,                                 // base methods
                             0x0,                                 // base protocols
                             0x0,                                 // ivars
                             0x0,                                 // weak ivar layout
                             0x0                                  // base properties
                         }
```

再看看name地址0x100003f70，该地址位于`__objc_classname` section。

```assembly
100003f70 4d 61 6e 00 ... # Man.
```

yes, name的地址的内容即为"Man"，寻找的类名。

## NSClassFromString

让我们再来看看NSClassFromString方式拿到的类的汇编：

Hopper:

```assembly
        ; ================ B E G I N N I N G   O F   P R O C E D U R E ================


                     sub_100003efb:
0000000100003efb         push       rbp                                         ; CODE XREF=EntryPoint+56
0000000100003efc         mov        rbp, rsp
0000000100003eff         lea        rdi, qword [cfstring_Man]                   ; @"Man", argument "aClassName" for method imp___stubs__NSClassFromString
0000000100003f06         pop        rbp
0000000100003f07         jmp        imp___stubs__NSClassFromString
                        ; endp
```
再看看cfstring_Man
```assembly
        ; Section __cfstring
        ; Range: [0x100004010; 0x100004030[ (32 bytes)
        ; File offset : [16400; 16432[ (32 bytes)
        ;   S_REGULAR

                     cfstring_Man:
0000000100004010         dq         ___CFConstantStringClassReference, 0x7c8, 0x100003f7b, 0x3 ; "Man", DATA XREF=sub_100003efb+4

  			; Section __cstring
        ; Range: [0x100003f7b; 0x100003f7f[ (4 bytes)
        ; File offset : [16251; 16255[ (4 bytes)
        ; Flags: 0x2
        ;   S_CSTRING_LITERALS

0000000100003f7b         db         "Man", 0                                    ; DATA XREF=cfstring_Man
```

以上就不详细做寻址的步骤了，直接通过hopper可以看到，最终指向的`__cstring` section中的Man字符串。

由此可以看出”Man“指向不是`__objc_classname`，而是`__cstring`。因此修改了`__objc_classname`后会影响原代码中NSClassFromString获取类。

那么就会想到，把__cstring sectin中的类名也替换掉不就可以了吗？

答案是不可以，`__cstring` section包含的是文件中所有的字符串，为了避免**非NSClassFromString**函数调用同样字符串异常，不能直接修改这个section。平替方案是使用宏替换原有代码中NSClassFromString的调用，用原有类名映射出混淆后的类名，然后调用NSClassFromString函数。这样场景的还有一些runtime的函数。

## 结论

Mach-O混淆是通过修改`__objc_classname`段来修改类名。在程序运行中类名只是个符号，用来映射类。