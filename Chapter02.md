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

事实上,