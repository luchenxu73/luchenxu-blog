---
title: "STM32H7系列在GCC环境下的ld链接文件配置"
description: "STM32H7B0开发描述"
keywords: "stm32h7,openocd,gcc,链接文件"

date: 2024-08-12T14:41:50+08:00

categories:
  - 嵌入式开发
tags:
  - stm32
  - gcc

toc: true
---



> 本文将在 `Clion` + `gcc` + `Openocd` 这一环境下搭建`stm32h7b0`的开发环境。重点涉及 `ld` 文件的修改，以便能够更好地适应H7系列多块RAM分区的利用。在修改`ld`文件的时候，读者应当理解基本的内存模型，并与启动文件`startup.s`联系起来。


STM32h7系列在Clion环境下的搭建

## LD文件基本分析

LD文件对于gcc编译器环境下的开发至关重要，例如：

1. 决定了你的程序中变量的存放位置。尤其是对于stm32h7系列，不同的外设能够访问到的内存区域不同。例如DMA1和DMA2无法访问DTCM，如果你的变量默认放在DTCM中，将无法使用dma进行搬运。
2. 一些应用要求你获取一些特殊的变量。例如 `int a = 1;` 这样带有初始值的变量，程序实际上需要在启动时从flash中将值拷贝到sram中。再例如，一些操作系统如 `threadx` 的动态内存分配，会需要一个 `first_unused_memory` 指针，指向空余内存的开头，也就是将程序中没有用到的内存用作动态内存利用起来，这意味着我们需要利用ld文件中的变量来获取到正确的内存位置。

下面，我们开始分析ld文件，首先从链接脚本的开头开始:
```c
/* Entry Point */
ENTRY(Reset_Handler)
```
凭借直觉，我们应当能猜出来开头的`ENTRY(Reset_Handler)`就是代指程序入口，至于`Reset_Handler`，我们知道它在启动文件里。此处无需我们多关心。现在我们接着往下看：

```c
/* Highest address of the user mode stack */
_estack = ORIGIN(DTCMRAM) + LENGTH(DTCMRAM);    /* end of RAM */
/* Generate a link error if heap and stack don't fit into RAM */
_Min_Heap_Size = 0x200;      /* required amount of heap  */
_Min_Stack_Size = 0x400; /* required amount of stack */

/* Specify the memory areas */
MEMORY
{
  ITCMRAM (xrw)  : ORIGIN = 0x00000000, LENGTH = 64K
  FLASH (rx)     : ORIGIN = 0x08000000, LENGTH = 128K
  DTCMRAM (xrw)  : ORIGIN = 0x20000000, LENGTH = 128K
  RAM (xrw)      : ORIGIN = 0x24000000, LENGTH = 1024K
  RAM_CD (xrw)   : ORIGIN = 0x30000000, LENGTH = 128K
  RAM_SRD (xrw)  : ORIGIN = 0x38000000, LENGTH = 32K
}
```

这一段代码里已经包含了不少信息。首先是`MEMORY`字段里分别对应了芯片的编址区域，`ORIGIN` 又显然是对应区域的起始位置。同时，我们从 `_estack` 可以看出，栈的高地址在DTCMRAM的末尾，又根据栈从高地址向低地址增长的特点，我们又明白栈应该是从DTCMRAM的末尾向DTCMRAM的高处增长，即栈的起始位置在 0x24000000 处。

接下来我们来看`Section`部分。我们重点关注一些段:

```c
.text :
{
    . = ALIGN(4);
    *(.text)           /* .text sections (code) */
    *(.text*)          /* .text* sections (code) */
    *(.glue_7)         /* glue arm to thumb code */
    *(.glue_7t)        /* glue thumb to arm code */
    *(.eh_frame)
    KEEP (*(.init))
    KEEP (*(.fini))
    . = ALIGN(4);
    _etext = .;        /* define a global symbols at end of code */
} >FLASH
```

这个text部分就是我们的code部分保存的位置，这些指令都保存在flash中。程序在执行时直接从flash加载指令。尾部的`>FLASH`就是对应上文`MEMORY`里的字段。对于H7系列，有一个ITCM可以专门保存指令，它与CPU直连，但是在这个链接文件里我们可以发现所有的text都被放到了flash里，因为里面有匹配规则`*(.text*)`和`*(.text*)`。具体我们可以看编译生成的map文件, 以`HAL_Init`函数为例，在map文件里搜索，发现生成`.text.HAL_Init`，对应的地址就在flash的编址区域里。

那么如果希望把一些函数放到 `ITCM` 里，我们需要单独处理，也就是需要在ld文件里开一个新的**ITCM段**，然后把对应的函数名匹配到这个段里。同时因为ram的断电丢数据的特殊性，我们还要在单片机启动时再把这些指令拷贝到ram里。具体的操作方式我们后面将会提及。

接下来我们看data段：

```c
/* used by the startup to initialize data */
_sidata = LOADADDR(.data);

/* Initialized data sections goes into RAM, load LMA copy after code */
.data : 
{
    . = ALIGN(4);
    _sdata = .;        /* create a global symbol at data start */
    *(.data)           /* .data sections */
    *(.data*)          /* .data* sections */
    . = ALIGN(4);
    _edata = .;        /* define a global symbol at data end */
} >DTCMRAM AT> FLASH

```

这个data段里保存的就是初始化值不为0的数据，当然它不包括局部变量，因为局部变量在栈上。需要注意的是，初始化值不为0的数据需要在flash中保存，再拷贝到ram的data段里。因为ram断电丢数据，丢完程序已经不知道你初始化值到底多少了，因此一定要在flash上记录这些值是多少。拷贝这一过程我们需要结合`startup_stm32h7b0xx.s`启动文件来看：

```armasm
Reset_Handler:
  ldr   sp, =_estack      /* set stack pointer */

/* Call the clock system initialization function.*/
  bl  SystemInit

/* Copy the data segment initializers from flash to SRAM */
  ldr r0, =_sdata
  ldr r1, =_edata
  ldr r2, =_sidata
  movs r3, #0
  b LoopCopyDataInit

CopyDataInit:
  ldr r4, [r2, r3]
  str r4, [r0, r3]
  adds r3, r3, #4

LoopCopyDataInit:
  adds r4, r0, r3
  cmp r4, r1
  bcc CopyDataInit
```

我们可以看到`Reset_Handler`就是程序的入口，对应ld文件里的ENTRY。进入`Reset_Handler`后我们首先加载栈地址，然后调用`SystemInit`，然后就开始了拷贝data段。`_sdata`和`_edata` 分别对应了data段的开始和结束，我们能在上文ld文件的data段里找到对应的变量名。然后`_sidata`就是在flash里data段的开始处。CopyDataInit的任务就是把`_sidata`处的内容开始，一直拷贝到`_sdata`处，需要拷贝的长度自然就是`_edata`减去`_sdata`。我们看到`r3`寄存器每次递增4，也就是offset，循环拷贝，直到 `r0+r3==r1` 为止。读者可以自行琢磨这段汇编代码。从这里我们需要明白，**ld文件里的变量是能够使用的**，后续我们也会在ld文件里定义一些变量来供我们使用。



接下来我们再看bss段：

```c
. = ALIGN(4);
.bss :
{
  /* This is used by the startup in order to initialize the .bss secion */
  _sbss = .;         /* define a global symbol at bss start */
  __bss_start__ = _sbss;
  *(.bss)
  *(.bss*)
  *(COMMON)
  . = ALIGN(4);
  _ebss = .;         /* define a global symbol at bss end */
  __bss_end__ = _ebss;
} >DTCMRAM
```

bss段是保存了未初始化的变量和初始化值为0的变量。因为这些变量只需简单地在内存里填充为0就行。同样也能在启动文件里发现对这片内存区域的处理：

```armasm
LoopCopyDataInit:
  adds r4, r0, r3
  cmp r4, r1
  bcc CopyDataInit
  
  /* Zero fill the bss segment. */
  ldr r2, =_sbss
  ldr r4, =_ebss
  movs r3, #0
  b LoopFillZerobss
  
FillZerobss:
  str  r3, [r2]
  adds r2, r2, #4

LoopFillZerobss:
  cmp r2, r4
  bcc FillZerobss
```

就在拷贝完data段后，就开始为bss段填充为0。`_sbss`和`_ebss`也同样能在ld文件中找到。

接下来就是堆，也到了文件的末尾：

```c
._user_heap_stack :
{
  . = ALIGN(8);
  PROVIDE ( end = . );
  PROVIDE ( _end = . );
  . = . + _Min_Heap_Size;
  . = . + _Min_Stack_Size;
  . = ALIGN(8);
} >DTCMRAM
```

堆从低地址向高地址增长，那么在此处我们可以看到，堆如果一直增长，那就会和栈撞上（栈从前面可知，从DTCMRAM的尾部向低地址增长）。根据该链接文件，我们判断是否超内存的方法就是从堆空间的起始地址处开始，加上堆和栈的大小，总和如果大于DTCMRAM的尾部，那么就说明内存不够了。

ld文件的简单剖析到此为止，接下来我们来实现一些典型的应用，例如：

1. 将大缓冲数组开辟到另一处RAM
2. 将代码放到ITCM
3. 开辟一个新的内存管理区域

## 如何将变量放置到另一处RAM中

由于stm32h7系列内存有很多个区域，将不同的变量划分到不同的RAM区域是一个很常见的操作，也是非常必要的，典型的原因有：

1. 不同内存区域在应用中的职责不同。例如，我们会默认把变量放在DTCM中，但是LCD的显存Buf通常会放到AXISRAM中。再例如，一些内存为了性能会开启缓存，但是一些用于DMA的内存不会开启缓存，为了方便操作，可能会把内存放到另一片内存区域中去。
2. 一些外设只能访问特定的内存区域。例如，DMA1和DMA2都无法访问DTCM，因此必须在其他地方开辟内存用于DMA。
  
总结下来，最典型的应用场景和需求就是，如何把一个变量或者数组在编译期放置到指定的RAM区域中。具体地，可以简单地分为以下两个步骤：

1. 在 `ld` 文件中添加对应的内存区域。ld 文件的修改可以照猫画虎地添加一个段。例如，我为AXI_SRAM创建了一个段：
```c
MEMORY  
{  
	DTCMRAM (xrw)      : ORIGIN = 0x20000000, LENGTH = 128K
	/* 注意一定要有这块内存 */
	RAM_D1 (xrw)      : ORIGIN = 0x24000000, LENGTH = 512K
	RAM_D2 (xrw)      : ORIGIN = 0x30000000, LENGTH = 288K  
	RAM_D3 (xrw)      : ORIGIN = 0x38000000, LENGTH = 64K  
	ITCMRAM (xrw)      : ORIGIN = 0x00000000, LENGTH = 64K  
	SDRAM (xrw)      : ORIGIN = 0xC0000000, LENGTH = 16M  
	FLASH (rx)      : ORIGIN = 0x90000000, LENGTH = 32M  
}

SECTIONS  
{
	/* The startup code goes first into FLASH */  
	.isr_vector :  
	{  
	  . = ALIGN(4);  
	  KEEP(*(.isr_vector)) /* Startup code */  
	  . = ALIGN(4);  
	} >FLASH
	
	/* 添加RAM_D1域的SRAM */
	.RAM_D1(NOLOAD) :  
	{  
	  . = ALIGN(4);  
	  *(.RAM_D1) 
	  *(.RAM_D1*)        
	  . = ALIGN(4);  
	} >RAM_D1

	/* 下省略 */
	/* ...... */
}
```

2. 为程序中的变量添加对应的修饰，告知编译器将变量分配到指定位置。与ac6编译器不同，GNU工具链下应当使用`__attribute__((__section__(x)))`语法：
```c
// 将 buf 放入 RAM_D1 中，注意不要忘记点号
__attribute__((__section__(".RAM_D1"))) uint8_t buf[1024];
```
当然，这么写比较繁琐，在`cdefs.h`中有`__section(x)`宏，这是一个依赖GCC编译器的拓展：
```c

// cdefs.h
#define __section(x)   __attribute__((__section__(x)))

// main.c
__section(".RAM_D1") uint8_t buf[1024];
```
然后，你只要使用到了这个变量，那么编译器就会将`buf`变量放置到`RAM_D1`中：
```c
// 使用该变量
buf[0] = 1;
```

> [!NOTE]
> 非默认内存区域的变量不应当设置非0初始值，如 `int a = 1;`。上述代码在程序运行时变量`a`的值依旧会被初始化为`0`，而非`1`。因为`a`的期望值`1`其实是保存在flash中，然后在程序启动时将`1`从flash拷贝到RAM中，这一过程可以在启动文件`startup_stm32h7b0xx.s`中的`LoopCopyDataInit`这一函数中。因此，如果不添加额外的拷贝过程，你在非默认内存区域给定的初始值都是无法正确在程序运行时初始化。


## 如何将代码放入 ITCM

从上面map文件的分析里看到，我们的函数都会编译成text前缀的玩意儿。例如`HAL_Init`函数，在map文件里，可以看到它变成了`.text.HAL_Init`。

## 如何获取空闲内存的位置



