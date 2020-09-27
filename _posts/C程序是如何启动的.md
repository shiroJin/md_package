---
title: C语言可执行文件是如何启动的
---
&emsp;&emsp;一个简单程序是如何启动的？简单来说，程序就是一个可执行文件，启动的过程分为以下三个部分：

> 1. 执行fork函数，创建出一个新的进程并清理用户空间。
> 2. 执行execve函数，加载器加载可执行文件。
> 3. 执行地址跳转到ELF文件的入口地址，程序启动。

<!--more-->

## 目录

[TOC]

## 一、可执行文件

&emsp;&emsp;&emsp;可执行文件是一种代码和数据的集合。当程序开始运行时，可执行文件不会立刻加载进物理内存，进程会为可执行在虚拟内存中分配相应数量的页，因为系统不会直接载入内存，系统按需调度页加载。装载在内存的文件也称为映像文件（image）。可执行文件中包含的往往不止代码段，还有数据段和BSS等，<u>每个段映射的大小必须是页的整数倍</u>。假如一个section中只有很少的内容，远不足以填满一页，那这个section还是会被分配一页的内存，空余的部分用0填充，这部分内存也就浪费了。为了避免段太多造成内存浪费，系统根据段的权限分为三种不同的类型：

* 可读可执行的段——代码段
* 可读可写的段——数据段和BSS段
* 只读的段——只读数据段

&emsp;&emsp;系统为上述的段引入了新的概念——Segment，一个Segment中可以包含一个或者多个相同权限的段。如此相同权限的段可以合并到一起，减少了页中为了对齐而产生的空余部分。

&emsp;&emsp;Section和Segment译意上都是“段“，不同的是section是对于ELF文件意义上的段，segment是对于装载时意义的段。装载时为了减少内存的浪费，将几个段合并作为一个段进行加载，这个<u>合并的段</u>页对齐。在装载的时候，物理内存和虚拟内存的映射的单位是页。因此存在一个较小的segment占了一页的小部分，剩余的都填充0。为了优化这一场景，系统采取多次映射的策略，即物理页和虚拟页一对多的映射关系，通过设置偏移量来确定数据内容的位置。在程序头部表的引导下，加载器可以很容易的将可执行文件的页大小的片（chunk）复制到代码段和数据段。

&emsp;&emsp;以下为一个简单的c程序代码，在Linux（x64, little endian）系统下，用gcc编译成一个可执行文件prog。可执行文件由ELF头和连续的Section（段）组成。

```c
#include<stdio.h>
int a = 1;
int b,c;
int main() {
  a++;
  printf("a = %d", a);
  return 0;
}
```

&emsp;&emsp;通过readelf和objdump工具查看可执行文件的头部信息结果如下：

```shell
$ readelf -l prog
Elf file type is DYN (Shared object file)
Entry point 0x540
There are 9 program headers, starting at offset 64

Program Headers:
  Type           Offset             VirtAddr           PhysAddr
                 FileSiz            MemSiz              Flags  Align
  PHDR           0x0000000000000040 0x0000000000000040 0x0000000000000040
                 0x00000000000001f8 0x00000000000001f8  R      0x8
  INTERP         0x0000000000000238 0x0000000000000238 0x0000000000000238
                 0x000000000000001c 0x000000000000001c  R      0x1
      [Requesting program interpreter: /lib64/ld-linux-x86-64.so.2]
  LOAD           0x0000000000000000 0x0000000000000000 0x0000000000000000
                 0x0000000000000840 0x0000000000000840  R E    0x200000
  LOAD           0x0000000000000db8 0x0000000000200db8 0x0000000000200db8
                 0x000000000000025c 0x0000000000000268  RW     0x200000
  DYNAMIC        0x0000000000000dc8 0x0000000000200dc8 0x0000000000200dc8
                 0x00000000000001f0 0x00000000000001f0  RW     0x8
  NOTE           0x0000000000000254 0x0000000000000254 0x0000000000000254
                 0x0000000000000044 0x0000000000000044  R      0x4
  GNU_EH_FRAME   0x00000000000006fc 0x00000000000006fc 0x00000000000006fc
                 0x000000000000003c 0x000000000000003c  R      0x4
  GNU_STACK      0x0000000000000000 0x0000000000000000 0x0000000000000000
                 0x0000000000000000 0x0000000000000000  RW     0x10
  GNU_RELRO      0x0000000000000db8 0x0000000000200db8 0x0000000000200db8
                 0x0000000000000248 0x0000000000000248  R      0x1

 Section to Segment mapping:
  Segment Sections...
   00     
   01     .interp 
   02     .interp .note.ABI-tag .note.gnu.build-id .gnu.hash .dynsym .dynstr .gnu.version .gnu.version_r .rela.dyn .rela.plt .init .plt .plt.got .text .fini .rodata .eh_frame_hdr .eh_frame 
   03     .init_array .fini_array .dynamic .got .data .bss 
   04     .dynamic 
   05     .note.ABI-tag .note.gnu.build-id 
   06     .eh_frame_hdr 
   07     
   08     .init_array .fini_array .dynamic .got 
```

| 字段     | 含义                  |
| -------- | --------------------- |
| Offset   | 文件中偏移量          |
| FileSiz  | Segment内容大小       |
| VirtAddr | 虚拟内存的加载地址    |
| MemSiz   | 占用的虚拟内存大小    |
| PhysAddr | 物理内存加载地址      |
| Flags    | 属性，R读，W写，E执行 |
| Align    | 对齐方式              |

```
$ objdump -h prog
prog:     file format elf64-x86-64

Sections:
Idx Name          Size      VMA               LMA               File off  Algn
  0 .interp       0000001c  0000000000000238  0000000000000238  00000238  2**0
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
  1 .note.ABI-tag 00000020  0000000000000254  0000000000000254  00000254  2**2
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
  2 .note.gnu.build-id 00000024  0000000000000274  0000000000000274  00000274  2**2
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
  3 .gnu.hash     0000001c  0000000000000298  0000000000000298  00000298  2**3
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
  4 .dynsym       000000a8  00000000000002b8  00000000000002b8  000002b8  2**3
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
  5 .dynstr       00000084  0000000000000360  0000000000000360  00000360  2**0
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
  6 .gnu.version  0000000e  00000000000003e4  00000000000003e4  000003e4  2**1
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
  7 .gnu.version_r 00000020  00000000000003f8  00000000000003f8  000003f8  2**3
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
  8 .rela.dyn     000000c0  0000000000000418  0000000000000418  00000418  2**3
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
  9 .rela.plt     00000018  00000000000004d8  00000000000004d8  000004d8  2**3
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
 10 .init         00000017  00000000000004f0  00000000000004f0  000004f0  2**2
                  CONTENTS, ALLOC, LOAD, READONLY, CODE
 11 .plt          00000020  0000000000000510  0000000000000510  00000510  2**4
                  CONTENTS, ALLOC, LOAD, READONLY, CODE
 12 .plt.got      00000008  0000000000000530  0000000000000530  00000530  2**3
                  CONTENTS, ALLOC, LOAD, READONLY, CODE
 13 .text         000001a1  0000000000000540  0000000000000540  00000540  2**4
                  CONTENTS, ALLOC, LOAD, READONLY, CODE
 14 .fini         00000009  00000000000006e4  00000000000006e4  000006e4  2**2
                  CONTENTS, ALLOC, LOAD, READONLY, CODE
 15 .rodata       0000000b  00000000000006f0  00000000000006f0  000006f0  2**2
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
 16 .eh_frame_hdr 0000003c  00000000000006fc  00000000000006fc  000006fc  2**2
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
 17 .eh_frame     00000108  0000000000000738  0000000000000738  00000738  2**3
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
 18 .init_array   00000008  0000000000200db8  0000000000200db8  00000db8  2**3
                  CONTENTS, ALLOC, LOAD, DATA
 19 .fini_array   00000008  0000000000200dc0  0000000000200dc0  00000dc0  2**3
                  CONTENTS, ALLOC, LOAD, DATA
 20 .dynamic      000001f0  0000000000200dc8  0000000000200dc8  00000dc8  2**3
                  CONTENTS, ALLOC, LOAD, DATA
 21 .got          00000048  0000000000200fb8  0000000000200fb8  00000fb8  2**3
                  CONTENTS, ALLOC, LOAD, DATA
 22 .data         00000014  0000000000201000  0000000000201000  00001000  2**3
                  CONTENTS, ALLOC, LOAD, DATA
 23 .bss          0000000c  0000000000201014  0000000000201014  00001014  2**2
                  ALLOC
 24 .comment      0000002b  0000000000000000  0000000000000000  00001014  2**0
                  CONTENTS, READONLY
```

| 字段     | 含义               |
| -------- | ------------------ |
| Size     | 段内容大小         |
| VMA      | 虚拟内存的加载地址 |
| LMA      | 物理内存加载地址   |
| File off | 文件的偏移量       |
| Algn     | 对齐方式           |

&emsp;&emsp;prog可执行文件一共有25个Section，9个Segment。section为ELF中不同的段，每个段都有自己的内容；例如.data段包含的是有初始值的数据，.bss段表示的是没有初始化值的数据，.text段代表着是代码。Segment与Section不同，Segment根据Section的属性不同划分，例如读写属性等。Segment服务于可执行文件的装载阶段，在可执行文件装载的时候，以Segment为段进行装载。

&emsp;&emsp;具体段内容的打印信息如下：

```shell
$ objdump -s prog
......
......
Contents of section .data:
 201000 00000000 00000000 08102000 00000000  .......... .....
 201010 01000000                             ....            
Contents of section .comment:
 0000 4743433a 20285562 756e7475 20372e34  GCC: (Ubuntu 7.4
 0010 2e302d31 7562756e 7475317e 31382e30  .0-1ubuntu1~18.0
 0020 342e3129 20372e34 2e3000             4.1) 7.4.0.   
```

> 在可执行文件中.bss段表示的是未初始化的数据，是没有初始化值的数据，所以可执行文件中.bss是没有是内容的。

## 二、进程

&emsp;&emsp;进程是程序一次的执行，是计算机科学中最深刻、最成功的概念之一。进程拥有独立的地址空间，一般情况下，包括代码区域（text region）、数据区域（data region）和堆栈（stack region）。

<img src="https://raw.githubusercontent.com/shiroJin/image-storage/master/image_load_process.png" alt="screenshot" style="zoom:50%;" />

## 三、虚拟内存

&emsp;&emsp;概念上而言，虚拟内存被组织为一个由存放在磁盘上的连续的字节单元的数组，每个字节都有唯一的虚拟地址（虚拟内存最小的单位的字节，每个地址都代表着一个字节）。操作系统采用的是按需加载页的方式加载程序，当虚拟内存中页被引用时才会将磁盘中对应位置的内容加载进物理内存。计算机系统中物理内容达到上限时，会将物理内存中页的内容换出到磁盘中，然后做为一个新的页。内存和磁盘两者的读写速度相差甚远，即使是固态硬盘，读写速度也远不如内存。在手机系统中，当内存达到上限时，采取的措施则是杀死进程，即常见的内存溢出崩溃。

&emsp;&emsp;每个进程都被分配自己独立的页表，页表常驻内存中，页表是页表条目（PTE）的数组，页表条目由一个有效位（valid bit）和一个地址组成。

- valid bit：占一个位，值为0或1，标识这个这页数据有没有被缓存到物理内存中。
- address：硬盘地址或者是物理内存地址。

<img src="https://raw.githubusercontent.com/shiroJin/image-storage/master/image_load_vm.png" alt="screenshot" style="zoom:50%;" />

&emsp;&emsp;虚拟内存的存在简化了链接、加载、共享和内存分配，独立于物理内存，让复杂确认地址的工作交给地址翻译器——MMU。

### 一、地址翻译（MMU）

&emsp;&emsp;地址翻译实质是对虚拟地址和物理地址的映射，表达式如下：` MAP：VAS→PAS *U* 未缓存`

<img src="https://raw.githubusercontent.com/shiroJin/image-storage/master/image_load_mmu.png" alt="screenshot" style="zoom:50%;" />

&emsp;&emsp;从上述结构中看出虚拟地址由虚拟页号和偏移量组成，根据虚拟地址可以从页表中查询到页表条目（PTE），PTE有效位推断出页有没有被缓存到物理内存中，如果没有缓存在物理内存中，则缺页处理。如果页已经被缓存，则根据PTE查询到物理地址，最后查询到物理内存中的内容。

***CPU中地址翻译的具体流程***

* 缓存命中
  1. 处理器将虚拟地址传送给MMU
  2. MMU根据虚拟地址生成PTE地址，并请求高速缓存/主存
  3. 高速缓存/主存返回PTE
  4. MMU根据PTE生成物理地址，并请求高速缓存/主存
  5. 高速缓存/主存返回请求的请求的数据<img src="https://raw.githubusercontent.com/shiroJin/image-storage/master/image_load_cache_hit.png" alt="screenshot" style="zoom:50%;" />
     VA：虚拟地址&emsp;MMU：地址翻译器&emsp;PTEA：页表条目地址&emsp;PET：页表条目&emsp;PA：物理地址

*  缓存未命中，缺页处理
  1. 处理器将虚拟地址传送给MMU
  2. MMU根据虚拟地址生成PTE地址，并请求高速缓存/主存高速缓存/主存返回PTE
  3. MMU根据PTE构造物理地址时，发现还未被缓存
  4. 触发缺页异常。
  5. 内核缺页异常处理程序工作缺页处理程序确定牺牲页，如果牺牲页中数据有过修改，则会写回磁盘中。
  6. 将需要缓存的数据换入物理内存中，更新PTE。
  7. 控制权返回到原来的进程中，MMU重新进行地址翻译的工作。<img src="https://raw.githubusercontent.com/shiroJin/image-storage/master/image_load_cache_unhit.png" alt="screenshot" style="zoom:50%;" />
  
### 二、ELF内存分布

  ```
  Program Headers:
    Type           Offset             VirtAddr           PhysAddr
                   FileSiz            MemSiz              Flags  Align
    LOAD           0x0000000000000000 0x0000000000000000 0x0000000000000000
                   0x0000000000000840 0x0000000000000840  R E    0x200000
    LOAD           0x0000000000000db8 0x0000000000200db8 0x0000000000200db8
                   0x000000000000025c 0x0000000000000268  RW     0x200000
  ```

  &emsp;&emsp;可执行文件prog的两个LOAD类型的Segment。第一个Segment的虚拟地址从0开始，内存大小为0x840，对齐方式为0x200000，在文件中偏移量为0，大小为0x840。一般虚拟内存的大小大于或等于文件内容的大小，至于为什么会出现大于，因为.bss段在文件中是没有内容的，但在内存中需要给他分配内存。

  再看第二个Segment，它的起始虚拟地址为0x200db8，大小为0x25c，文件中的偏移量为0xdb8，对齐同样是0x200000。这里不免会有些疑惑，第一个segment的起始地址从0开始，对齐为0x200000，按计算第二个segment虚拟内存的起始地址应该为0x200000，但事实上是0x200db8。

  &emsp;&emsp;这其实是ELF的一种规范，在[ELF标准文档](http://refspecs.linuxbase.org/elf/elf.pdf)中有一段描述:

  > Loadable process segments must have congruent values for p_vaddr and p_offset, modulo the page size.
  >
  > 虚拟地址对页大小取模必须等于文件偏移量对页大小取模。

  &emsp;&emsp;在[动态链接文档](https://docs.oracle.com/cd/E19683-01/816-1386/6m7qcoblk/index.html#chapter6-34713)还有一段描述：

  > Logically, the system enforces the memory permissions as if each segment were complete and separate The segments addresses are adjusted to ensure that each logical page in the address space has a single set of permissions. 

  &emsp;&emsp;其实这种内存地址和文件偏移量对齐一种优化，为了方便可执行文件更好的进行分页。下图为64k对齐的可执行文件的文件分布和虚拟内存分布图。为了保证在虚拟内存中，每个页只有一种权限集合，每个Segment都会被分配独立的虚拟页。而在物理内存中，两个Segment的文件内容可以在同一个物理页中，这个物理页会映射两次，一次到Text Segment的虚拟页和一次到Data Segment的虚拟页中。下图中text segment和data segment在文件中是在同一页上的，映射到物理内存中后也在同一页上。物理页映射到虚拟页上时，因为两个segment共享同一个物理页，因此，需要文件偏移量来分配同一页上不同区域的空间给虚拟页。例如下图data segment在物理页中，0x4000后的内容才是Segment的内容，因此Segment的虚拟地址从0x8064000开始，保持与文件偏移量对齐，更方便的进行映射。

  <img src="https://raw.githubusercontent.com/shiroJin/image-storage/master/Executable_Memory_Map.png" alt="screenshot" style="zoom:50%;" />


## 四、链接

&emsp;&emsp;在程序装载完成后，内核将控制权又交给了进程，进程根据ELF头部信息，启动链接器完成链接。首先描述下链接器的主要职责，一个c文件经过预处理、编译后生成的便是一个*.o*后缀的ELF文件，ELF文件中的<u>.rel</u>段就是需要重定位的段，在链接器链接完成后，rel段就会移除。下表为ELF文件主体结构，每个ELF文件都是由多个段组成的，一些复杂的ELF文件会涵盖很多不同的段，一些简单的ELF文件可能只包括少数的段，这些都是由代码中数据决定的。

**ELF文件结构如下：**

| Sections  | 描述                                   |
| --------- | -------------------------------------- |
| .text     | 代码段                                 |
| .data     | 未初始化和为0的全局变量/静态变量。     |
| .bss      | 未初始化的全局变量                     |
| .symtab   | 符号表                                 |
| .rel.text | 模块中引用或定义的全局函数的重定位信息 |
| .rel.data | 模块中引用或定义的全局变量的重定位信息 |
| .strtab   | 字符串表                               |
| ...       | 一些其他的段                           |

&emsp;&emsp;链接的过程分为两种，静态链接和动态链接。静态链接在生成可执行文件的时候就已完成，动态链接在程序装载的时候由动态链接器执行。

&emsp;&emsp;**链接器的主要工作内容为两部分：**

> 1. **符号解析**
>    将符号的引用和符号的定义关联起来，生成一个完整的符号表。
>
> 2. **重定位**
>    单个文件中引用的符号内存地址是不正确的，需要在链接过程中进行纠正。

### 静态链接

&emsp;&emsp;每个ELF都有自己的局部符号和全局符号，在编译的产物中，不同的ELF文件会引用到其他ELF文件的符号，静态链接器的工作就是在经过地址分配、符号解析、符号重定位后将多个文件中引符号重定位，不同文件中相同段的合并，最后合成一个可执行文件。

#### 静态链接结构

&emsp;&emsp;可重定位文件中保存着重定向信息，链接器根据文件中的重定向信息对文件进行重定位。

* .rel.data

  数据段的重定向信息

* .rel.text
  代码段的重定向信息

#### 静态链接过程

##### 一、段空间和地址分配

&emsp;&emsp;扫描所有的输入目标文件，收集所有目标文件的符号表到一个全局符号表中，将相同的段合并。在所有段合并成一个完整的文件后，符号的地址也随之可以确定下来。

<img src="https://raw.githubusercontent.com/shiroJin/image-storage/master/linker_1.png" alt="linker_1" style="zoom: 33%;" />

<p style="text-align:center;font-style:italic;">section合并</p>
>**段合并**：程序在装载的时候，段的装载地址和空间的对齐单位是页，对于x86设备来说，页的大小是4096字节，即4kb。对于一个仅仅一个字节的段，它占据的空间也是4kb，这就造成了极大的空间浪费。因此相同段需要合并。

##### 二、符号解析

&emsp;&emsp;每个编译文件中都有自己符号表，符号有些事静态的，有些是全局的。对于全局的，不同的文件定义的、引用的符号各不相同。全局变量的符号放在Common段中，静态变量放在bss段中，因为静态变量的大小是确定的，而全局变量因为弱类型进制的缘故，全局变量的符号大小时不确定的。对于静态的符号，段的地址确定后，静态符号的地址也就能确定。对于全局变量，不容的文件中可能定义了相同的符号，对于有冲突的符号需要进行符号决议，最终确定符号的地址。

> **符号决议：**
> 确定符号的定义，符号的定义有强弱之分，强符号覆盖弱符号。若在不同段中定义了相同的强符号，链接就会失败。

##### 三、符号重定位

&emsp;&emsp;对于单个文件，引用了其他文件的符号，这个符号的地址是未确认的，只是一个占位符，但不是实际符号的地址。例如，一个文件中引用了foo函数，但是foo函数定义在其他文件中，当编译器编译这个文件时，编译器并不知道foo这个符号的地址。对于单个的ELF文件，foo函数这个符号是没有定义的。符号的重定位是链接器是主要工作内容。链接器将没有地址的符号赋予正确的地址。

&emsp;&emsp;符号重定位有两种方式，绝对寻址和相对寻址。

> A：保存在被修正位置的值
>
> B：被修正的位置（相对于段开始的偏移量或者虚拟地址）
>
> P：符号的实际地址
>
> * 绝对寻址 S+A
>
> * 相对寻址 S+A-P

##### 四、输出链接后的文件

&emsp;&emsp;待所有符号重定位结束后，得到一个.out文件，一个可执行文件，文件中只有.bss，.text，.data段。没有符号表之类的段。

### 动态链接

&emsp;&emsp;前面讲到静态链接在生成可执行文件前就完成了链接过程，静态链接库也会编译到可执行文件中，在可执行文件中都有一份副本。如果每个可执行文件中都有同一个静态库的副本，久而久之会导致一些内存浪费的问题。如果可执行文件可以使用的库可以共享，那就可以减少内存开支，动态链接库也就孕育而生。其中共享库的符号解析和重定位的工作由动态链接器完成。

<img src="https://raw.githubusercontent.com/shiroJin/image-storage/master/linker_1.png" alt="linker_2" style="zoom:50%;" />

<p style="text-align: center;font-style: italic">动态库内存分配</p>
&emsp;&emsp;动态链接库的代码在多个进程间是共享的，动态库只会加载到内存中一次。当随后的进程链接动态库时不需要再次加载进内存，从而到达了节省内存的目的。

&emsp;&emsp;在静态链接阶段，根据动态库的符号表，动态库中的符号会被标记。在静态链接的时候不会对标记的符号进行重定位，重定位的过程留在了程序加载时。

&emsp;&emsp;*动态库的装载地址在编译阶段无法确定，只有在装载时才能确定*。因此动态库的符号重定位与静态库的重定位方式不同。另外，对于每个动态库，**多个进程之间代码段是共享的，数据段在每个进程中都会有自己独立的副本**。

#### 动态链接技术

&emsp;&emsp;为了使共享模块的代码段加载到内存的任何位置而无需链接器进行修改，现代系统在编译动态库的时候生成位置无关代码，而无需重定位。对于同一个目标模块中的符号引用，通过相对寻址的方式进行引用，而对于外部模块的符号则需要额外的方式去引用。

##### 一、位置无关代码PIC（Position Independent Code）

&emsp;*&emsp;*模块内的符号引用通过相对地址访问、而模块间的符号引用通过GOT间接引用。

* 模块内函数调用及跳转

  内部函数的相对位置不会改变，他们的相对的偏移量是确定的，因此内部函数调用通过相对寻址进行引用。

  > call + 相对偏移量

* 模块内数据访问

  跟内部函数一样，模块内部的数据距整理个模块的起始地址的相对偏移量是固定的，根据相对偏移量可以找到对应的数据，即pc+offset。

  > x64（64位）：RIP寄存器 + 偏移量
  >
  > x86（32位）：因为x86下没有rip指针，所以想要拿到pc，需要通过__x86.get_pc_thunk.ax函数获取。然后再加上偏移量。

* 模块间函数调用及跳转

  模块间的函数符号地址与自身的装载地址无关，与其他模块的装载地址相关，因此无法直接通过相对寻址的方式进行调用或跳转，ELF会在数据段建一个全局偏移量表（GOT，Global Offset Table），一个指向外部符号地址的数组，每一项存放的是一个8字节的地址。因为GOT在数据段，所以每个进程都有一个副本。当代码中需要引用外部符号时，先访问GOT条目，通过间接寻址对其进行间接引用。当模块被装载时，链接器会装载依赖的其他模块，然后重定位GOT条目的地址。由此可以发现，代码段的符号重定位变成了数据段的符号重定位。

  > 首先将GOT条目地址通过相对寻址压入寄存器，然后通过寄存器间接寻址引用。
  >
  > <img src="https://raw.githubusercontent.com/shiroJin/image-storage/master/linker_got.png" alt="screenshot" style="zoom: 33%;" />

* 模块间数据访问

  模块间数据的访问处理方式与上述函数调用方式相同，都是通过GOT进行间接引用。

##### 二、过程链接表PLT（Procedure Linkage Table)

&emsp;&emsp;动态链接器在重定位GOT中地址的过程发生在程序启动的时候，复杂的GOT重定位也会降低程序的启动速度。为了解决这一问题，共享库中引入过程链接表（PLT，Procedure Linkage Table），将GOT地址的绑定**延迟**到第一次引用时。GOT表也拆分为两张表，.got和.plt.got，.got存放的是变量的引用，.plt.got存放的是函数的引用。每个函数的GOT条目都会对应一个PLT条目。与GOT不同的是，PLT为代码段。每个PLT条目都是一段代码。以为下foo函数的PLT条目：

```c
jump *(foo@got)
push offset
jump PLT0
```

&emsp;&emsp;foo@got中存放着foo函数的地址，foo@got的初始地址为下一条指令的地址，即执行``push offset``，其中offset为偏移量，也就是这个PLT条目的序号，链接器会根据offset的值才重定位引用函数的地址。在PLT结构中，PLT[0]为公共的重定位函数——_dl_runtime_resolve。_dl_runtime_resolve函数位于动态链接器中，对入参函数进行重定位，并将重定位的地址写入foo@got。当第二次调用foo函数，将直接跳转到foo函数的地址。这样的设计非常巧妙，一个符号的重定位过程只会发生一次，随后的调用无需再进行重定位，相比之下只是多了一个间接引用，有效的解决了启动时重定位的时间花费。

> 外部模块函数的引用的过程如下：
>
> 1. 函数的引用首先跳转到PLT条目
> 2. PLT条目中通过GOT条目进行间接跳转，由于GOT条目的初始值为下一条指令，程序继续执行PLT条目的下一个指令。
> 3. 将函数的序号压入栈中，跳转到PLT[0]条目中
> 4. PLT[0]把GOT[0]压入栈中，GOT[1]存放的是动态链接器的第一个参数，然后再通过GOT[2]跳转到动态链接器中，动态链接器对引用的函数进行重定位，将重定位后的地址写入到GOT条目中，最后跳转到引用的函数。

#### 动态链接结构

&emsp;&emsp;在ELF文件中有.interp和.dynamic段描述动态链接器信息系。

* .interp
  为可执行文件动态链接的动态链接器不是系统固定的，而是在可执行文件中指定的。.interp段的内容就是动态链接器的地址。

  ```
  Contents of section .interp:
   0238 2f6c6962 36342f6c 642d6c69 6e75782d  /lib64/ld-linux-
   0248 7838362d 36342e73 6f2e3200           x86-64.so.2.
  ```

* .dynamic
  段中保存了动态链接所需要的基本信息，例如，所依赖的库，动态链接符号表的位置、动态链接重定位表的位置信息等等。

* .dynsym
  动态链接符号表

* .dynstr
  动态链接字符串表

* .rela.dyn
  .got段的重定向表，相对于静态链接中.rel.data

* .rela.plt
  .plt段的重定向信息，相当于静态链接中.rel.text

#### 动态链接过程

&emsp;&emsp;可执行文件的虚拟内存分配完成后，根据程序头部表，启动动态链接器进行重定位。

1. 动态链接器将可执行文件的符号表和链接器本身的符号表合并到一个**全局符号表**中。
2. 链接器收集可执行文件所依赖的共享对象到一个集合中。
3. 链接器开始装载所有的共享对象，如果共享对象依赖其他的共享对象，相应的共享对象也会被装载进来，整个装载的过程近似图遍历。链接器根据广度优先的顺序装载共享对象。共享对象被装载进来后，它的符号也会被合并到全局符号表中。
4. 所有的共享对象装载完成后，全局符号表包含所有动态链接需要的符号。
5. 链接器遍历所有的重定向条目，对其进行重定向，对引用地址进行修正。

## 五、结尾

&emsp;&emsp;至此，可执行文件的装载和动态链接完成，系统将控制权交给程序的入口函数——main函数。程序开始运行。

## 六、参考

《程序员的自我修养——链接装载与库》

《深入理解计算机系统》

[《Dynamic Linking》](https://docs.oracle.com/cd/E19683-01/816-1386/6m7qcoblk/index.html#chapter6-34713)

[《ELF》](http://refspecs.linuxbase.org/elf/elf.pdf)

