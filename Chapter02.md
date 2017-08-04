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

第一行把PC(program counter)寄存器里的`地址`的值,此处为0x1008c,赋值到r2里.你需要谨记文件被加载到0x8000处,因为几乎所有的ARM设备在此处都是一样的,所以链接器也知道如何处理.在基地址0x8000上加上偏移量0x808c(也就是0x1008c)地址处的值就是0x20200000,即gpio变量初始值(RPi2是0x3F200000).以上就是为什么没有任何明确的初始化也能正常运行的原因分析.Let's go on.

