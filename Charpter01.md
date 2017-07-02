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

在一个OS的内核运行之前，MMU也也没有运行，因为他还没有被初始化。此时的核心正运行在内核模式。总线上的地址因此会通过它们的ARM物理地址被访问到。从图中我们可以看到，在RPi1中，VC CPU的地址0x7E000000被映射到ARM物理地址0x20000000。这一点很重要！

