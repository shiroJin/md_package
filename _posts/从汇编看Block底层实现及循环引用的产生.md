---
title: 从汇编看Block底层
---

# block是什么

源码：

```objective-c
- (void)foo2 {
    self.myBlock = ^{
        [self foo];
    };
}
```

Assemble后：

```assembly
"-[Foo foo2]":                          ; @"\01-[Foo foo2]"
Lfunc_begin4:
; %bb.0:
	sub	sp, sp, #96                     ; =96
	stp	x29, x30, [sp, #80]             ; 16-byte Folded Spill
	add	x29, sp, #80                    ; =80
	stur	x0, [x29, #-8]
	stur	x1, [x29, #-16]
	add	x8, sp, #24                     ; =24
	str	x8, [sp, #8]                    ; 8-byte Folded Spill
Ltmp12:
	adrp	x9, __NSConcreteStackBlock@GOTPAGE
	ldr	x9, [x9, __NSConcreteStackBlock@GOTPAGEOFF]
	str	x9, [sp, #24]
	mov	w9, #-1040187392
	str	w9, [sp, #32]
	str	wzr, [sp, #36]
	adrp	x9, "___11-[Foo foo2]_block_invoke"@PAGE
	add	x9, x9, "___11-[Foo foo2]_block_invoke"@PAGEOFF
	str	x9, [sp, #40]
	adrp	x9, "___block_descriptor_40_e8_32s_e5_v8?0l"@PAGE
	add	x9, x9, "___block_descriptor_40_e8_32s_e5_v8?0l"@PAGEOFF
	str	x9, [sp, #48]
	add	x8, x8, #32                     ; =32
	str	x8, [sp, #16]                   ; 8-byte Folded Spill
	ldur	x0, [x29, #-8]
	bl	_objc_retain
	ldr	x2, [sp, #8]                    ; 8-byte Folded Reload
	str	x0, [sp, #56]
	ldur	x0, [x29, #-8]
	adrp	x8, _OBJC_SELECTOR_REFERENCES_.2@PAGE
	ldr	x1, [x8, _OBJC_SELECTOR_REFERENCES_.2@PAGEOFF]
	bl	_objc_msgSend
	ldr	x0, [sp, #16]                   ; 8-byte Folded Reload
	mov	x1, #0
	bl	_objc_storeStrong
	ldp	x29, x30, [sp, #80]             ; 16-byte Folded Reload
	add	sp, sp, #96                     ; =96
	ret
Ltmp13:
Lfunc_end4:
                                        ; -- End function
	.p2align	2                               ; -- Begin function __11-[Foo foo2]_block_invoke
"___11-[Foo foo2]_block_invoke":        ; @"__11-[Foo foo2]_block_invoke"
Lfunc_begin5:
; %bb.0:
	sub	sp, sp, #32                     ; =32
	stp	x29, x30, [sp, #16]             ; 16-byte Folded Spill
	add	x29, sp, #16                    ; =16
	str	x0, [sp, #8]
	str	x0, [sp]
Ltmp14:
	ldr	x0, [x0, #32]
	adrp	x8, _OBJC_SELECTOR_REFERENCES_@PAGE
	ldr	x1, [x8, _OBJC_SELECTOR_REFERENCES_@PAGEOFF]
	bl	_objc_msgSend
Ltmp15:
	ldp	x29, x30, [sp, #16]             ; 16-byte Folded Reload
	add	sp, sp, #32                     ; =32
	ret
Ltmp16:
Lfunc_end5:
                                        ; -- End function
```

这里假设起始sp=0x60，方便后续计算。依次解释指令的执行过程和寄存器，栈的值。

```assembly
	sub	sp, sp, #96                     ; =96
	stp	x29, x30, [sp, #80]             ; 16-byte Folded Spill
	add	x29, sp, #80                    ; =80
	stur	x0, [x29, #-8]
	stur	x1, [x29, #-16]
```

1. sub指令；sp = sp - 0x60 => sp = 0x00
2. store pair指令； x29, x30寄存器的值存到sp+0x50=0x50的地址，x29，x30表示fp，lr寄存器。
3. add指令；fp = sp + 0x50 = 0x50
4. store指令；x0寄存器的值存到[x29, #-8]=0x50-0x8=0x48的地址。x0=self
5. 同上，x1存到0x40的地址。x1=@selector(foo2)

此时寄存器和栈的状态如下

| 寄存器 | 值              |
| ------ | --------------- |
| x0     | self            |
| x1     | @selector(foo2) |
| fp     | 0x50            |
| sp     | 0x00            |

| 栈地址 | 值              |
| ------ | --------------- |
| 0x60   | 上个函数的lr    |
| 0x58   | 上个函数的fp    |
| 0x50   | self            |
| 0x48   | @selector(foo2) |

```assembly
	add	x8, sp, #24                     ; =24
	str	x8, [sp, #8]                    ; 8-byte Folded Spill
Ltmp12:
	adrp	x9, __NSConcreteStackBlock@GOTPAGE
	ldr	x9, [x9, __NSConcreteStackBlock@GOTPAGEOFF]
	str	x9, [sp, #24]
	mov	w9, #-1040187392
	str	w9, [sp, #32]
	str	wzr, [sp, #36]
	adrp	x9, "___11-[Foo foo2]_block_invoke"@PAGE
	add	x9, x9, "___11-[Foo foo2]_block_invoke"@PAGEOFF
	str	x9, [sp, #40]
	adrp	x9, "___block_descriptor_40_e8_32s_e5_v8?0l"@PAGE
	add	x9, x9, "___block_descriptor_40_e8_32s_e5_v8?0l"@PAGEOFF
	str	x9, [sp, #48]
```

1. x8 = sp + 24 = 0x18
2. x8寄存器的值存到0x08的地址（[sp, #8] = 0x00 + 0x8）
3. adrp是针对aslr技术，获取偏移后的地址；将__NSConcreteStackBlock的isa读取到x9寄存器
4. 将x9寄存器的值保存到地址0x18
5. w9（4字节）赋值#-1040187392
6. w9寄存器的值存到地址0x20
7. wzr（word zero register），地址0x24写入4字节的0。
8. 同上，x9寄存器赋值"___11-[Foo foo2]_block_invoke"的地址
9. x9寄存器的值存到地址0x28
10. 同上将"___block_descriptor_40_e8_32s_e5_v8?0l"的地址存到地址0x30

| 寄存器 | 值              |
| ------ | --------------- |
| x0     | self            |
| x1     | @selector(foo2) |
| fp     | 0x50            |
| sp     | 0x00            |
| x8     | 0x18            |

| 栈地址 | 值                                       |
| ------ | ---------------------------------------- |
| 0x60   | 上个函数的lr                             |
| 0x58   | 上个函数的fp                             |
| 0x50   | self                                     |
| 0x48   | @selector(foo2)                          |
| ...    |                                          |
| 0x30   | "___block_descriptor_40_e8_32s_e5_v8?0l" |
| 0x28   | "___11-[Foo foo2]_block_invoke"          |
| 0x20   | -1040187392; 0                           |
| 0x18   | __NSConcreteStackBlock                   |
| 0x10   |                                          |
| 0x08   | 0x18                                     |
| 0x00   |                                          |

```assembly
	add	x8, x8, #32                     ; =32
	str	x8, [sp, #16]                   ; 8-byte Folded Spill
	ldur	x0, [x29, #-8]
	bl	_objc_retain
	ldr	x2, [sp, #8]                    ; 8-byte Folded Reload
	str	x0, [sp, #56]
	ldur	x0, [x29, #-8]
	adrp	x8, _OBJC_SELECTOR_REFERENCES_.2@PAGE
	ldr	x1, [x8, _OBJC_SELECTOR_REFERENCES_.2@PAGEOFF]
	bl	_objc_msgSend
	ldr	x0, [sp, #16]                   ; 8-byte Folded Reload
	mov	x1, #0
	bl	_objc_storeStrong
```

1. x8 = x8 + 0x20 = 0x38
2. x8寄存器值存到地址0x10
3. x0 = self
4. 调用_objc_retain函数
5. x2 = 0x18
6. x0寄存器值存到地址0x38
7. _OBJC_SELECTOR_REFERENCES_.2指的是@selector(setMyBlock:)。x1 = @selector(setMyBlock:)
8. 调用_objc_msgSend函数
9. 地址0x10的值读取到x0寄存器
10. x1 = 0
11. 调用_objc_storeStrong

| 寄存器 | 值              |
| ------ | --------------- |
| x0     | self            |
| x1     | @selector(foo2) |
| fp     | 0x50            |
| sp     | 0x00            |
| x8     | 0x18            |

| 栈地址 | 值                                       |
| ------ | ---------------------------------------- |
| 0x60   | 上个函数的lr                             |
| 0x58   | 上个函数的fp                             |
| 0x50   | self                                     |
| 0x48   | @selector(foo2)                          |
| 0x38   | self                                     |
| 0x30   | "___block_descriptor_40_e8_32s_e5_v8?0l" |
| 0x28   | "___11-[Foo foo2]_block_invoke"          |
| 0x20   | -1040187392; 0                           |
| 0x18   | __NSConcreteStackBlock                   |
| 0x10   | 0x38                                     |
| 0x08   | 0x18                                     |
| 0x00   |                                          |

```assembly
	ldp	x29, x30, [sp, #80]             ; 16-byte Folded Reload
	add	sp, sp, #96                     ; =96
	ret
```

1. 恢复上个函数的fp，lr寄存器值
2. sp -= 0x60
3. return

上述就是foo2函数的指令执行过程， 可以发现block初始化的时候是分配在栈空间，block也是个对象，有isa指正，这里是__NSConcreteStackBlock类。

通过`clang -rewrite-objc Foo.m`可以看到block结构体的声明，如下：

```c++
#ifndef BLOCK_IMPL
#define BLOCK_IMPL
struct __block_impl {
  void *isa;
  int Flags;
  int Reserved;
  void *FuncPtr;
};

struct __Foo__foo2_block_impl_0 {
  struct __block_impl impl;
  struct __Foo__foo2_block_desc_0* Desc;
  Foo *const __strong self;
  __Foo__foo2_block_impl_0(void *fp, struct __Foo__foo2_block_desc_0 *desc, Foo *const __strong _self, int flags=0) : self(_self) {
    impl.isa = &_NSConcreteStackBlock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
  }
};
```

| 栈地址 | 值                                       |
| ------ | ---------------------------------------- |
| 0x38   | self                                     |
| 0x30   | "___block_descriptor_40_e8_32s_e5_v8?0l" |
| 0x28   | "___11-[Foo foo2]_block_invoke"          |
| 0x20   | -1040187392; 0                           |
| 0x18   | __NSConcreteStackBlock                   |

跟栈去的内存分配完全吻合。从低地址到高地址依次为isa，Flags，Reserved，FuncPtr，Desc，self。其中self就是block捕获的变量。

接着看下block_descriptor到底是什么。

```assembly
	.private_extern	"___block_descriptor_40_e8_32s_e5_v8?0l" ; @"__block_descriptor_40_e8_32s_e5_v8\01?0l"
	.section	__DATA,__const
	.globl	"___block_descriptor_40_e8_32s_e5_v8?0l"
	.weak_def_can_be_hidden	"___block_descriptor_40_e8_32s_e5_v8?0l"
	.p2align	3
"___block_descriptor_40_e8_32s_e5_v8?0l":
	.quad	0                               ; 0x0
	.quad	40                              ; 0x28
	.quad	___copy_helper_block_e8_32s
	.quad	___destroy_helper_block_e8_32s
	.quad	l_.str
	.quad	256                             ; 0x100
```

可以看到`___block_descriptor_40_e8_32s_e5_v8`在data段。

cpp的结构如下

```c++
static struct __Foo__foo_block_desc_0 {
  size_t reserved;
  size_t Block_size;
  void (*copy)(struct __Foo__foo_block_impl_0*, struct __Foo__foo_block_impl_0*);
  void (*dispose)(struct __Foo__foo_block_impl_0*);
} __Foo__foo_block_desc_0_DATA = { 0, sizeof(struct __Foo__foo_block_impl_0), __Foo__foo_block_copy_0, __Foo__foo_block_dispose_0};
```

结合`__Foo__foo_block_desc_0`的结构，可以得到reserved=0，Block_size=40。跟上述栈的内存分配吻合。而后则是copy和destory函数。

# 为什么会产生循环引用

```assembly
	bl	_objc_retain 			; x0 = self
```

上述汇编代码中可以看到在构建block时，self被调用retain。从而导致了循环引用的问题出现。

# __weak是如何解除循环引用的？

接着上述代码加上weak修饰符，如下👇🏻。

```objc
- (void)foo {
    __weak Foo *wself = self;
    self.myBlock = ^{
        [wself foo];
    };
}
```

assemble：

```assembly
"-[Foo foo]":                           ; @"\01-[Foo foo]"
Lfunc_begin0:
; %bb.0:
	sub	sp, sp, #128                    ; =128
	stp	x29, x30, [sp, #112]            ; 16-byte Folded Spill
	add	x29, sp, #112                   ; =112
	stur	x0, [x29, #-8]
	stur	x1, [x29, #-16]
Ltmp3:
	ldur	x1, [x29, #-8]
	sub	x0, x29, #24                    ; =24
	str	x0, [sp, #8]                    ; 8-byte Folded Spill
	bl	_objc_initWeak
	ldr	x1, [sp, #8]                    ; 8-byte Folded Reload
	add	x8, sp, #48                     ; =48
	str	x8, [sp, #24]                   ; 8-byte Folded Spill
	adrp	x9, __NSConcreteStackBlock@GOTPAGE
	ldr	x9, [x9, __NSConcreteStackBlock@GOTPAGEOFF]
	str	x9, [sp, #48]
	mov	w9, #-1040187392
	str	w9, [sp, #56]
	str	wzr, [sp, #60]
	adrp	x9, "___10-[Foo foo]_block_invoke"@PAGE
	add	x9, x9, "___10-[Foo foo]_block_invoke"@PAGEOFF
	str	x9, [sp, #64]
	adrp	x9, "___block_descriptor_40_e8_32w_e5_v8?0l"@PAGE
	add	x9, x9, "___block_descriptor_40_e8_32w_e5_v8?0l"@PAGEOFF
	str	x9, [sp, #72]
	add	x0, x8, #32                     ; =32
	str	x0, [sp, #16]                   ; 8-byte Folded Spill
	bl	_objc_copyWeak
	ldr	x2, [sp, #24]                   ; 8-byte Folded Reload
	ldur	x0, [x29, #-8]
	adrp	x8, _OBJC_SELECTOR_REFERENCES_.2@PAGE
	ldr	x1, [x8, _OBJC_SELECTOR_REFERENCES_.2@PAGEOFF]
Ltmp0:
	bl	_objc_msgSend
Ltmp1:
; %bb.1:
	ldr	x0, [sp, #16]                   ; 8-byte Folded Reload
	bl	_objc_destroyWeak
	sub	x0, x29, #24                    ; =24
	bl	_objc_destroyWeak
	ldp	x29, x30, [sp, #112]            ; 16-byte Folded Reload
	add	sp, sp, #128                    ; =128
	ret
```

从汇编中可以看到，在初始化wself的时候调用了objc_initWeak函数，在构建block结构的时候使用了objc_copyWeak。没有使用objc_retain，因此self的引用计数没有加1。从而没有循环引用的出现。

# block内为什么需要__strong呢？

看👇🏻代码，block中两次引用了wself。

```objc
- (void)foo {
    __weak Foo *wself = self;
    self.myBlock = ^{
        [wself foo1];
        [wself foo1];
    };
}
```

assemble:

```assembly
"___10-[Foo foo]_block_invoke":         ; @"__10-[Foo foo]_block_invoke"
Lfunc_begin1:
; %bb.0:
	sub	sp, sp, #64                     ; =64
	stp	x29, x30, [sp, #48]             ; 16-byte Folded Spill
	add	x29, sp, #48                    ; =48
	str	x0, [sp, #8]                    ; 8-byte Folded Spill
	stur	x0, [x29, #-8]
	stur	x0, [x29, #-16]
Ltmp8:
	add	x0, x0, #32                     ; =32
	bl	_objc_loadWeakRetained
	str	x0, [sp]                        ; 8-byte Folded Spill
	adrp	x8, _OBJC_SELECTOR_REFERENCES_.2@PAGE
	str	x8, [sp, #16]                   ; 8-byte Folded Spill
	ldr	x1, [x8, _OBJC_SELECTOR_REFERENCES_.2@PAGEOFF]
	bl	_objc_msgSend
	ldr	x0, [sp]                        ; 8-byte Folded Reload
	bl	_objc_release
	ldr	x0, [sp, #8]                    ; 8-byte Folded Reload
	add	x0, x0, #32                     ; =32
	bl	_objc_loadWeakRetained
	ldr	x8, [sp, #16]                   ; 8-byte Folded Reload
	str	x0, [sp, #24]                   ; 8-byte Folded Spill
	ldr	x1, [x8, _OBJC_SELECTOR_REFERENCES_.2@PAGEOFF]
	bl	_objc_msgSend
	ldr	x0, [sp, #24]                   ; 8-byte Folded Reload
	bl	_objc_release
Ltmp9:
	ldp	x29, x30, [sp, #48]             ; 16-byte Folded Reload
	add	sp, sp, #64                     ; =64
	ret
```

```assembly
	bl	_objc_loadWeakRetained
	str	x0, [sp]                        ; 8-byte Folded Spill
	adrp	x8, _OBJC_SELECTOR_REFERENCES_.2@PAGE
	str	x8, [sp, #16]                   ; 8-byte Folded Spill
	ldr	x1, [x8, _OBJC_SELECTOR_REFERENCES_.2@PAGEOFF]
	bl	_objc_msgSend
	ldr	x0, [sp]                        ; 8-byte Folded Reload
	bl	_objc_release
```

从这块片段可以看到在引用wself时，先调用了objc_loadWeakRetained，refcnt + 1。然后通过objc_msgSend调用了具体的方法，接着调用了objc_release，refcnt - 1。可以看到苹果底层在引用weak时还是比较严谨的。

可以看到在引用两次wself时，每次引用都会先调用objc_loadWeakRetained，紧接着调用objc_msgSend。在多线程的场景下，两次引用中间self存在被释放的情况，当self被释放后，wself就会被weak系统机制置为nil，从而导致在block的执行过程中self被释放，从而导致一些错误。

接着看一下使用strong修饰后的结果。

```objc
- (void)foo {
    __weak Foo *wself = self;
    self.myBlock = ^{
        __strong Foo *sself = wself;
        [sself foo];
        [sself foo];
    };
}
```

assemble:

```assembly
"___10-[Foo foo]_block_invoke":         ; @"__10-[Foo foo]_block_invoke"
Lfunc_begin1:
; %bb.0:
	sub	sp, sp, #64                     ; =64
	stp	x29, x30, [sp, #48]             ; 16-byte Folded Spill
	add	x29, sp, #48                    ; =48
	stur	x0, [x29, #-8]
	stur	x0, [x29, #-16]
Ltmp5:
	add	x0, x0, #32                     ; =32
	bl	_objc_loadWeakRetained
	add	x8, sp, #24                     ; =24
	str	x8, [sp, #16]                   ; 8-byte Folded Spill
	str	x0, [sp, #24]
	ldr	x0, [sp, #24]
	adrp	x8, _OBJC_SELECTOR_REFERENCES_@PAGE
	str	x8, [sp, #8]                    ; 8-byte Folded Spill
	ldr	x1, [x8, _OBJC_SELECTOR_REFERENCES_@PAGEOFF]
	bl	_objc_msgSend
	ldr	x8, [sp, #8]                    ; 8-byte Folded Reload
	ldr	x0, [sp, #24]
	ldr	x1, [x8, _OBJC_SELECTOR_REFERENCES_@PAGEOFF]
	bl	_objc_msgSend
	ldr	x0, [sp, #16]                   ; 8-byte Folded Reload
	mov	x1, #0
Ltmp6:
	bl	_objc_storeStrong
	ldp	x29, x30, [sp, #48]             ; 16-byte Folded Reload
	add	sp, sp, #64                     ; =64
	ret
```

在加上strong修饰后，因为后续代码中都在引用sself，所以sself在最后才通过objc_storeStrong(&sself, nil)的方式进行一次release。从而也保证了在block调用过程中，self的refcnt一直是+1的状态，在block执行过程中不存在被提前释放的情况。



