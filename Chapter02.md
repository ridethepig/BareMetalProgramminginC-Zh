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

首先,代码能够不被初始化就能正常工作,是因为变量和代码存在于同一内存空间里.bootloading的过程把RPi内核装载到RAM里执行,而GPU里的引导程序比ARM处理器先运行,因此,变量在img文件里的位置就成为了他们在内存里的位置了.(相信你还记得kernel.img被加载以后cpu才开始运行)