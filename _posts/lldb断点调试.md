---
title: LLDB断点调试
---

​	LLDB是个开源的内置于XCode的具有REPL(read-eval-print-loop)特征的Debugger，也是iOS开发中，配合断点使用的调试命令工具。

#### 基本操作命令

​	在lldb中输入help能看到所有的命令，通过help <command>可以看到相应命令的具体用法。以下介绍几种常用的命令。

+ 打印输出 

> p         -- Evaluate an expression on the current thread.  Displays any
>
> ​               returned value with LLDB's default formatting.
>
> po        -- Evaluate an expression on the current thread.  Displays any
>
> ​               returned value with formatting controlled by the type's author.

p和po两个命令是日常打印输出最常用的。po，p是expression的缩写。在命令行中输入help po，可以看到po和p命令的详细信息，

``` 'po' is an abbreviation for 'expression -O  —'，```

``` 'p' is an abbreviation for 'expression —'```

po输出的是对象的description方法的返回值，也可以重写对象的description方法来改变输出。而p输出的是LLDB的默认格式。例如下图NSArray的打印输出。po和p命令同时也能执行代码。根接下来的expr命令功能相同。

+ 动态执行代码

> expression        -- Evaluate an expression on the current thread.  Displays
>                        any returned value with LLDB's default formatting.

expr命令可以执行输入的代码块(call命令的效果相同，都是调用__lldb_expr函数)。通过输入的代码块，我们可以动态的改变变量的值，可以调用一个方法…。使用Option+Enter键可以输入多行代码。

如上图，动态的执行obj=nil这段代码，将obj的值置为nil。需要补充的是，expr命令接收的代码不能直接使用枚举值，例如NSRoundUp需要转化为(NSRoundingMode)1。此外，expr命令不能执行return value。

> display   -- Evaluate an expression at every stop (see 'help target
>                stop-hook'.)

display作用的与expr相同，都是执行一段代码，而display命令在每一次到断点的时候都会执行一遍输入的代码。

+ 查看堆栈信息

> **(lldb)** **help bt**
>
> ​     ---Show the current thread's call stack.  Any numeric argument displays at
>
> ​     most that many frames.  The argument 'all' displays all threads.  Expects
>
> ​     'raw' input (see 'help raw-input'.)
>
> 
>
> Syntax: bt [<digit> | all]
>
> 
>
> 'bt' is an abbreviation for '_regexp-bt'

​	此外，在lldb中只要获取到对象的地址，就能使用expr，call命令调用对象的方法。因为，oc是方法的调用都是msg_send函数。

