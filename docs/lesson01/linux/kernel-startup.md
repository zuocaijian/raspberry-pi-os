## 1.4: Linux 启动流程

### 寻找入口点

在了解完了 Linux 项目结构以及它是如何构建的之后，下一个逻辑步骤就是找出程序的入口点。与大多程序不一样，这一步对 Linux 内核而言并不那么容易。 

我们准备要做的第一件事就是来看看 [arm64 linker script(链接脚本)](https://github.com/torvalds/linux/blob/v4.14/arch/arm64/kernel/vmlinux.lds.S)。我们先前已经看过了这个链接脚本 [在主 makefile 文件中是如何使用的](https://github.com/torvalds/linux/blob/v4.14/Makefile#L970)。从这一行代码，我们可以很容易的推断出在哪里可以找到针对于某一特定CPU架构的链接脚本。

需要提一下的是，我们准备查看的文件并不是一个真正的链接脚本 - 它是一个模板，通过实际值来替换这个模板中的宏来生成真正的链接脚本。但正是因为这个文件由大量的宏组成，才能方便的看出不同CPU体系架构链接脚本之间的区别点。

链接脚本中的第一段被称为 [.head.text](https://github.com/torvalds/linux/blob/v4.14/arch/arm64/kernel/vmlinux.lds.S#L96)。这个段对我们来说非常重要，因为入口点应该就定义在这里面。如果你仔细想想，就明白这是非常有道理的：在内核被加载后，二进制镜像文件中的内容被拷贝到内存中的某块区域，然后从这块内存区域的起始处开始执行。这意味着，只要通过寻找谁使用了 `.head.text` 这个段，我们就可以找到入口点。事实也确实如此, `arm64` 体系架构只有一个文件使用了 [__HEAD](https://github.com/torvalds/linux/blob/v4.14/include/linux/init.h#L90) 宏，这个宏展开就是 `.section    ".head.text","ax"` - 就是 [head.S](https://github.com/torvalds/linux/blob/v4.14/arch/arm64/kernel/head.S)这个文件。

 在 `head.S` 文件中我们能找到的第一个可执行的代码行，就是 [这一行了](https://github.com/torvalds/linux/blob/v4.14/arch/arm64/kernel/head.S#L85)。这里我们make perfect sense使用汇编器指令 `branch` 中的 `b` 来跳转到 [stext](https://github.com/torvalds/linux/blob/v4.14/arch/arm64/kernel/head.S#L116) 函数。这也是你的引导启动内核后执行的第一个函数。

接下来我们来探索一下 `stext` 这个函数内部做了哪些我们还不曾了解的事。首先，我们需要在 RPi 操作系统中实现类似的功能，这也是我们接下来几节课要讲的内容。我们马上要做的是先了解一下内核引导相关的几个及其关键的概念。

### Linux 引导加载器以及引导协议

当 Linux 内核引导的时候，它假定机器硬件是处于一种已准备好的 "已知状态" 下的。定义这种状态的一系列规则被称为 "引导协议"，对于 `arm64` 体系架构而言，这个协议文档记录在 [这里](https://github.com/torvalds/linux/blob/v4.14/Documentation/arm64/booting.txt)。除此之外，它还定义了一些其他的东西，比如，必须而且只能是在主 CPU 上开始执行，必须关闭内存映射单元，禁用全部中断等。 

好了，那么是谁负责把机器设置到这种已知状态的呢?通常，在内核加载和初始化之前会运行一个特殊的程序。这个特殊程序被称之为 `bootloader(引导加载器)`。引导加载器程序的代码一般是与机器硬件强相关的，树莓派就是一个很好的例子。树莓派有一个内置在板子上的引导启动程序。我们只能使用 [config.txt](https://www.raspberrypi.org/documentation/configuration/config-txt/) 文件来自定义它的一些行为。 

### UEFI 引导

不过，有一种引导加载程序可以内置在内核镜像本身里面。这种引导启动程序只能在支持 [Unified Extensible Firmware Interface (UEFI)(统一的可扩展固件接口)](https://en.wikipedia.org/wiki/Unified_Extensible_Firmware_Interface) 的平台上使用。支持 UEFI 的设备提供一系列标准的服务来运行软件，这些服务可以用来获取到关于机器本身及其兼容性的全部所必需的信息。 UEFI 还要求计算机硬件能够运行兼容 [Portable Executable (PE)](https://en.wikipedia.org/wiki/Portable_Executable) 格式的文件。Linux 内核 UEFI 引导启动程序利用了这样一种特性：它把 `PE` 头注入到 Linux 内核镜像的起始处，这样计算机硬件就会把这个镜像文件当成是一个普通的 `PE` 文件。这是在 [efi-header.S](https://github.com/torvalds/linux/blob/v4.14/arch/arm64/kernel/efi-header.S) 文件中完成的。这个文件定义了 [__EFI_PE_HEADER](https://github.com/torvalds/linux/blob/v4.14/arch/arm64/kernel/efi-header.S#L13) 宏，这个宏在 [head.S](https://github.com/torvalds/linux/blob/v4.14/arch/arm64/kernel/head.S#L98) 中有用到。

在 `__EFI_PE_HEADER` 中定义的一个重要属性是明确 [ UEFI 入口点的位置](https://github.com/torvalds/linux/blob/v4.14/arch/arm64/kernel/efi-header.S#L33) ，这个入口点本身可以在 [efi-entry.S](https://github.com/torvalds/linux/blob/v4.14/arch/arm64/kernel/efi-entry.S#L32) 中找到。从这个入口点的位置开始，你可以顺着代码看看 UEFI 引导加载程序究竟做了哪些事情(这些源代码相对而言比较直观)。不过我们就到此为止了，因为这一节中我们并不是要深入研究 UEFI 引导加载程序的详细细节，而是让你有一些关于 Linux 内核是如何使用 UEFI 的大概的印象。

### 设备树

当我开始研究 Linux 内核启动代码的时候，我发现这其中大量提及到 `Device Trees(设备数)`。它看起来像是一个关键概念，我认为有必要对这个概念进行讨论。

当我们在写 `树莓派 操作系统` 内核的时候，我们用 [BCM2835 ARM 外围设备手册](https://www.raspberrypi.org/documentation/hardware/raspberrypi/bcm2835/BCM2835-ARM-Peripherals.pdf) 来计算出某个特定内存映射寄存器所在的确切偏移量。这个值明显在每块板子上都会有所不同，很幸运，我们只需要支持他们其中的一个就行了。不过如我们要支持成百上千不同的板子该怎么办呢?如果我们在内核代码中硬编码每块板子的值得话，那将会变成一团糟。而且即使我们准备这样去做，我们要如何确定我们当前用的是那一块板子呢? 比如， `BCM2835` 就根本不提供能将这些信息传递给运行时内核的任何方法。

设备树为我们提供了上面提及到的问题的一种解决方法。它是用于描述计算机硬件的一种特殊格式。在 [这里](https://www.devicetree.org/) 可以找到设备树的规范说明。在内核执行前，引导加载程序选出合适的设备树文件并把它当作一个参数传递给内核。如果你看一下在树莓派 SD 卡引导区的那些文件，你会发现这里有大量的 `.dtb` 文件。 `.dtb` 被编译成设备树文件。你可以选一些放到 `config.txt` 文件中，来启用或者禁用树莓派的某些硬件。 [树莓派官方文档](https://www.raspberrypi.org/documentation/configuration/device-tree.md) 中有更多关于这些细节的相关描述。

好了，现在是时候来看一下真正的设备树文件是什么样的了。就把试着找出 [树莓派3代B型](https://www.raspberrypi.org/products/raspberry-pi-3-model-b/) 的设备树文件当作一个快速小练习吧。从这个 [文档](https://www.raspberrypi.org/documentation/hardware/raspberrypi/bcm2837/README.md) 中我们可以获知， `树莓派3代B型` 使用了一块叫做 `BCM2837` 的芯片。如果你搜索这个名字你就可以找到 [/arch/arm64/boot/dts/broadcom/bcm2837-rpi-3-b.dts](https://github.com/torvalds/linux/blob/v4.14/arch/arm64/boot/dts/broadcom/bcm2837-rpi-3-b.dts) 这个文件。正如你可能会看出来的那样， `arm` 体系架构下也包含了相同的文件。这是很有道理的，因为 `ARM.v8` 处理器同时也支持32位模式。

接着，我们可以发现 [bcm2837-rpi-3-b.dts](https://github.com/torvalds/linux/blob/v4.14/arch/arm/boot/dts/bcm2837-rpi-3-b.dts) 是属于 [arm](https://github.com/torvalds/linux/tree/v4.14/arch/arm) 体系架构。我们前面已经看到过，这些设备树文件可以引用另一个设备树文件。 `bcm2837-rpi-3-b.dts` 的情况就是这样子的 - 它只包含那些专为 `BCM2837` 定义的定义，并重用其他所有的定义。例如， `bcm2837-rpi-3-b.dts` 指出 [这个设备目前有1GB内存](https://github.com/torvalds/linux/blob/v4.14/arch/arm/boot/dts/bcm2837-rpi-3-b.dts#L18). 

正如我前面提及的那样， `BCM2837` 和 `BCM2835` 有着同样的外围设备文件，如果你跟踪头文件看下去，你可以发现 [bcm283x.dtsi](https://github.com/torvalds/linux/blob/v4.14/arch/arm/boot/dts/bcm283x.dtsi) 实际上定义了这些硬件的大部分。 

一个设备树定义由很多嵌套其他设备树的块组成。在最顶层我们通常能找到像 [cpus](https://github.com/torvalds/linux/blob/v4.14/arch/arm/boot/dts/bcm2837.dtsi#L30) 或 [memory](https://github.com/torvalds/linux/blob/v4.14/arch/arm/boot/dts/bcm2837-rpi-3-b.dts#L17) 这样的块。这些块的所代表的含义应当是非常清楚的。另一个在 `bcm283x.dtsi` 中能找到的有趣顶层元素是 [SoC](https://github.com/torvalds/linux/blob/v4.14/arch/arm/boot/dts/bcm283x.dtsi#L52)，它表示 [SoC(片上系统)](https://en.wikipedia.org/wiki/System_on_a_chip)，SoC告诉我们，所有的外围设备都通过内存映射寄存器直接映射到内存中的某块区域。 `soc` 也被用作所有外围设备的父元素。它的一个子元素是 [gpio](https://github.com/torvalds/linux/blob/v4.14/arch/arm/boot/dts/bcm283x.dtsi#L147)。这个子元素定义了 [reg = <0x7e200000 0xb4>](https://github.com/torvalds/linux/blob/v4.14/arch/arm/boot/dts/bcm283x.dtsi#L149) 属性，这个属性告诉我们这些 GPIO 内存映射寄存器 位于 `[0x7e200000 : 0x7e2000b4]` 这个区域。  `gpio` 的其中一个子元素 [定义如下](https://github.com/torvalds/linux/blob/v4.14/arch/arm/boot/dts/bcm283x.dtsi#L474)

```
uart1_gpio14: uart1_gpio14 {
        brcm,pins = <14 15>;
        brcm,function = <BCM2835_FSEL_ALT5>;
};
``` 
这个定义告诉我们，如果选择可选功能5用于 14 和 15 号两个针脚，那么这两个阵脚将连接到 `uart1` 这个设备上。你很容易就可以猜到， `uart1` 就是我们前面用过的 Mini UART。

关于设备树你需要知道的一个重要的点就是，其格式是可执行的。每个设备都可以定义他自己的属性以及嵌套的(属性)块。这些属性很明显传递给了设备驱动，驱动负责中断这些设备。但是内核是如何获取到两个相邻设备树之间所对应的(属性)块的呢?它使用 `compatible` 属性来做到这一点。比如，对 `uart1` 设备来说， `compatible` 属性被指定如下：

```
compatible = "brcm,bcm2835-aux-uart";
```

实际确实如此，如果你在 Linux 源代码中搜索 `bcm2835-aux-uart`，你可以找到一个对应的驱动，这个驱动被定义在 [8250_bcm2835aux.c](https://github.com/torvalds/linux/blob/v4.14/drivers/tty/serial/8250/8250_bcm2835aux.c)。

### 结论

你可以把这一章当作是为阅读 `arm64` 引导代码做的准备 - 因为如果你不明白我们刚刚研究的这些概念，那你很难去学习 `arm64` 的引导代码。下一节课，我们将回到 `stext` 函数，并仔细研究它的工作原理。

##### 上一页

1.3 [内核初始化： 内核构建系统](../../../docs/lesson01/linux/build-system.md)

##### 下一页

1.5 [内核初始化： 练习](../../../docs/lesson01/exercises.md)
