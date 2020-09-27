---
title: 从源码看对nil发消息是安全的
---

​	在iOS开发中，我们会发现给nil发送消息时安全的，但是给null消息就会崩溃。这里探讨一下nil和null两者的区别。

​	打印nil和[NSNull null]的地址

> ​	nil: 0x0
>
> ​	null: 0x10c054ff0

​	当向两者发送消息时候，会调用msg_send函数进行方法的调用，msg_send函数的第一个参数调用者本身，这里为nil和null，第二个参数是SEL——方法的名称，而后跟着的是方法的参数。

> msg_send(nil, selector)
>
> msg_sen(null, selector)

​	接下来看msg_send源码的源码，因为参数及性能的问题，msg_send是用汇编写的，以下代码为msg_send起始和末尾片段。

```assembly
	ENTRY _objc_msgSend
	UNWIND _objc_msgSend, NoFrame

	cmp	p0, #0			// nil check and tagged pointer check
#if SUPPORT_TAGGED_POINTERS
	b.le	LNilOrTagged		//  (MSB tagged pointer looks negative)
#else
	b.eq	LReturnZero
#endif
	...
	...
LReturnZero:
	// x0 is already zero
	mov	x1, #0
	movi	d0, #0
	movi	d1, #0
	movi	d2, #0
	movi	d3, #0
	ret
```

​	汇编代码中第一个指令就是`cmp p0, #0`，比较第一个参数和数字0。从前面的打印结果可以看到nil的值为0x0，与0相等，然后调转到LReturnZero。LReturnZero中将寄存器的值还原为0，然后return回去。因此当给nil发消息时，不会崩溃，msg_send函数会返回0。再来看看null，因为null指针的值不是0，因此会继续走寻找方法的函数，然后null本身没有方法，因此最后抛出错误崩溃。

