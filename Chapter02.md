# 第二章 - C语言运行时(The C RunTime)

在这一章里面,我们将更加深入的研究链接器和C-Library,最终我们将得到一个可用的C-Library链接.这看上去挺好的,不是吗?

我们的代码里暂时还没有C-Runtime(不是C-Library).在许多的嵌入式系统中,C-Runtime是必要的,否则很快就会崩溃.最显而易见的一件事,就是在大部分嵌入式操作系统中,static变量是不会被初始化的.这就是为什么我们先前的代码中并没有使用预初始化(pre-initialized)的变量.我们使用了预处理器来初始化变量而不是直接赋值.

## C-Runtime是如何启动的

让我们先来修改代码,用一个预初始化的变量来试试看:

part-2/armc-04:

```c
#include "rpi-gpio.h"

/** GPIO Register set */
volatile unsigned int* gpio = (unsigned int*)GPIO_BASE;

/** Simple loop variable */
volatile unsigned int tim;

/** Main function - we'll never return from here */
int main(void)
{
    /* Write 1 to the GPIO16 init nibble in the Function Select 1 GPIO
       peripheral register to enable GPIO16 as an output */
    gpio[LED_GPFSEL] |= (1 &lt;&lt; LED_GPFBIT);

    /* Never exit as there is no OS to exit to! */
    while(1)
    {
        for(tim = 0; tim &lt; 500000; tim++)
            ;

        /* Set the LED GPIO pin low ( Turn OK LED on for original Pi, and off
           for plus models )*/
        gpio[LED_GPCLR] = (1 &lt;&lt; LED_GPIO_BIT);

        for(tim = 0; tim &lt; 500000; tim++)
            ;

        /* Set the LED GPIO pin high ( Turn OK LED off for original Pi, and on
           for plus models )*/
        gpio[LED_GPSET] = (1 &lt;&lt; LED_GPIO_BIT);
    }
}
```

编译以上代码,你会发现一些变化:

* .img文件的大小现在是33kb,但之前仅仅有100b左右.
* 代码仍然可以工作!难道根本就不需要一个C-Runtime在调用main函数前来初始化gpio变量吗?

事实上,这个嵌入式操作系统很不同,因为我们向内存里加载了一整个二进制的映像.多数的系统都有一个不可变的内存段(Flash/ROM),用于存放可执行代码.系统中还会有一个可变的内存段(RAM),用于存放变量数据.变量存在于RAM中,每一个都有一个相应的内存中的位置.

当编译一个存在Flash中并使用RAM储存变量的可执行程序时,我们需要复制一份初始值给Flash中的变量,以便系统每一次启动时,可以用初始值来把这些变量给初始化.我们还需要存在于Flash中的代码来把这些值复制到变量里去.当然了,在main()函数被调用*之前*.

以上便是C-Runtime(CRT)所做的事情之一.这是一个通常由工具链自动生成的代码对象(code object).这通常不是唯一个被链接的对象,中断向量表(Interrupt Vector Table)也会被链接.有一个链接脚本会告诉链接器如何在内存布局中组织这些附加的代码./*这段其实在下没有看懂*/

以上这些事情你当然是不会知道的.你一般只需要通过命令行选择所使用的处理器或者嵌入式系统,然后合适的linker脚本以及CRT就会自动的被链接进你的代码里.

在此,我强烈建议你去看看你的安装的arm-none-gcc-eabi里的一些文件,比如`arm-none-eabi`子目录和`lib`子目录里的.C-Runtime的代码是一个二进制对象文件,文件名是`crt0.o`;而C-Library是对象文件(即`.o`文件)的打包(archive,即`.a`文件),文件名是`libc.a`(可能会有不同的名称,因为版本不同);你还可能会看到一些`.ld`文件.在`ldscripts`子目录下,会有标准的linker脚本,你如果记住他们在哪里,对于你来说还是很有价值的.GCC还有一种可以明确规定系统设置的机制,叫做specs files,其实就是GCC的编译配置文件.你可以通过这个机制来创建机器声明(machine specification),这样就可以指定目标机器了.你可以选择自己的带有命令行参数的specs file,否则GCC会使用默认的specs file.

specs files 被认为是GNU工具链里比较高级的内容,但是这真的是一个很方便的方式来应用指定硬件的编译设置.(embedded engineer还是很有必要会用这个的 :D)

现在,问题来了:为什么代码能够运行?为什么编译后的大小变为了33k?

首先,代码之所以能够不被初始化就能正常工作,是因为变量和代码存在于同一内存空间里.bootloading的过程把RPi内核装载到RAM里执行,而GPU里的引导程序比ARM处理器先运行,因此,变量在img文件里的位置就成为了他们在内存里的位置了.(相信你还记得kernel.img被加载以后cpu才开始运行)映像文件被加载到0x8000处然后执行,所以bootloader本质上做了通常是由CRT完成的任务,即把被初始化变量的初始值从不易失内存(non-volatile memory)到非不易失内存.Cool

接下来,我们来看看反汇编以后的代码.不要问我反汇编器在哪里.(命令就是arm-none-eabi-objdump).
onjdump会帮你完成所有的事情,且无需指定目标处理器.在示例代码里有disassemble.bat和disassemble.sh,你可以使用它们来反编译代码.执行完毕后,你应该会得到一个kernel.asm文件,内容如下(可能略有不同,RPi2):

```
Disassembly of section .text:

00008000 <main>:
    8000:   e59f2078    ldr r2, [pc, #120]  ; 8080 <main+0x80>
    8004:   e59f3078    ldr r3, [pc, #120]  ; 8084 <main+0x84>
    8008:   e5920000    ldr r0, [r2]
    800c:   e59f2074    ldr r2, [pc, #116]  ; 8088 <main+0x88>
    8010:   e5901004    ldr r1, [r0, #4]
    8014:   e3a0e000    mov lr, #0
    8018:   e3811701    orr r1, r1, #262144 ; 0x40000
    801c:   e5801004    str r1, [r0, #4]
    8020:   e3a0c801    mov ip, #65536  ; 0x10000
    8024:   e583e000    str lr, [r3]
    8028:   e5931000    ldr r1, [r3]
    802c:   e1510002    cmp r1, r2
    8030:   8a000005    bhi 804c <main+0x4c>
    8034:   e5931000    ldr r1, [r3]
    8038:   e2811001    add r1, r1, #1
    803c:   e5831000    str r1, [r3]
    8040:   e5931000    ldr r1, [r3]
    8044:   e1510002    cmp r1, r2
    8048:   9afffff9    bls 8034 <main+0x34>
    804c:   e580c028    str ip, [r0, #40]   ; 0x28
    8050:   e583e000    str lr, [r3]
    8054:   e5931000    ldr r1, [r3]
    8058:   e1510002    cmp r1, r2
    805c:   8a000005    bhi 8078 <main+0x78>
    8060:   e5931000    ldr r1, [r3]
    8064:   e2811001    add r1, r1, #1
    8068:   e5831000    str r1, [r3]
    806c:   e5931000    ldr r1, [r3]
    8070:   e1510002    cmp r1, r2
    8074:   9afffff9    bls 8060 <main+0x60>
    8078:   e580c01c    str ip, [r0, #28]
    807c:   eaffffe8    b   8024 <main+0x24>
    8080:   0001008c    andeq   r0, r1, ip, lsl #1
    8084:   00010090    muleq   r1, r0, r0
    8088:   0007a11f    andeq   sl, r7, pc, lsl r1

Disassembly of section .data:

0001008c <__data_start>:
   1008c:   20200000    eorcs   r0, r0, r0
```

第一行把PC(program counter)+120 里的`地址`的值,此处为0x1008c,赋值到r2里.你需要谨记文件被加载到0x8000处,因为几乎所有的ARM设备在此处都是一样的,所以链接器也知道如何处理.在基地址0x8000上加上偏移量0x808c(也就是0x1008c)地址处的值就是0x20200000,即gpio变量初始值(RPi2是0x3F200000).以上就是为什么没有任何明确的初始化也能正常运行的原因分析.Let's go on.

可执行映像末尾处的值可以用十六进制编辑器来查看。![Hex Editor View](/img/armc-04-kernel-img-hex-editor.jpg)

下一行却把PC+120里的'地址'放进了r3里，此处的值为0x10090，居然超出了映像文件大小。这个变量的之将会是未定义的。这就是tim variable（我也不知道是什么），它事实上不会被初始化为任何值，直到我们在第73行的一个for循环开始使用它。（如果你了解一些C语言标准的话，肯定就会知道这是个不正确的行为，这事儿过一会再讲）

接下来检查一下我们是否是正确的：

```
\arm-tutorial-rpi\part-2\armc-04>arm-none-eabi-nm kernel.elf

00010094 A __bss_end__
00010090 A __bss_start
00010090 A __bss_start__
0001008c D __data_start
00010094 A __end__
00010094 A _bss_end__
00010090 A _edata
00010094 A _end
00080000 N _stack
         U _start
0001008c D gpio
00008000 T main
00010090 B tim
```

很好，我们的猜想和[nm](https://sourceware.org/binutils/docs/binutils/nm.html)（一个gnu工具链里输出目标文件的符号清单的程序）所输出的结果一样。

##模拟

如果你在使用linux，那么你就可以使用可以模拟arm的GDB来调试程序。以下是我运行调试的一些结果，希望对于你的理解有所裨益：

```
brian@brian-PH67-UD3-B3 ~ $ arm-none-eabi-gdb
GNU gdb (7.6.50.20131218-0ubuntu1+1) 7.6.50.20131218-cvs
Copyright (C) 2013 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.  Type "show copying"
and "show warranty" for details.
This GDB was configured as "--host=x86_64-linux-gnu --target=arm-none-eabi".
Type "show configuration" for configuration details.
For bug reporting instructions, please see:
<http://www.gnu.org/software/gdb/bugs/>.
Find the GDB manual and other documentation resources online at:
<http://www.gnu.org/software/gdb/documentation/>.
For help, type "help".
Type "apropos word" to search for commands related to "word".
(gdb) target sim
Connected to the simulator.
(gdb) load kernel.elf
Loading section .text, size 0x8c vma 0x8000
Loading section .data, size 0x4 vma 0x1008c
Start address 0x8000
Transfer rate: 1152 bits in <1 sec.
(gdb) file kernel.elf
Reading symbols from kernel.elf...done.
(gdb) break main
Breakpoint 1 at 0x8000: file armc-04.c, line 45.
(gdb) run
Starting program: /home/brian/kernel.elf

Breakpoint 1, main () at armc-04.c:45
45      armc-04.c: No such file or directory.
(gdb) disassemble
Dump of assembler code for function main:
=> 0x00008000 <+0>:     ldr     r2, [pc, #120]  ; 0x8080 <main+128>
   0x00008004 <+4>:     ldr     r3, [pc, #120]  ; 0x8084 <main+132>
   0x00008008 <+8>:     ldr     r0, [r2]
   0x0000800c <+12>:    ldr     r2, [pc, #116]  ; 0x8088 <main+136>
   0x00008010 <+16>:    ldr     r1, [r0, #4]
   0x00008014 <+20>:    mov     lr, #0
   0x00008018 <+24>:    orr     r1, r1, #262144 ; 0x40000
   0x0000801c <+28>:    str     r1, [r0, #4]
   0x00008020 <+32>:    mov     r12, #65536     ; 0x10000
   0x00008024 <+36>:    str     lr, [r3]
   0x00008028 <+40>:    ldr     r1, [r3]
   0x0000802c <+44>:    cmp     r1, r2
   0x00008030 <+48>:    bhi     0x804c <main+76>
   0x00008034 <+52>:    ldr     r1, [r3]
   0x00008038 <+56>:    add     r1, r1, #1
   0x0000803c <+60>:    str     r1, [r3]
   0x00008040 <+64>:    ldr     r1, [r3]
   0x00008044 <+68>:    cmp     r1, r2
   0x00008048 <+72>:    bls     0x8034 <main+52>
   0x0000804c <+76>:    str     r12, [r0, #40]  ; 0x28
   0x00008050 <+80>:    str     lr, [r3]
   0x00008054 <+84>:    ldr     r1, [r3]
   0x00008058 <+88>:    cmp     r1, r2
   0x0000805c <+92>:    bhi     0x8078 <main+120>
   0x00008060 <+96>:    ldr     r1, [r3]
   0x00008064 <+100>:   add     r1, r1, #1
   0x00008068 <+104>:   str     r1, [r3]
   0x0000806c <+108>:   ldr     r1, [r3]
   0x00008070 <+112>:   cmp     r1, r2
   0x00008074 <+116>:   bls     0x8060 <main+96>
   0x00008078 <+120>:   str     r12, [r0, #28]
   0x0000807c <+124>:   b       0x8024 <main+36>
   0x00008080 <+128>:   andeq   r0, r1, r12, lsl #1
   0x00008084 <+132>:   muleq   r1, r0, r0
   0x00008088 <+136>:   andeq   r10, r7, pc, lsl r1
End of assembler dump.
(gdb) stepi
0x00008004      45      in armc-04.c
(gdb) info registers
r0             0x0      0
r1             0x0      0
r2             0x1008c  65676
r3             0x0      0
sp             0x800    0x800
lr             0x0      0
pc             0x8004   0x8004 <main+4>
cpsr           0x13     19
(gdb) stepi
0x00008008      45      in armc-04.c
(gdb) info registers
r0             0x0      0
r1             0x0      0
r2             0x1008c  65676
r3             0x10090  65680
sp             0x800    0x800
lr             0x0      0
pc             0x8008   0x8008 <main+8>
cpsr           0x13     19
(gdb) stepi
50      in armc-04.c
(gdb) info registers
r0             0x20200000       538968064
r1             0x0      0
r2             0x1008c  65676
r3             0x10090  65680
sp             0x800    0x800
lr             0x0      0
pc             0x800c   0x800c <main+12>
cpsr           0x13     19
(gdb) stepi
45      in armc-04.c
(gdb) info registers
r0             0x20200000       538968064
r1             0x0      0
r2             0x7a11f  499999
r3             0x10090  65680
sp             0x800    0x800
lr             0x0      0
pc             0x8010   0x8010 <main+16>
cpsr           0x13     19
(gdb) stepi
50      in armc-04.c
(gdb) info registers
r0             0x20200000       538968064
r1             0x93939393       2475922323
r2             0x7a11f  499999
r3             0x10090  65680
sp             0x800    0x800
lr             0x0      0
pc             0x8014   0x8014 <main+20>
cpsr           0x13     19
(gdb) stepi
45      in armc-04.c
(gdb) info registers
r0             0x20200000       538968064
r1             0x93939393       2475922323
r2             0x7a11f  499999
r3             0x10090  65680
sp             0x800    0x800
lr             0x0      0
pc             0x8018   0x8018 <main+24>
cpsr           0x13     19
(gdb) stepi
0x0000801c      45      in armc-04.c
(gdb) info registers
r0             0x20200000       538968064
r1             0x93979393       2476184467
r2             0x7a11f  499999
r3             0x10090  65680
sp             0x800    0x800
lr             0x0      0
pc             0x801c   0x801c <main+28>
cpsr           0x13     19
(gdb)
```

现在，让我们做一个实验。我之前把0x1008c（变量gpio的地址）分解为0x8000 + 0x808c是有原因的：

## part-2/armc05.c

```c

#include "rpi-gpio.h"

/** GPIO Register set */
volatile unsigned int* gpio = (unsigned int*)GPIO_BASE;

/** Simple loop variable */
volatile unsigned int tim;

/** Main function - we'll never return from here */
int main(void) __attribute__((naked));
int main(void)
{
    /* Write 1 to the GPIO16 init nibble in the Function Select 1 GPIO
       peripheral register to enable GPIO16 as an output */
    gpio[LED_GPFSEL] |= (1 << LED_GPFBIT);

    /* Never exit as there is no OS to exit to! */
    while(1)
    {
        for(tim = 0; tim < 500000; tim++)
            ;

        /* Set the LED GPIO pin low ( Turn OK LED on for original Pi, and off
           for plus models )*/
        gpio[LED_GPCLR] = (1 << LED_GPIO_BIT);

        for(tim = 0; tim < 500000; tim++)
            ;

        /* Set the LED GPIO pin high ( Turn OK LED off for original Pi, and on
           for plus models )*/
        gpio[LED_GPSET] = (1 << LED_GPIO_BIT);

        for(tim = 0; tim < 500000; tim++)
            ;

        /* Set the LED GPIO pin low ( Turn OK LED on for original Pi, and off
           for plus models )*/
        gpio[LED_GPCLR] = (1 << LED_GPIO_BIT);
    }
}
```

armc-05.c相较于armc-04.c，只是稍微加了代码。用nm看一看就会发现，增加代码后，`__data_start`段的首地址比之前往后了，而且数量正好和我们增加的代码相同。

```
C:\Users\Brian\Documents\GitHub\arm-tutorial-rpi\part-2\armc-05>arm-none-eabi-nm
 kernel.elf
000100c0 A __bss_end__
000100bc A __bss_start
000100bc A __bss_start__
000100b8 D __data_start
000100c0 A __end__
000100c0 A _bss_end__
000100bc A _edata
000100c0 A _end
00080000 N _stack
         U _start
000100b8 D gpio
00008000 T main
000100bc B tim
```

