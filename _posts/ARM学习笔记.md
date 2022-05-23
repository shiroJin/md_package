---
title: ARM学习笔记
---

# ARM学习笔记

在xcode中，选中文件后`Product->Perform Action->Assemble xxx.m`就能看到汇编代码，可以更好的了解代码的执行过程。这里简单介绍下常用的一些命令（instruction）。



## 寄存器

| x0-x30  | 64bit | 通用寄存器，如果有需要可以当做32bit使用：WO-W30              |
| ------- | ----- | ------------------------------------------------------------ |
| FP(x29) | 64bit | 保存栈帧地址(栈底指针)。栈基址。函数调用的时候的起始栈地址   |
| LR(x30) | 64bit | 通常称X30为程序链接寄存器，保存子程序结束后需要执行的下一条指令（上一次函数调用的下一条指令地址），即返回地址。 |
| SP      | 64bit | 保存栈指针,使用 SP/WSP来进行对SP寄存器的访问。               |
| PC      | 64bit | 程序计数器，俗称PC指针，总是指向即将要执行的下一条指令,在arm64中，软件是不能改写PC寄存器的。 |
| CPSR    | 64bit | 状态寄存器                                                   |

**状态寄存器条件码**

| 操作码 | 条件码助记符                  | 标志     | 含义               |
| ------ | ----------------------------- | -------- | ------------------ |
| 0000   | EQ                            | Z=1      | 相等               |
| 0001   | NE(Not Equal)                 | Z=0      | 不相等             |
| 0010   | CS/HS(Carry Set/High or Same) | C=1      | 无符号数大于或等于 |
| 0011   | CC/LO(Carry Clear/LOwer)      | C=0      | 无符号数小于       |
| 0100   | MI(MInus)                     | N=1      | 负数               |
| 0101   | PL(PLus)                      | N=0      | 正数或零           |
| 0110   | VS(oVerflow set)              | V=1      | 溢出               |
| 0111   | VC(oVerflow clear)            | V=0      | 没有溢出           |
| 1000   | HI(High)                      | C=1,Z=0  | 无符号数大于       |
| 1001   | LS(Lower or Same)             | C=0,Z=1  | 无符号数小于或等于 |
| 1010   | GE(Greater or Equal)          | N=V      | 有符号数大于或等于 |
| 1011   | LT(Less Than)                 | N!=V     | 有符号数小于       |
| 1100   | GT(Greater Than)              | Z=0,N=V  | 有符号数大于       |
| 1101   | LE(Less or Equal)             | Z=1,N!=V | 有符号数小于或等于 |
| 1110   | AL                            | 任何     | 无条件执行(默认)   |
| 1111   | NV                            | 任何     | 从不执行           |

## 指令与伪指令

指令有对应的机器码，CPU可以直接识别并执行。而伪指令，没有对应的机器码，它需要经过编译器翻译成指令才能被CPU识别和执行。

```assembly
ldr r1, =0xfff //伪指令
.text
.global _start
_start:
@这是一条伪指令，没有对应的机器码，被编译器翻译为LDR       R0,[PC,#0x0008]
LDR R0, =var ; int * R0 = var
@这是一条指令
LDR R1, [R0] ; int R1 = *R0
@指令有对应的机器码，编译器原样执行
MOV R1, R0
NOP
NOP
.data
var: int * var; *var = 0x8;
.word 0x8
.end
```



## 常见指令

### mov

将某一寄存器的值复制到另一寄存器（只能用于寄存器与寄存器或者寄存器与常量之间传值，不能用于内存地址），如：

```assembly
mov x1, x0    ; 将寄存器x0的值赋值给x1
```

### str&stur

存储指令，(store register) 将寄存器中的值写入到内存中。str和stur都是存储指令，区别在寻址的偏移量正负之分。如：

```assembly
str x0, [sp, #0x8]    ; 将寄存器 x0 中的值保存到栈内存 [sp + 0x8] 处 
stur x0, [x29, #-0x8]    ; 将寄存器x0中的值保存到栈内存 [x29 - 0x8] 处
```

### ldr&ldur

读取指令（load register）；将内存地址的值读取到寄存器中。ldr和ldur的区别与上述存储指令相同，ldr用于正偏移的地址运算，ldur用于负地址。如：

```assembly
ldr x0, [sp, #0x8]    ; 将栈地址sp+0x8的值读取到x0寄存器中
ldur x0, [x29, #-0x8]    ; 将栈地址x29-0x8的值读取到x0寄存器中
```

### stp

入栈指令（`str` 的变种指令，可以同时操作两个寄存器），如：

```assembly
stp x29, x30, [sp, #0x10] 	; 将 x29, x30 的值存入 sp 偏移 16 个字节的位置 
```

### ldp

出栈指令（`ldr` 的变种指令，可以同时操作两个寄存器），如：

```assembly
ldp x29, x30, [sp, #0x10] 	; 将 sp 偏移 16 个字节的值取出来，存入寄存器 x29 和寄存器 x30 
```

### adrp

用来定位数据段中的数据用, 因为 *aslr* 会导致代码及数据的地址随机化, 用 *adrp* 来根据 *pc* 做辅助定位

```assembly
adrp	x8, _OBJC_SELECTOR_REFERENCES_@PAGE    ;将'_OBJC_SELECTOR_REFERENCES_'所在section的起始地址保存到x8寄存器中
ldr	x1, [x8, _OBJC_SELECTOR_REFERENCES_@PAGEOFF]    ; 加上偏移量做一个寻址操作，并将地址保存到x1寄存器中。
```

### sub

减法指令；将两寄存器的值相减 并将结果保存到指定寄存器中，如：

```assembly
sub x0, x0, #0x8    ; x0 = x0 - 0x8
```

### add

加法指定；如：

```assembly
add x0, x0, #0x1    ; x0 = x0 + 0x1
```

### sdiv

除法运算指令；如：

```assembly
sdiv x0, x1, x2       ; x0 = x1 / x2
```

### mul

乘法运算指令；如：

```assembly
mul x0, x1, x2        ; x0 = x1 * x2
```

### and

按位与。如：

```assembly
and x0, x0, #0xf      ; x0 = x0 & 0xf
```

### orr

按位或。如：

```assembly
orr x0, x0, #0xf      ; x0 = x0 | 0xf
```

### eor

按位异或。如：

```assembly
eor x0, x0, #0xf      ; x0 = x0 ^ 0xf
```

### LSL

逻辑左移

### LSR

逻辑右移

### ASR

算术右移

### ROR

循环右移

```assembly
cbz	w8, LBB11_2    ; 如果w8寄存器的值为0，则跳转到‘LBB11_2’处
```

### cbnz

和非 0 比较（Compare），如果结果非零（Non Zero）就转移（只能跳到后面的指令）;

```assembly
	cbnz	w8, LBB11_2    ; 如果w8寄存器的值不等于0则跳转到‘LBB11_2’label处。
```

### subs

比较指令；

```assembly
subs x0, x1, x2    ; x0 = x1 - x2, 并设置 CPSR 寄存器的 C 标志位
```

### cmp

比较指令，相当于 `subs`，影响程序状态寄存器`CPSR `;


### cset

比较指令，满足条件，则并置 `1`，否则置 `0` ，如：

```
cmp w8, #2        ; 将寄存器 w8 的值和常量 2 进行比较
cset w8, gt       ; 如果是大于(grater than)，则将寄存器 w8 的值设置为 1，否则设置为 0
```

### b 

（branch）跳转到某地址（无返回）, 不会改变 *lr (x30)* 寄存器的值，只改变pc寄存的值；一般是本方法内的跳转，如 `while` 循环，`if else` 等 ，如：

```assembly
b LBB0_1      ; 直接跳转到标签 ‘LLB0_1’ 处开始执行
```

#### b.le（条件码）

```assembly
b.le	LBB11_2     ; CPSR寄存器 C=0，则跳转‘LBB11_2’
```

### br

功能同上b指令。不同的是跳转的地址由寄存器传递。

```assembly
br x8    ; x8寄存器的值作为跳转地址
```

### bl

call指令。跳转到某地址（有返回），先将下一指令地址（即函数返回地址）保存到寄存器 *lr* (x30)中，再进行跳转 ；一般用于不同方法直接的调用 ，如：

```assembly
bl 0x100cfa754	; 先将下一指令地址（‘0x100cfa754’ 函数调用后的返回地址）保存到寄存器 ‘lr’ 中，然后再调用 ‘0x100cfa754’ 函数
```

### blr

跳转到 `某寄存器` (的值)指向的地址（有返回），先将下一指令地址（即函数返回地址）保存到寄存器  *lr* (x30)中，再进行跳转；与bl不同的是，bl指令的内容具体的地址，blr则是读取寄存器内的值。如：

```assembly
blr x20       ; 先将下一指令地址（‘x20’指向的函数调用后的返回地址）保存到寄存器 ‘lr’ 中，然后再调用 ‘x20’ 指向的函数
```

### fcvtzs

(Float Convert To Zero Signed)*浮点数* 转化为 *定点数* （舍入为0），如：

```assembly
fcvtzs w0, s0	    ; 将向量寄存器 s0 的值(浮点数，转换成 定点数)保存到寄存器 w0 中
复制代码
```

### cbz

和 0 比较（Compare），如果结果为零（Zero）就转移（只能跳到后面的指令）;


### ret

子程序（函数调用）返回指令，返回地址已默认保存在寄存器 *lr* (x30) 中



## Label

在汇编代码中常常会看到`LBB0_2:` ,这样的字样。`LBB0_2`称之为`local label`，在跳转中常常会用到

> `LBB0_2`are local labels, which are normally used as branch destinations within a function.

```assembly
b.ge	LBB11_2    ; CPSR的标志是ge时，pc寄存器置为LBB11_2处的地址
```



## 注释

ARM中的注释，如：

1. "@"符号作为注释可以放在语句的开始处
2. ";"作为流程只能放在语句的末尾

```assembly
sub	sp, sp, #32                     ; =32
```



## 伪指令

### 数据定义伪操作

数据定义伪操作一般用于为特定的数据分配内存单元，同时对该内存单元中的数据进行初始化，觉的数据定义伪操作有:

#### .byte

在存储器中分配 一个字节的内存单元，用指定的数据对该存储单元进行初始化。

```assembly
.byte 0x1
```

在当前地址分配一个字节的存储单元，将将其初始化为1，类似于C语言中的char _label = 1.

#### .short

在存储器中分配2个字节的内存单元，并用指定的数据对该存储单元进行初始化，用于与.byte类似

#### .word

在存储器中分配4个字节的内存单元，并用指定的数据对该存储单元进行初始化，用于与.byte类似

#### .long

与.word的功能相同

#### .quad

.quad的功能是在内存中分配8个字节的存储单元，并用指定的数据对该存储单元进行初始化。

#### .float

在存储器中分配4个字节的存储空间，并用指定的浮点数据对该空间进行初始化。

#### .space

.space伪操作用于分配一片连续的内存区域，并将其初始化为指定的值，如果后面的填充值省略不写，则默认在后面填充0

#### .skip

等同与.space

#### .string, ascii, .asciz

这3条伪操作的功能都是定义一个字符串：

```assembly
_label:
.string "Hello, World!"
```

#### .rept

.rept伪操作功能是 重复执行后面的指令，以.rept开始，并以.endr结束，用法:

```assembly
.rept 3
add r1, r1, #1
.endr
```

### 杂项操作伪指令

GNU汇编中还有一些其他的伪操作，在汇编程序中经常会使用到它们，包括而在这些：

#### .align

.align伪操作可通过添加填充字节的试，使当前位置满足指定的对齐方式。举例：

```assembly
.align 2
.string "abcde"
```

声明后面的字符串的对齐方式是4(2的2次方)字节对齐，这个字符串会占用8个字节的存储空间。

#### .section

.section伪操作用于定义一个段，一个GNU的源程序至少需要一个段，大的程序可以包含多个代码段和数据段。

可能用来定义自定义段

```assembly
.section	__DATA,__const    ; 表示数据所在的segment和section
```

#### .data

.data伪操作用来定义一个数据段

#### .text

.text伪操作用来定义一个数据段

#### .include

.include 伪操作用来包含一个头文件。

#### .extern

.extern用于声明一个外部符号，即告诉编译器当前符号不是在本源文件中定义的，而是在其它源文件中定义的，当前文件需要引用这个符号。

#### .weak

.weak用来声明一个符号是弱符号，即如果这个符号没有定义，编译器就会忽略，不会报错。

#### .end

.end代表程序的结束位置

#### .globl

`.globl	"___block_descriptor_40_e8_32w_e5_v8?0l"`表示一个全局的符号



## 其他

#### 栈对齐

虽然该函数没有临时变量，但是调用 printf 函数后，编译器自动会加上 该函数*返回值* 的处理，由于 **arm64 规定了整数型返回值放在** **x0** **寄存器里**，因此会隐藏有一个局部变量 int return_value; 的声明在，该临时变量占用 4字节空间；又因为 **arm64 下对于使用** **sp** **作为地址基址寻址的时候，必须要** **16byte-alignment**（对齐），所以申请了 16字节空间作为临时变量使用。具体参见 [这里](https://link.juejin.cn/?target=https%3A%2F%2Fcommunity.arm.com%2Fdeveloper%2Fip-products%2Fprocessors%2Fb%2Fprocessors-ip-blog%2Fposts%2Fusing-the-stack-in-aarch64-implementing-push-and-pop)。



## 参考

[arm64 架构之入栈/出栈操作](https://juejin.cn/post/6844903816362459144)

[汇编quad_ARM汇编](https://blog.csdn.net/weixin_32004383/article/details/114020564)