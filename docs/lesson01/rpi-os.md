## 1.1: RPi 操作系统介绍， 以及裸板程序 "Hello, world!"

通过写一个小的，纯裸板的 "Hello, World" 应用来开启一场属于我们的操作系统开发之旅。我假定你已经阅读了 [先决条件](../Prerequisites.md) 并且已经按照要求做好了所有的准备。如果没有，请现在开始准备。

在下一步开始之前，我想先建立一个简单的命名约定。在README文件中你可以看到整个教程的划分。 每一节课都由一些我称之为 "章" 的文件组成(现在, 你正在阅读第 一 节课, 第 1.1 章)。 每一章又被分为许多 "节" 。 这种命名约定可以让我引用资料的不同部分。

另一个你需要注意的事情就是这个教程包含了大量的源代码实例。 通常我会以一个完整的代码块作为课程的开始，然后去逐行解释它。 

### 项目结构

每一节课的源代码结构相同。你可以在 [这里](https://github.com/s-matyukevich/raspberry-pi-os/tree/master/src/lesson01) 找到本节课程的源代码。 让我们来简单描述一下这个文件夹的主要组成部分：
1. **Makefile** 我们将使用 [make](http://www.math.tau.ac.il/~danha/courses/software1/make-intro.html) 工具来构建这个内核。通过配置 Makefile 文件来控制 `make` 的行为，Makefile包含了如何编译和链接源代码的指令。 
1. **build.sh 和 build.bat** 如果你使用了 Docker 那你就需要这些文件。 这样，你就不需要在你笔记本电脑上安装 make 工具和编译工具链了。
1. **src** 这个文件夹包含了所有的源代码。
1. **include** 所有的头文件都放在这个文件夹中。 

### Makefile

现在让我们来仔细看一下这个工程的 Makefile。 make 工具最简单的用途是用来自动决定哪些程序片段需要重新编译，并且发出指令去编译他们。如果你不熟悉 make 和 Makefile，我推荐你去阅读 [这篇](http://opensourceforu.com/2012/06/gnu-make-in-detail-for-beginners/) 文章。 
在 [这里](https://github.com/s-matyukevich/raspberry-pi-os/blob/master/src/lesson01/Makefile) 可以找到 Makefile 在第一节课中的使用。全部内容如下所示：
```
ARMGNU ?= aarch64-linux-gnu

COPS = -Wall -nostdlib -nostartfiles -ffreestanding -Iinclude -mgeneral-regs-only
ASMOPS = -Iinclude 

BUILD_DIR = build
SRC_DIR = src

all : kernel7.img

clean :
    rm -rf $(BUILD_DIR) *.img 

$(BUILD_DIR)/%_c.o: $(SRC_DIR)/%.c
    mkdir -p $(@D)
    $(ARMGNU)-gcc $(COPS) -MMD -c $< -o $@

$(BUILD_DIR)/%_s.o: $(SRC_DIR)/%.S
    $(ARMGNU)-gcc $(ASMOPS) -MMD -c $< -o $@

C_FILES = $(wildcard $(SRC_DIR)/*.c)
ASM_FILES = $(wildcard $(SRC_DIR)/*.S)
OBJ_FILES = $(C_FILES:$(SRC_DIR)/%.c=$(BUILD_DIR)/%_c.o)
OBJ_FILES += $(ASM_FILES:$(SRC_DIR)/%.S=$(BUILD_DIR)/%_s.o)

DEP_FILES = $(OBJ_FILES:%.o=%.d)
-include $(DEP_FILES)

kernel7.img: $(SRC_DIR)/linker.ld $(OBJ_FILES)
    $(ARMGNU)-ld -T $(SRC_DIR)/linker.ld -o $(BUILD_DIR)/kernel7.elf  $(OBJ_FILES)
    $(ARMGNU)-objcopy $(BUILD_DIR)/kernel7.elf -O binary kernel7.img
``` 
现在，让我们来详细检查一下这个文件：
```
ARMGNU ?= aarch64-linux-gnu
```
这个 Makefile 以一个变量的定义开始。 `ARMGNU` 是一个交叉编译器前缀。我们需要一个 [交叉编译器](https://en.wikipedia.org/wiki/Cross_compiler)，因为我们正在一台 `x86` 电脑上编译运行在 `arm64` 架构机器上的源代码。 所以我们不是用 `gcc`，而是使用 `aarch64-linux-gnu-gcc`。 

```
COPS = -Wall -nostdlib -nostartfiles -ffreestanding -Iinclude -mgeneral-regs-only
ASMOPS = -Iinclude 
```

`COPS` 和 `ASMOPS` 分别是当编译C和汇编代码时我们传递给编译器的可选项。 对这些可选项的简单解释如下：

* **-Wall** 显示所有的警告。
* **-nostdlib** 不使用标准C库。大部分标准C库的调用其实是在与操作系统交互。 我们正在写一个裸板程序，没有任何底层操作系统，所以标准C库对我们来说是不可用的。
* **-nostartfiles** 不使用标准启动文件。 启动文件负责初设置初始栈指针，初始化静态数据，并且跳转到主函数入口处。所有的这些工作将由我们自己来完成。
* **-ffreestanding** 独立环境是指标准C库可能不存在的这样一种环境，程序也不一定非得是由 main 函数来启动。 `-ffreestanding` 这个可选项指示编译器不要假定标准函数具有他们通常的定义。
* **-Iinclude** 指示编译器在 `include` 文件夹下查找头文件。
* **-mgeneral-regs-only**. 只使用通用寄存器。 ARM 同时还有 [NEON](https://developer.arm.com/technologies/neon) 寄存器。 我们不想让编译器使用 NEON 寄存器，因为这些寄存器将会增加额外的复杂度 (比如，我们在切换上下文的时候需要保存这些寄存器的状态)。

```
BUILD_DIR = build
SRC_DIR = src
```

`SRC_DIR` 和 `BUILD_DIR` 分别是包含源代码和编译后目标(或中间)文件的目录。

```
all : kernel7.img

clean :
    rm -rf $(BUILD_DIR) *.img 
```

接下来，我们定义 make 的目标(任务)。前两个目标非常简单： `all` 是一个默认的目标，当你键入 `make` 命令并且不带任何参数时，这个目标将被执行。 (`make` 命令总是把第一个目标当成是默认目标)。这个目标只是将所有的工作分别重定向到不同的目标，`kernel7.img`。 `clean` 这个目标负责删除所有编译后的目标(或中间)文件和内核镜像文件。

```
$(BUILD_DIR)/%_c.o: $(SRC_DIR)/%.c
    mkdir -p $(@D)
    $(ARMGNU)-gcc $(COPS) -MMD -c $< -o $@

$(BUILD_DIR)/%_s.o: $(SRC_DIR)/%.S
    $(ARMGNU)-gcc $(ASMOPS) -MMD -c $< -o $@
```

紧接着的两个目标负责编译C和汇编文件。 例如，假设在 `src` 目录下我们有 `foo.c` 和 `foo.S` 两个文件，这两个目标分别将他们编译成 `build/foo_c.o` 和 `build/foo_s.o`。 `$<` 和 `$@` 将在运行时被替换成输入和输出文件名 (`foo.c` 和 `foo_c.o`)。 在编译 C 文件之前，我们还将创建 `build` 目录以防这个目录不存在。

```
C_FILES = $(wildcard $(SRC_DIR)/*.c)
ASM_FILES = $(wildcard $(SRC_DIR)/*.S)
OBJ_FILES = $(C_FILES:$(SRC_DIR)/%.c=$(BUILD_DIR)/%_c.o)
OBJ_FILES += $(ASM_FILES:$(SRC_DIR)/%.S=$(BUILD_DIR)/%_s.o)
```

此处我们将构建一系列由C和汇编等相关源文件所生成的目标文件 (`OBJ_FILES`)。

```
DEP_FILES = $(OBJ_FILES:%.o=%.d)
-include $(DEP_FILES)
```

接下来的两行代码有点棘手。如果你仔细看一下我们是如何定义 C 和汇编源文件的编译目标的话，你就会注意到我们使用了 `-MMD` 这个参数。 这个参数指示 `gcc` 编译器为每一个生成的对象文件创建依赖文件。 依赖文件定义了特定源文件的所有依赖项。这些依赖项通常包含一系列的头文件。我们需要把所有生成的依赖文件包含进来，以便于在头文件发生更改的时候知道哪些需要重新编译。 

```
$(ARMGNU)-ld -T $(SRC_DIR)/linker.ld -o kernel7.elf  $(OBJ_FILES)
``` 

我们使用 `OBJ_FILES` 数组来编译 `kernel7.elf` 文件。我们使用 `src/linker.ld` 这个链接脚本来定义生成的可执行镜像文件的基本布局。 (我们将在下一小节讨论这个链接脚本文件)。

```
$(ARMGNU)-objcopy kernel7.elf -O binary kernel7.img
```

`kernel7.elf` 是 [ELF](https://en.wikipedia.org/wiki/Executable_and_Linkable_Format) 格式的(可执行可链接格式)。问题是，ELF 文件是被设计用来由操作系统执行的文件格式。为了编写一个裸板程序，我们需要从这个 ELF 文件中提取所有的可执行机器代码段和数据段，然后将它们写入 `kernel7.img` 这个镜像中。 

### 链接脚本

链接脚本的一个主要目的是描述输入文件 (`_c.o` 和 `_s.o`) 到输出文件 (`.elf`)之间的映射关系。 在 [这里](https://sourceware.org/binutils/docs/ld/Scripts.html#Scripts) 可以找到更多关于链接脚本的知识。现在让我们来看一下 RPi 操作系统的链接脚本：

```
SECTIONS
{
    .text.boot : { *(.text.boot) }
    .text :  { *(.text) }
    .rodata : { *(.rodata) }
    .data : { *(.data) }
    . = ALIGN(0x8);
    bss_begin = .;
    .bss : { *(.bss*) } 
    bss_end = .;
}
``` 

启动后，树莓派会加载 `kernel7.img` 这个镜像文件到内存中，并且从文件开头的地方开始执行。 这就是为什么 `.text.boot` 段(已编译程序的机器代码)必须放在最前面的原因；我们将把操作系统的启动代码放到这个 段 中。 `.text`， `.rodata` 和 `.data` 这几个段包含了编译好的内核指令、只读数据，和已初始化的全局和静态变量 - 并没有关于这些概念的特别补充说明(可以参考《深入理解计算机系统》这本书的第 7 章了解更多信息)。 `.bss` 段包含了未初始化的全局和静态变量。 通过将这些数据放在单独的段中这种方式，使得编译器可以在生成 ELF 二进制文件的时候节省一些空间 –– 只在 ELF 头中存储这些段的大小，而不存储这些段本身。在把镜像文件加载到内存中以后，我们必须将 `.bss` 段中的数据初始化为0；这也是为什么我们需要记录每个段的开始和结束位置(从符号 `bss_begin` 到符号 `bss_end` ) ，以及对齐每个段(字节对齐)，这样每个指令的起始地址都是8的倍数。如果不对齐每个段，那么将很难在 `bss` 段的开始处使用 `str` 指令来进行初始化数据，因为 `str` 指令只能用于8字节对齐的地址。

### 引导内核

现在是时候来看一下 [boot.S](https://github.com/s-matyukevich/raspberry-pi-os/blob/master/src/lesson01/src/boot.S) 这个文件了。这个文件包含了内核启动代码：

```
#include "mm.h"

.section ".text.boot"

.globl _start
_start:
    mrs    x0, mpidr_el1        
    and    x0, x0,#0xFF        // Check processor id
    cbz    x0, master        // Hang for all non-primary CPU
    b    proc_hang

proc_hang: 
    b proc_hang

master:
    adr    x0, bss_begin
    adr    x1, bss_end
    sub    x1, x1, x0
    bl     memzero

    mov    sp, #LOW_MEMORY
    bl    kernel_main
```
让我们来详细的查看一下一个文件：
```
.section ".text.boot"
```
首先，我们应该知道 `boot.S` 文件中定义的所有内容都应该放在 `.text.boot` 段中。在前面的内容中，我们可以看到这个部分被链接脚本文件放在内核镜像文件的最前面。所以，当内核启动时，从执行 `start` 这个函数开始：
```
.globl _start
_start:
    mrs    x0, mpidr_el1        
    and    x0, x0,#0xFF        // Check processor id
    cbz    x0, master        // Hang for all non-primary CPU
    b    proc_hang
```

这个函数要做的第一件事就是检查处理器的ID。树莓派3代有4个处理器核心，在设备通电以后，每个处理器核心开始执行相同的代码。然而，我们并不需要4个核心同时工作；我们只需要第一个核心来工作就行了，让其他的核心陷入死循环中。这就是 `_start` 这个函数的责任所在。从 [mpidr_el1](http://infocenter.arm.com/help/index.jsp?topic=/com.arm.doc.ddi0500g/BABHBJCI.html) 这个系统寄存器我们可以获取到处理器核心的ID。如果当前的处理器核心ID为0，那就切换到 `master` 这个函数的执行处开始执行：

```
master:
    adr    x0, bss_begin
    adr    x1, bss_end
    sub    x1, x1, x0
    bl     memzero
```

在这个函数中，我们通过调用 `memzero` 来清除 `.bss` 段。我们将在后面定义这个函数。在 ARMv8 架构中，按照惯例，可以通过 x0-x6 这几个寄存器来向被调用函数传递前7个参数。 `memzero` 这个函数只接受两个参数：起始地址 (`bss_begin`)以及需要被清除的段的大小(`bss_end - bss_begin`)。

```
    mov    sp, #LOW_MEMORY
    bl    kernel_main
```

在清除 `.bss` 这个段后，我们初始化栈指针跳转到 `kernel_main` 这个函数的执行处开始执行。 树莓派从地址 0 开始加载内核；这就是为什么栈指针可以被设置到任何足够高的位置，这样即使当内存镜像文件变得十分大的时候，栈 区内容也不会被擦除覆写。 `LOW_MEMORY` 被定义在 [mm.h](https://github.com/s-matyukevich/raspberry-pi-os/blob/master/src/lesson01/include/mm.h) 这个头文件中，大小等于 4MB。 我们内核的栈区不会变的很大，而且镜像文件本身也很小，所以 `4MB` 对我们来说已经足够大了。 

对于哪些不熟悉 ARM 汇编器语法的人，让我们来快速总结一下我们使用过的指令：

* [**mrs**](http://www.keil.com/support/man/docs/armasm/armasm_dom1361289881374.htm) 从系统寄存器中取出值加载到一个通用寄存器(x0–x30)
* [**and**](http://www.keil.com/support/man/docs/armasm/armasm_dom1361289863017.htm) 执行逻辑(位) 与 操作。我们使用这个指令来删除我们从 `mpidr_el1` 这个系统寄存器中获取到的值的后两位。
* [**cbz**](http://www.keil.com/support/man/docs/armasm/armasm_dom1361289867296.htm) 将先前执行的操作结果与 0 进行比较，如果结果为真，那就跳转 (或者 ARM 术语中的 `branch` )到指令参数指定的 Label 处。
* [**b**](http://www.keil.com/support/man/docs/armasm/armasm_dom1361289863797.htm) 无条件跳转到标号 Label 处执行。
* [**adr**](http://www.keil.com/support/man/docs/armasm/armasm_dom1361289862147.htm) 将标签的相对地址加载到目标寄存器中。 此处，我们希望将指针指向 `.bss` 区的开始和结束的地方。
* [**sub**](http://www.keil.com/support/man/docs/armasm/armasm_dom1361289908389.htm) 将两个寄存器中的值相减。
* [**bl**](http://www.keil.com/support/man/docs/armasm/armasm_dom1361289865686.htm) "带链接的分支"：无条件执行分支语句，同时将当前的 PC 值保存到 x30 (链接寄存器)。当子程序结束的时候，使用 `ret` 这个指令来返回到跳转指令之后的那个指令处执行。
* [**mov**](http://www.keil.com/support/man/docs/armasm/armasm_dom1361289878994.htm) 将一个寄存器的值移动到另一个寄存器，或者将一个常量值移动到寄存器。

[这里](http://infocenter.arm.com/help/index.jsp?topic=/com.arm.doc.den0024a/index.html) 是一份 ARMv8-A 开发者指南。 如果你不熟悉 ARM 指令集体系架构，这将是一份很不错的资料。 [这一页](http://infocenter.arm.com/help/index.jsp?topic=/com.arm.doc.den0024a/ch09s01s01.html) 明确的概述了在 ABI(应用程序二进制接口)中寄存器的使用约定。

### 关于 `kernel_main` 这个函数

我们已经看到，引导代码最终将控制权交给了 `kernel_main` 这个函数。让我们来看一下这个函数：

```
#include "mini_uart.h"

void kernel_main(void)
{
    uart_init();
    uart_send_string("Hello, world!\r\n");

    while (1) {
        uart_send(uart_recv());
    }
}

```

这是内核中最为简单的函数之一。它使用 `Mini UART` 来向屏幕打印以及读取用户输入。这个内核只是打印 `Hello, world!`，然后就进入一个无限循环当中，读取用户输入的字符并回显在屏幕中。

### 关于树莓派这个设备 

现在我们来深入研究一下树莓派。在开始之前，我推荐你去下载一份 [BCM2835 ARM 外围设备手册](https://www.raspberrypi.org/documentation/hardware/raspberrypi/bcm2835/BCM2835-ARM-Peripherals.pdf). BCM2835 是树莓派 A 型, B 型, 以及 B+ 型号使用的开发板。 树莓派 3 代使用的是 BCM2837，但是其底层架构与 BCM2835 相同。

在我们开始处理这些实现细节之前，我想分享一些关于如何使用内存映像设备的基本概念。 BCM2835 是一款简单 [SOC (System on a chip (片上系统))](https://en.wikipedia.org/wiki/System_on_a_chip) 开发板. 在这块板子上，可以通过操作内存映射寄存器来访问外围设备。 树莓派 3 代的外围设备内存地址开始于 `0x3F000000`。为了使用或者配置某个特定的外围设备，你需要向其中一个外围设备的寄存器中写入数据。一个外围设备的寄存器就是一块32位的内存区域。 关于每个寄存器中的每个位所代表的含义可以在 `BCM2835 ARM 外围设备手册` 中查到。

从 `kernel_main` 这个函数中，你能猜到我们准备使用一个 `Mini UART` 外围设备来工作。 UART 是指 [Universal asynchronous receiver-transmitter(通用串口)](https://en.wikipedia.org/wiki/Universal_asynchronous_receiver-transmitter)。这个设备能够将存储在它的一个内存映射寄存器中的值转换成一个高电平和低电平组成的序列。这个序列通过 `TTL-to-serial cable(串口转接线)` 传递给你的计算机，然后由你的终端模拟器来解析这个序列。我们将使用 `Mini UART` 帮助我们完成与树莓派的交互操作。 如果你想看 Mini UART 寄存器的使用规范，请翻阅 `BCM2835 ARM 外围设备手册` 的第八页。

另一个你需要熟悉的设备是 GPIO [General-purpose input/output(通用输入输出)](https://en.wikipedia.org/wiki/General-purpose_input/output). GPIOs 由 `GPIO 引脚` 来负责控制。你应该能够从下图中很容易地识别出它们：

![Raspberry Pi GPIO pins](../../images/gpio-pins.jpg)

通过不同的 GPIO 引脚可以用来配置 GPIO 的行为。例如，为了能够使用 `Mini UART`，我们需要激活14和15两个引脚，并设置好才能使用这个外围设备。下图说明了 GPIO 引脚的编号：

![Raspberry Pi GPIO pin numbers](../../images/gpio-numbers.png)

### Mini UART 初始化

现在我们来看一下 Mini UART 是如何初始化的。相关代码在 [mini_uart.c](https://github.com/s-matyukevich/raspberry-pi-os/blob/master/src/lesson01/src/mini_uart.c)文件中：

```
void uart_init ( void )
{
    unsigned int selector;

    selector = get32(GPFSEL1);
    selector &= ~(7<<12);                   // clean gpio14
    selector |= 2<<12;                      // set alt5 for gpio14
    selector &= ~(7<<15);                   // clean gpio15
    selector |= 2<<15;                      // set alt5 for gpio 15
    put32(GPFSEL1,selector);

    put32(GPPUD,0);
    delay(150);
    put32(GPPUDCLK0,(1<<14)|(1<<15));
    delay(150);
    put32(GPPUDCLK0,0);

    put32(AUX_ENABLES,1);                   //Enable mini uart (this also enables access to it registers)
    put32(AUX_MU_CNTL_REG,0);               //Disable auto flow control and disable receiver and transmitter (for now)
    put32(AUX_MU_IER_REG,0);                //Disable receive and transmit interrupts
    put32(AUX_MU_LCR_REG,3);                //Enable 8 bit mode
    put32(AUX_MU_MCR_REG,0);                //Set RTS line to be always high
    put32(AUX_MU_BAUD_REG,270);             //Set baud rate to 115200
    put32(AUX_MU_IIR_REG,6);                //Clear FIFO

    put32(AUX_MU_CNTL_REG,3);               //Finaly, enable transmitter and receiver
}
``` 

在这里，我们用到了 `put32` 和 `get32` 这两个函数。函数非常简单；它可以让我们往32位寄存器中读写数据。你可以在 [utils.S](https://github.com/s-matyukevich/raspberry-pi-os/blob/master/src/lesson01/src/utils.S)这个文件中看一下它们是如何实现的。 `uart_init` 是本节课最复杂也是最重要的函数之一，在接下来的三个小节中我们还将继续研究它。

#### GPIO 可选功能 

首先，我们需要激活 GPIO 引脚。大多数引脚可以使用不同的设备，所以在使用某个特定的引脚之前，我们需要选择这个引脚的 `alternative function`(可选功能)。 `alternative function` 是一个可以为每个引脚设置的从 0 到 5 的数字，用来配置哪个设备连接到该引脚。你可以从下图中看到全部可用的 GPIO 的可选功能列表(图片来自于 `BCM2835 ARM 外围设备手册` 第102页)：

![Raspberry Pi GPIO alternative functions](../../images/alt.png?raw=true)

其中你可以看到14和15两个引脚分别具有 TXD1(发送数据) 和 RXD1(接受数据) 的可选功能。 这意味这，如果我们使用14和15号引脚的可选功能 5 ，那么它们将分别被用作 Mini UART 发送数据引脚和 Mini UART 接受数据引脚。 `GPFSEL1` 这个寄存器用来控制10-19号引脚的 可选功能 。这些寄存器中全部 位 对应含义如下表所示(`BCM2835 ARM 外围设备操作手册` 第92页)：

![Raspberry Pi GPIO function selector](../../images/gpfsel1.png?raw=true)

现在你应该能够理解下面的这几行代码了，它们是用来配置14和15号 GPIO 的引脚以便于使用 Mini UART：

```
    unsigned int selector;

    selector = get32(GPFSEL1);
    selector &= ~(7<<12);                   // clean gpio14
    selector |= 2<<12;                      // set alt5 for gpio14
    selector &= ~(7<<15);                   // clean gpio15
    selector |= 2<<15;                      // set alt5 for gpio 15
    put32(GPFSEL1,selector);
```

#### GPIO 高低电平 

当你使用树莓派 GPIO 的时候，你会经常碰到高、低电平这样的术语。关于这个概念的详细解析可以看 [这篇](https://grantwinney.com/using-pullup-and-pulldown-resistors-on-the-raspberry-pi/) 文章。 对于哪些懒得看全文的人，我在这里简单解释一下高、低电平这个概念。

如果你使用某个特定引脚作为输入，但该引脚不与任何东西相连，那么你就没办法区分这个引脚的值到底是 0 还是 1。事实上，设备将会是一个随机值。高低电平机制能让你避免这个问题。如果你将引脚设为高电平而不连接任何东西，那么值就会一直为 `1` (对于低电平来说，这个值将总是为 0)。像我们这种情况，我们既不需要高电平也不需要低电平，因为14和15号引脚一直都有连接东西。即使重启，引脚的状态也会保存，所以在我们使用引脚之前，我们总是应该初始化它们的状态。有三种可用的状态：高电平、低电平，以及0电平(移除当前高、低电平状态)，我们需要第三种。

切换引脚状态并不是一个简单的额过程，因为他需要用到电路上的物理开关。这个过程涉及到 `GPPUD` 和 `GPPUDCLK` 两个寄存器，相关描述可参考 `BCM2835 ARM 外围设备手册` 第101页。摘录如下：

```
The GPIO Pull-up/down Clock Registers control the actuation of internal pull-downs on
the respective GPIO pins. These registers must be used in conjunction with the GPPUD
register to effect GPIO Pull-up/down changes. The following sequence of events is
required:
1. Write to GPPUD to set the required control signal (i.e. Pull-up or Pull-Down or neither
to remove the current Pull-up/down)
2. Wait 150 cycles – this provides the required set-up time for the control signal
3. Write to GPPUDCLK0/1 to clock the control signal into the GPIO pads you wish to
modify – NOTE only the pads which receive a clock will be modified, all others will
retain their previous state.
4. Wait 150 cycles – this provides the required hold time for the control signal
5. Write to GPPUD to remove the control signal
6. Write to GPPUDCLK0/1 to remove the clock
``` 

这个过程描述了我们如何移除一个引脚的高、低电平状态，也就是我们在下面代码中对14和15号引脚所做的事情：

```
    put32(GPPUD,0);
    delay(150);
    put32(GPPUDCLK0,(1<<14)|(1<<15));
    delay(150);
    put32(GPPUDCLK0,0);
```

#### 初始化 Mini UART

现在我们的 Mini UART 已经连接上了 GPIO 引脚，引脚也已经配置好了。剩下的 `uart_init` 函数用来决定 Mini UART 的初始化。 

```
    put32(AUX_ENABLES,1);                   //Enable mini uart (this also enables access to its registers)
    put32(AUX_MU_CNTL_REG,0);               //Disable auto flow control and disable receiver and transmitter (for now)
    put32(AUX_MU_IER_REG,0);                //Disable receive and transmit interrupts
    put32(AUX_MU_LCR_REG,3);                //Enable 8 bit mode
    put32(AUX_MU_MCR_REG,0);                //Set RTS line to be always high
    put32(AUX_MU_BAUD_REG,270);             //Set baud rate to 115200
    put32(AUX_MU_IIR_REG,6);                //Clear FIFO

    put32(AUX_MU_CNTL_REG,3);               //Finaly, enable transmitter and receiver
```
我们来逐行检查这个代码片段。 

```
    put32(AUX_ENABLES,1);                   //Enable mini uart (this also enables access to its registers)
```
这一行设置 Mini UART 为可用。我们必须在开始出这么做，因为这也同样允许我们访问其它所有的 Mini UART 寄存器。

```
    put32(AUX_MU_CNTL_REG,0);               //Disable auto flow control and disable receiver and transmitter (for now)
```
此处，在配置完成前，我们先禁用数据发送器和数据接收器。我们还永远的禁用了自动流程控制，因为这需要我们用到额外的 GPIO 引脚，这是TTL-to-serial 转接线所不支持的功能。关于自动流程控制的更多细节，你可以参考 [这篇](http://www.deater.net/weave/vmwprod/hardware/pi-rts/) 文章。

```
    put32(AUX_MU_IER_REG,0);                //Disable receive and transmit interrupts
```
可以配置 Mini UART 让它每次在有新的数据可用的时候生成一个处理器中断。我们将在第三节课中讲解使用中断，所以现在我们只是简单的禁用这个功能。

```
    put32(AUX_MU_LCR_REG,3);                //Enable 8 bit mode
```
Mini UART 能狗支持7位或8位的操作。这是因为ASCII字符标准集是7位，扩展集为8位。我们准备使用8位模式。 请注意，在 `BCM2835 ARM 外围设备手册` 中关于 `AUX_MU_LCR_REG` 这个寄存器的描述有一处错误。全部错误列表在 [这里](https://elinux.org/BCM2835_datasheet_errata)

```
    put32(AUX_MU_MCR_REG,0);                //Set RTS line to be always high
```
RTS线用于流控制，我们用不到。把它设为总是高电平。
```
    put32(AUX_MU_BAUD_REG,270);             //Set baud rate to 115200
```
波特率是信息在通信信道中传输的速率。 “115200 baud” 意味着串口每秒最大能传输115200位。 树莓派的 mini UART 的波特率应该与你终端模拟器使用的波特率一致。 
Mini UART 根据下面这个方程来计算波特率：
```
baudrate = system_clock_freq / (8 * ( baudrate_reg + 1 )) 
```
因为 `system_clock_freq` 是 250 MHz，所以我们轻易就能计算出 `baudrate_reg` 的值是270。

``` 
    put32(AUX_MU_CNTL_REG,3);               //Finaly, enable transmitter and receiver
```
这行代码执行完以后，Mini UART 就已经准备好开始工作了!

### 使用 Mini UART 发送数据

Mini UART 准备好之后，我们可以尝试使用它来发送和接受一些数据。 我们可以使用下面这两个函数来做到这一点：

```
void uart_send ( char c )
{
    while(1) {
        if(get32(AUX_MU_LSR_REG)&0x20) 
            break;
    }
    put32(AUX_MU_IO_REG,c);
}

char uart_recv ( void )
{
    while(1) {
        if(get32(AUX_MU_LSR_REG)&0x01) 
            break;
    }
    return(get32(AUX_MU_IO_REG)&0xFF);
}
```

这两个函数都使用一个无限循环作为开始，其目的是检查设备是否准备发送或者接收数据。我们正在使用 `AUX_MU_LSR_REG` 这个寄存器来完成这个功能。 第 0 为，如果被设置为1，表示数据准备好了；以为着我们可以从 UART 中读取到它。第 5 位，如果设置为1，表示发射器是空的，这意味着我们可以往 UART 写入数据。
接下来，我们使用 `AUX_MU_IO_REG` 寄存器来传送或者接受字符值。

我们也可以使用下面这个简单的函数来发送字符串：

```
void uart_send_string(char* str)
{
    for (int i = 0; str[i] != '\0'; i ++) {
        uart_send((char)str[i]);
    }
}
```
这个函数只是重复的对一个字符串进行迭代操作，将它们一个接一个的发送出去。 

### 树莓派配置

树莓派启动流程如下(精简后的)：

1. 设备通电。
1. 启动GPU，并从引导分区读取 `config.txt` 文件。这个文件包含了一些 GPU 用来进一步调整启动流程的配置参数。
1. `kernel7.img` 被加载到内存中并且执行。

为了能够运行我们的简易操作系统，`config.txt` 这个配置文件应该这样写：

```
arm_control=0x200
kernel_old=1
disable_commandline_tags=1
```
* `arm_control=0x200` 指定处理器以64位模式启动。 
* `kernel_old=1` 指定在地址 0 处加载内核镜像。
* `disable_commandline_tags` 指示 GPU 不传递任何命令行参数给引导镜像。


### 测试这个内核

现在我们解释完了所有的源代码，是时候来运行一下它了。为了构建和测试内核，你需要做如下操作：

1. 从 [src/lesson01](https://github.com/s-matyukevich/raspberry-pi-os/tree/master/src/lesson01) 文件目录下执行 `./build.sh` 或者 `./build.bat` 来构建这个内核。 
1. 拷贝生成好的 `kernel7.img` 文件到你的树莓派SD卡 `boot` 分区中。确保启动分区中其他所有文件都没有动过(更多信息可以看 [这个](https://github.com/s-matyukevich/raspberry-pi-os/issues/43) 问题)
1. 参考上一小节所描述的内容修改 `config.txt` 这个文件。
1. 如 [提前声明](../Prerequisites.md) 中所描述的那样连接 TTL 转接线。
1. 给你的树莓派通电。
1. 打开你的终端模拟器。你应该能在你的终端模拟器上看到 `Hello, world!` 字样信息。

##### 上一页

[提前申明](../../docs/Prerequisites.md)

##### 下一页

1.2 [内核初始化： Linux 项目结构](../../docs/lesson01/linux/project-structure.md)
