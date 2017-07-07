# 第一部分 开始旅程

树莓派基本是做软件开发的，它自带了一个不错的linux发行版（Raspian）。但有时，我们仍需要一个“没有操作系统”（自行google：操作系统分类）的实时（real-time）操作系统。因此我觉得，做一个教程来利用这Great的硬件资源是极好的。这个教程借鉴了[Cambridge University Tutorials](http://www.cl.cam.ac.uk/freshers/raspberrypi/tutorials/)的教程的思路。他们的教程写的很好，但是他们并没有真正建立一个如所说的那样的操作系统。并且，他们用的是汇编而不是C语言。在此，我将用C代替汇编，模仿他们的教程。C语言编译器仅仅是将C代码转成汇编，并把汇编码转成可执行文件。

>我在此极力推荐去看看剑桥大学的教程（正如上文所提），它们很好。如果你会一点汇编的话，直接去点击上文档链接。这个教程提供和那个相似的体验，但是增加了对其背后过程的理解，而且它使用C语言。

## 树莓派的交叉编译器（BCM2835 & BCM2386)

[GCC ARM Embedded](https://launchpad.net/gcc-arm-embedded)项目提供了一个GCC工具链，Win/Linux皆可使用。你最好把它装上。在win上，你最好把它装在路径里没有空的的地方。在Linux上，你可以安装一个类似于“gcc-arm-none-eabi”的编译器。在终端里（会看这个教程的人，应该不用问“终端是什么”了吧），输入“arm-none-eabi-gcc”。如果你得到一个如下的回显，那么：恭喜你，安装成功！//在大部分linux发行版里，都是有这个包的，直接用包管理器下载就好了。

    >arm-none-eabi-gcc
    arm-none-eabi-gcc: fatal error: no input files compilation terminated.

## 编译器版本

我尝试了4.9版本的，但几个样例过不了（OI惯用语），它生成了错误的代码，用[4.7](https://launchpad.net/gcc-arm-embedded/4.7/4.7-2013-q3-update)的就没有这个问题了。[eLinux page](http://elinux.org/RPi_Software#ARM)给出了一个最好的编译选项：

    -Ofast -mfpu=vfp -mfloat-abi=hard -march=armv6zk -mtune=arm1176jzf-s

注意一下，-Ofast选项可能会出现一些不必要的麻烦，所以我建议使用更为传统的 -O2 选项。其他标志只是告诉GCC我们使用什么类型的浮点单元，告诉它去产生硬浮点代码（GCC可能会创建软件浮点支持），并告诉GCC ARM处理器的体系结构，来产生最佳的和兼容的汇编和机器码。

RPi2的架构和派1是不一样的。原来的ARM1176被换成了一块4核的ARM Cortex-A7处理器。因此，要使用不同的选项来使整对于Pi2的编译更有效：

    -O2 -mfpu=neon-vfpv4 -mfloat-abi=hard -march=armv7-a -mtune=cortex-a7

正如你从[ARM specification of the Cortex A7](http://www.arm.com/products/processors/cortex-a/cortex-a7.php)文档中所看到的，它有一个VFPV4浮点处理器和一个NEON引擎。以上信息可以从[GCC ARM options page](https://gcc.gnu.org/onlinedocs/gcc/ARM-Options.html)中找到。

## 了解“编译器”和“链接器”

为了使用C编译器，我们需要理解编译器做什么和链接器做什么来生成可执行代码。编译器将C语句转换成汇编程序，并对汇编指令进行优化。这实际上是所有C编译器都会做的！然后，C编译器隐式调用汇编器将该文件（通常是临时的）装配到对象文件中。其中会有可重定位的机器代码和供连接器使用的符号信息。现代的C编译器把汇编码通过管道（自行google）传给汇编程序，所以没有中间文件，因为创建文件比通过一个程序将数据从一个程序传递到另一个程序要慢得多。链接器的工作是将所有内容链接到一个可执行文件中，它需要链接程序脚本。链接器脚本告诉链接器如何组织各种对象文件。链接器将按照链接器脚本中的规则排列所有对象，并将符号转成地址。此时，我们将渐渐明白，C程序不仅仅是我们键入的代码。在C程序运行前，有一些基本的事情会被处理。例如，一些变量需要初始化为某些值，一些变量需要初始化为0。这是由一个对象文件处理的，通常由链接器隐式链接，因为链接器脚本将包含对它的引用。目标文件被称为crt0.o（C Runtimt Zero）。这些代码使用连接器可以处理的符号，来清理初始化的变量开始和结束的地方，为了清零这个内存段。（说实话，我没看懂）它一般会设置一个堆栈指针，它总是包括一个对于_main的调用。重要提示：出现在C代码中的符号，在汇编里会在前面加上一个下划线。（即 main -> _main）

>这段附上原文（基本是直译的）
>This code uses symbols that the linker can resolve to clear the start of the area where initialised variables starts and ends in order to zero this memory section. It generally sets up a stack pointer, and it always includes a call to _main. Here’s an important note: symbols present in C code get prepended with an underscore in the generation of the assembler version of the code. So where the start of a C program is the main symbol, in assembler we need to refer to it as it’s assembler version which is _main.

## Github

所有的代码都可以在这个[Github repo](https://github.com/BrianSidebotham/arm-tutorial-rpi)中找到，所以你可以去把它下载下来，并在学习此教程时，编译和修改它们。首先让我们来看看如何编译最简单的那个。

## part-1/armc-00

```
int main(void)
{
    while(1)
    {

    }

    return 0;
}
```
在part-1/armc-00目录下，有针对于不同版本树莓派的源码编译脚本。总共有三个版本：原版的是名为built.bat/sh；B+有增加的IO连接口，但它仍然是V1的板子，它的文件名是build-rpi-bplus.bat/sh；最后，V2的板子有扩展的IO口以及四核处理器，它的文件名是build-rpi-2.bat/sh。

V1的板子装的是Broadcom BCM2835 (ARM1176)而V2的板子用的是BCM2836
(ARM Cortex A7).因为处理器的不同，我们使用不同的编译命令。

```
arm-none-eabi-gcc -O2 -mfpu=vfp -mfloat-abi=hard -march=armv6zk -mtune=arm1176jzf-s arm-test.c
```

```
arm-none-eabi-gcc -O2 -mfpu=vfp -mfloat-abi=hard -march=armv7-a -mtune=cortex-a7 arm-test.c
```

GCC成功地编译了代码（因为这里没有C语言的错误），但连接器会显示如下错误信息：

```
.../arm-none-eabi/lib/fpu\libc.a(lib_a-exit.o): In function `exit':
exit.c:(.text.exit+0x2c): undefined reference to `_exit'
collect2.exe: error: ld returned 1 exit status
```

在上面的一行命令中，我们调用了C语言编译器、汇编器和链接器。C编译器为我们做了大部分琐碎的任务，使我们的生活更加方便，但是因为我们是嵌入式工程师（不是吗？）我们必须了解编译器、汇编器和链接器在非常底层的环境下如何工作，因为我们一般工作在定制的系统上，我们得熟练地向工具链做出描述。

由错误信息可见，这里有一个` _exit `符号失踪了。这个符号被我们使用的C库引用。实际上它是一个系统调用。它被设计为由操作系统实现。它将在程序终止时被调用。在我们的例子中，我们的程序是我们自己的操作系统，我们的程序是唯一运行的，事实上，我们永远不会退出，所以我们不需要真正担心它。系统调用可以是空白的，只需要提供它们，以便链接器解析符号。C函数库有要求系统调用，有是它们已经被实现为空函数，或是一个固定的功能。你可以看看[newlib documentation on system calls](http://sourceware.org/newlib/libc.html#Stubs)来了解系统调用。Newlib是一个开源的轻量级C函数库。C函数库提供了所有的在biaozC头文件中的C语言功能。比如`stdio.h`, `stlib.h`, `string.h`，等等。

在这一点上，我想提醒的是，标准的Hello World示例不会在没有操作系统的此处工作，并且它确实是一个未被实现的系统调用，因此它不能成为我们的第一个示例。在printf(...)函数的最底端，包括了一个写操作函数——“write” ——这个函数被所有C函数库里需要写入文件的函数调用。对于printf函数，它需要向文件stdout写入。通常来说，一个正在运行stdout的OS产生在屏幕上可见的输出，并且它可以被OS用管道传输给另一个文件系统上的文件。如果没有OS，stdout通常会被打印到一个UART端口上，这样你就可以在你的远程连接的PC屏幕上看到输出量。我们在以后的教程里会考虑如何实现这个。let's move on...

解决链接问题最简单的方法是提供一个最小的退出函数来满足链接器。因为它永远不会被使用，所有我们需要做的是把连接器关掉，让它解决_exit。现在我们可以编译代码的下一个版本了：

## part-1/armc-01

```
int main(void)
{
    while(1)
    {

    }

    return 0;
}

void exit(int code)
{
    while(1)
        ;
}
```

在exit函数里放置一个无限循环是很重要的。在本来不是被设计为用来编写OS的c lib里(所以是 arm-NONE-eabi-*)，_exit被标记为无返回的函数。我们必须确保它不会返回，否则我们将会得到一个关于它的警告。 _exit函数的原型总是包含一个int型的退出代码。现在再使用相同的生成命令，我们就会得到一个clean build。Yay！（这个不大会翻，比较的传神）但是一个真正的问题出现了。我们需要提供自己的链接脚本和C启动代码，才能提供C lib底层的操作系统。当然，我们可以使用GCC的命令 `-nostartfiles`，就是不使用任何的C语言启动程序，我们也不再需要exit函数了。

## 了解处理器

就像在剑桥大学的那份教程中的那样，我们将复制他们最初点亮led的示例，以便知道我们的代码是否正确运行。

### 树莓派的启动过程

首先，让我们来看一下pi处理器的boot原理。我们应该知道，博通的bcm2385有两个处理器，一个是videocore（TM）GPU：这就是为什么Raspberry Pi能良好运行媒体中心和其他操作系统。这两个处理器共享外围总线，也必须共享一些中断资源。

GPU在复位或开机时开始运行，它包含了读取MMC总线上sd卡的第一个FAT分区的代码。它搜索并加载bootcode.binw文件到内存并执行代码。反过来，bootcode.bin bootloader 在SD卡上搜索一个文件名为start.elf和config.txt文件，设定不同的内核设置。然后再次搜索SD卡，寻找kernel.img文件，然后把它加载到内存的一个特定的地址（0x8000），启动ARM处理器，在那个内存位置执行。GPU现在已经开启并运行，ARM处理器就会运行包含在kernel.img里的代码。start.elf文件包含运行在GPU上的代码来提供OpenGL等的需要。

因此，如果你想要运行自己的代码，你需要首先把它编译为可执行文件，并改名为kernel.img，把它粘到一个FAT格式的sd卡里。这张卡里还要有bootcode.bin和start.elf。最新的RPi firmware可以在[GitHub](https://github.com/raspberrypi/firmware)下载。bootloader可以在[boot sub-directory](https://github.com/raspberrypi/firmware/tree/master/boot)目录下找到。firmware里其余的部分是闭源的二进制多媒体系统驱动。它们没什么用，就不用管他们了。

这意味着当处理器开始运行我们的代码时，它已经启动并运行。时钟源和PLL设置已经决定了，并在被在bootloader中编程，缓解了我们的问题。我们只需要把一个已经运行的核心的设备寄存器弄乱就可以了。我也不是很擅长这种事情，通常我的代码首先会正确的设置时钟和PLL来初始化处理器,但GPU已经为我们设置好了基本的时钟方案。

我们要做的第一件事就是开启GPIO控制器。我们没有可以依赖的驱动因为没有操作系统在运行。然而bootloader所做的只是启动处理器，使其进入工作状态，准备好加载OS。

你需要下载一份[Raspberry-Pi BCM2835 peripherals datahsheet](http://www.raspberrypi.org/wp-content/uploads/2012/02/BCM2835-ARM-Peripherals.pdf) 。通过阅读数据手册，我们可以知道如何控制BCM2835的外设IO设备。我会指导大家如何使用GPIO外设设备——总是会有些gotcha's的。（在下才疏学浅，不知道gotcha's是什么）。

我们在直接写代码之前，最好先来阅读一些关于处理器的基本信息。最重要的是有关虚拟内存的信息。在数据手册的第五页上，我们可以看到一张处理器的IO图。作为嵌入式工程师，我们必须通过IO图来了解处理器的外设的地址，并且在一些情况下，还要知道如何编写连接脚本，比如在有多重地址空间的时候。

![ARM Virtual addresses](http://www.valvers.com/wp-content/uploads/2013/01/arm-c-virtual-addresses.jpg)

VC CPU总线和Broadcom Video Core CPU连在一起。虽然Video Core CPU是从SD卡里引导的，但是会被在kernel.img中代码被调用时，移交给ARM核心处理。所以我们不关心VC CPU总线的地址。

ARM物理地址（ARM Physical addresses）是处理器原来的IO map，因为此时MMU（ARM的Memory Management Unit）没有被启用。如果mmu被启用，我们就需要注意虚拟地址了。

在一个OS的内核运行之前，MMU也没有运行，因为他还没有被初始化。此时的核心正运行在内核模式。总线上的地址因此会通过它们的ARM物理地址被访问到。从图中我们可以看到，在RPi1中，VC CPU的地址0x7E000000被映射到ARM物理地址0x20000000。这一点很重要！

虽然没有任何的文档著名，但是RPi2的ARM IO基地址保被设置带0x3F000000而不是0x20000000。树莓派基金会并没有提供我们需要的完善的文档，事实上，他们的[attitude suggests](http://www.raspberrypi.org/forums/viewtopic.php?f=72&amp;t=98400)认为我们（软件工程师）是一群魔术师，并不需要任何的文档。（//惊！）真是遗憾！如果你是一名论坛的成员，请帮助我们要求提供更多的文档。作为工程师，尤其是工业上，我们决不能接受这样的生产商，所以我们会转向其他的硬件平台。事实上，我使用配备了TI Cortex A8 的Beaglebone Black开发板，它有一份很好的文档。

无论如何，基地址还是可以从uboot的补丁里找到。因为树莓派2使用了BCM2836核心，所以可以搜一下这个版本的uboot，会找到一个[patch for supporting the Raspberry-Pi 2(RPI2的支持补丁)](http://lists.denx.de/pipermail/u-boot/2015-February/204483.html).

再往下看文档，我们会发现关于GPIO外设的部分。(Chapter 6,page 89).

## 运行代码看输出

我们终于可以看到代码在RPi上运行起来了。接下来会继续使用Cambridge tutorials里的第一个例子，就是点亮LED灯的那一个。这就是一个类似于“HELLO,WORLD!”程序的例子。通常,嵌入式开发里的hello,world是点亮一个闪烁的LED灯，这样我们才会知道，处理器是连续运行的.我们待会再处理这件事.

GPIO外设在BCM2835上的基地址是0x7E200000.我们刚刚也知道了,处理器会将它转成物理地址0x20200000 (0x3F200000 for RPI2)这是GPIO寄存器集中度第一个寄存器--GPIO Function Select 0

我们需要设置GPIO来使用IO引脚.[Raspberry-Pi schematic diagrams](http://www.raspberrypi.org/wp-content/uploads/2012/10/Raspberry-Pi-R2.0-Schematics-Issue2.2_027.pdf)中,LED灯被连到了GPIO16上(Sheet 2, B5).这个led在低电平是被触发,这是一个标准的惯例.也就是说,要打开led,我们需要输出一个0--即这个引脚被处理器接到了0V上;要关掉它,就输出1--引脚被接到了VDD上.

不幸的是,缺乏文档还是很普遍,我们并没有RPI2或者B+的原理图.GPIO线似乎被重新接过了,就像Florin在评论区里所说的,RPi B+的LED接在了GPIO47上面.我已经在括号里注明了这些事项.

回到处理器手册上来.我们要做的第一件事就是把GPIO pin设置为output模式,也就是要把GPIO16(GPIO47 RPI+)设置为output.

` GPIO Function Select 1 `寄存器的第18至20位控制了GPIO16; ` GPIO Function Select 4 `寄存器的第21至23位控制GPIO47

在C语言里,我们会设置一个指针指向一个寄存器,并用这个指针来向里面写入值.这种指针通常有一个修饰符volatile,这样,编译器就不会做出一些意料之外(可能会出现问题)的操作.(关于这一点,自行google volatile的用法)如果我们不把这个寄存器(也就是那个指针)标记为volatile,那么编译器就可以任意地检查这个寄存器会不会被再次访问到,(在此程序里就不会),所以我们为了无论什么目的写入的数据,不会被程序使用,并且优化器可以随意的丢弃这个写入,以为它没有用.

这种效果是确实需要的,但只是外表上可视的,就是看到GPIO pin的模式的改变.我们通过volatile关键字通知编译器不要把任何关于此变量的事情都当作理所当然,它只需要做好我们通知它做的.

```
#ifdef RPI2
    #define GPIO_BASE 0x3F200000UL
#else
    #define GPIO_BASE 0x20200000UL
#endif

volatile unsigned int* gpio_fs1 = (unsigned int*)(GPIO_BASE+0x04);
volatile unsigned int* gpio_fs4 = (unsigned int*)(GPIO_BASE+0x10);
```

我们需要向功能选择寄存器里的相关位写入一个1来把GPIO设为output.此处我们需要知道一个事实,这个寄存器会在重置后变成0,所以我们作如下的设置:

```
#if defined( RPIPLUS ) || defined ( RPI2 )
    *gpio_fs4 |= (1<<21);
#else
    *gpio_fs1 |= (1<<18);
#endif
```

这段代码看上去有点乱,但我们以后会去慢慢优化它.现在我们只是想点亮led并且搞清楚它为什么会被点亮.

ARM GPIO外设进行IO的方式特别的有趣，和其他的处理器不太一样，它有一个SET寄存器和一个CLEAR寄存器。向SET里的任何一位写入1都会使相应的GPIO pin被设置为1(高电平);相应的,向CLEAR里的任何一位写入1都会让相应的GPIO pin 被CLEAR成0(低电平).这样做(把一个寄存器的每一位和一个pin关联在一起,并且可以直接设置pin脚底输出)其实是有原因的,但本教程篇幅所限,恕不详细讲解.

我们需要向CLEAR的1~16为写入1(即把他们的output设为0)来点亮led:

```
*gpio_clear |= (1<<16);
```

我们把所学的东西放在了一个小小的例子里,接下来就来看看整个代码并将它们编译运行吧!(终于到了愉快的复制粘贴代码的时间了)

### part-1/armc-02

```
/* The base address of the GPIO peripheral (ARM Physical Address)
   GPIO外设的基地址(ARM物理地址) */
#ifdef RPI2
    #define GPIO_BASE       0x3F200000UL
#else
    #define GPIO_BASE       0x20200000UL
#endif

#if defined( RPIBPLUS ) || defined( RPI2 )
    #define LED_GPFSEL      GPIO_GPFSEL4
    #define LED_GPFBIT      21
    #define LED_GPSET       GPIO_GPSET1
    #define LED_GPCLR       GPIO_GPCLR1
    #define LED_GPIO_BIT    15
#else
    #define LED_GPFSEL      GPIO_GPFSEL1
    #define LED_GPFBIT      18
    #define LED_GPSET       GPIO_GPSET0
    #define LED_GPCLR       GPIO_GPCLR0
    #define LED_GPIO_BIT    16
#endif

#define GPIO_GPFSEL0    0
#define GPIO_GPFSEL1    1
#define GPIO_GPFSEL2    2
#define GPIO_GPFSEL3    3
#define GPIO_GPFSEL4    4
#define GPIO_GPFSEL5    5

#define GPIO_GPSET0     7
#define GPIO_GPSET1     8

#define GPIO_GPCLR0     10
#define GPIO_GPCLR1     11

#define GPIO_GPLEV0     13
#define GPIO_GPLEV1     14

#define GPIO_GPEDS0     16
#define GPIO_GPEDS1     17

#define GPIO_GPREN0     19
#define GPIO_GPREN1     20

#define GPIO_GPFEN0     22
#define GPIO_GPFEN1     23

#define GPIO_GPHEN0     25
#define GPIO_GPHEN1     26

#define GPIO_GPLEN0     28
#define GPIO_GPLEN1     29

#define GPIO_GPAREN0    31
#define GPIO_GPAREN1    32

#define GPIO_GPAFEN0    34
#define GPIO_GPAFEN1    35

#define GPIO_GPPUD      37
#define GPIO_GPPUDCLK0  38
#define GPIO_GPPUDCLK1  39

/** GPIO寄存器设置 */
volatile unsigned int* gpio;

/** 简单的循环变量 */
volatile unsigned int tim;

/** Main函数 - 我们不会从这里返回 */
int main(void)
{
    /* 分配GPIO外设的地址 (使用ARM Physical Address) */
    gpio = (unsigned int*)GPIO_BASE;

    /* Write 1 to the GPIO16 init nibble in the Function Select 1 GPIO
       peripheral register to enable GPIO16 as an output */
    gpio[LED_GPFSEL] |= (1 &lt;&lt; LED_GPFBIT);

    /* 不要返回!因为没有可以返回到的地方(无家可归hhh) */
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

现在要把编译命令稍微改动一下:

```
arm-none-eabi-gcc -O2 -mfpu=vfp -mfloat-abi=hard -march=armv6zk -mtune=arm1176jzf-s -nostartfiles armc-2.c -o kernel.elf
```

链接器给出了一个警告,我们将待会处理它.重点在于链接器帮我们解决了一个问题.下面给出那个被忽略的警告:

```
.../arm-none-eabi/bin/ld.exe: warning: cannot find entry symbol _start; defaulting to 00008000
```

我们从编译中可以看出,标准的ELF格式的输出文件是一个被打包好的可执行文件,其中包含了OS可能会需要的信息.但我们只需要机器码,所以用如下命令解压这个文件:

```
arm-none-eabi-objcopy kernel.elf -O binary kernel.img
```

### 关于ELF格式的简介

[ELF](http://en.wikipedia.org/wiki/Executable_and_Linkable_Format) 是一种常用的文件格式，它里面有机器码和元数据(即metadata).元数据是一种好东西,在大部分现代的操作系统中运行程序可不是仅仅把程序加载到内存里,然后处理器执行就好了的,所以通常会有一个程序加载器(executable loader)通过元数据获取更多有关信息.比方说函数调用接口可能每个程序不一样,这意味着在程序里调用函数的时候,会有不同的调用协议(?)使用不同的寄存器代表不同的含义.程序加载器是否会让一个程序加载进内存都是可以确定的.ELF格式的元数据还包括了所有需要加载的共享对象(.so文件,也就是Windows下的.dll).如果任何一个被依赖的lib丢失的话,程序加载器也不会让这个程序被加载运行.这一切都是为了,也确实增强了系统的兼容性和稳定性.

但是我们并没有OS,bootloader也没有除了读取、直接复制kernel.img进内存0x8000地址处以外的加载器了.所以我们需要把ELF元数据给扔掉,只留下机器码就行了.

### 回到我们的例子里

最终得到的kerne.img文件应该只包含ARM机器码,大概几十个字节大小,你可能会注意到,kernel.elf会有大约34KB大小,不管了.重命名原来SD卡上的kernel.img(如果你之前装了系统的话)为kernel.img.old之类的东西,并把你自己编译的kernel.img复制进去.从这个SD卡启动之后,OK led就会一直亮下去.正常的启动过程是OK led先亮,然后熄灭.如果灯一直不亮,那么你编译链接,或者是代码写的有问题.如果led保持发光,那么你就成功了.

一个闪烁的led可能更适合说明代码确实在运行.让我们快点把代码换一下,来简单的让led闪烁起来.我们接下来还会去处理C library的问题,因为C library实在是不怎么实用.

编译part-1/armc-03里的代码,虽然代码清单和part-1/armc-02里的是一模一样的,但是生成脚本用objcopy命令把ELF格式转成了可以部署在SD卡上的二进制文件(正如上文所述,去掉了metadata).

然后,OK LED就闪了.^_^ :D

因为是第一节,所以我已经把每一种型号的RPi板子的kernel.img编译好了放在代码里了.你可以直接运行这些预编译的代码,看看效果,并和你自己的进行比较,以确保你能正确的编译这些代码.以后就要你自己编译代码了.

虽说这些代码的写法可能有些奇怪,但请务必这么写,肯定是有原因的.不过你可以自己进行改动,做一些实验,这里有一个提醒:
- 不要使用自动变量(就是不要用自动分配初始值的变量),也不要用初始化过的变量,因为暂时还没有C runtime的支持.

现在我们可以进行下一节了.