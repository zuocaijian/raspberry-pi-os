## 2.1: 处理器初始化 

这节课，我们准备更深入的学习使用 ARM 处理器。处理器有一些操作系统可以用到的关键功能特性。第一个关键特性就是 "异常级别(Exception levels)"。

### 异常级别

每个支持 ARM.v8 架构的 ARM 处理器都有4种异常级别。你可以把一种异常级别(简称 `EL`)当作是处理器的一种执行模式，在这种执行模式下，只有有限的子操作和一部分寄存器是可用的。最低级别的异常是 0(EL0)。 当处理器工作在这种级别的时候，它基本只能使用通用寄存器(X0 - X30)和栈指针寄存器(SP)。EL0 还允许使用 `STR` 和 `LDR` 指令来向内存中存数据以及从内存中取数据，以及其他少量的用户程序用到的常用指令。

一个操作系统应该处理好不同的异常级别，因为系统需要利用异常级别来实现进程隔离。一个进程不应该能够获取到其他进程的数据。为了能做到这一点，操作系统总是将用户程序运行在 EL0。运行在这种异常级别下的程序只能使用它自己的虚拟内存，而不能访问可以改变虚拟内存设置的指令。所以，为了确保进程隔离，操作系统需要为每个进程准备单独的虚拟内存映射，并且在将执行权交给用户程序之前将处理器置于 EL0 状态。

操作系统通常工作下 EL1 状态下。运行在这种异常级别下的处理器可以访问能配置虚拟内存设置的寄存器，和其他一些系统寄存器。树莓派 操作系统同样也是工作在 EL1 状态下。

我们不会大量使用 EL2 和 EL3，不过我还是简单的介绍一下，这样你就能明白为什么 EL2 和 EL3是必需的。 

EL2 的使用场景是当我们用到 hypervisor(虚拟化技术、多操作系统) 的时候。在这种情况下，宿主操作系统运行在 EL2，而客户端操作系统只能使用 EL1。这可以让宿主操作系统与那些客户端操作系统隔离，就像操作系统与普通用户程序隔离那样。

EL3 用于将 ARM 从"安全模式状态(Secure World)" 切换到 "非安全模式状态(Insecure world)"。这种抽象存在可以为运行在两个不同模式状态下的软件提供全硬件层的隔离。安全模式状态下的应用完全没办法访问或者修改非安全模式状态下的另一个应用的信息(包括指令和数据)，这种限制实在硬件层上强制执行的。 

### 内核调试

下一步我想做的就是确定我们当前是用的是哪种异常等级。不过当我尝试去做的时候，我突然意识到，我们的内核目前还只能在屏幕上打印一些常量字符串，而我需要的是类似于 [printf](https://en.wikipedia.org/wiki/Printf_format_string) 这样的函数。使用 `printf` 我可以很方便的显示出不同寄存器、变量的值。这样的功能对内核开发来说是很关键的，因为这样你就不需要其他调试工具的支持了， `printf` 也就成为了你获取程序内部正在发生什么的唯一方法。

对于 RPi 操作系统来说，我不打算重复造轮子了，而是使用一个 [已有的 printf 实现](http://www.sparetimelabs.com/tinyprintf/tinyprintf.php)。这个函数几乎都是写对字符串的操作，从内核开发者的角度来看，这个函数没什么意思。我使用的这个实现很小，没有额外的依赖，这样把它整合到内核中就简单多了。我唯一要做的事就是定义一个 `putc` 函数，用它来向屏幕发送单个字符。函数 [在此](https://github.com/s-matyukevich/raspberry-pi-os/blob/master/src/lesson02/src/mini_uart.c#L59) 定义，它仅仅就是使用了早就有了的 `uart_send` 函数。同时，我们需要初始化 `printf` 这个库，以及指定 `putc` 这个函数的位置。这些都在 [一行代码](https://github.com/s-matyukevich/raspberry-pi-os/blob/master/src/lesson02/src/kernel.c#L8) 中搞定。

### 找出当前异常级别

现在，我们有了与 `printf` 等价的函数，可以完成最开始的任务了：获取操作系统被引导加载时候的异常等级。定义在 [此处](https://github.com/s-matyukevich/raspberry-pi-os/blob/master/src/lesson02/src/utils.S#L1) 的一个很短的函数可以实现这个功能，函数看起来像下面这样：

```
.globl get_el
get_el:
    mrs x0, CurrentEL
    lsr x0, x0, #2
    ret
```

这里我们用 `mrs` 指令把从 `CurrentEL` 系统寄存器读取到的值放到 `x0` 寄存器中。然后我们把这个值右移2位(这么做是因为 `CurrentEL` 寄存器的前2位被保留，值总是为0)。最后在 `x0` 寄存器中就是当前异常级别所对应的整型数了。现在，唯一剩下的事就是把它显示出来，像 [这样](https://github.com/s-matyukevich/raspberry-pi-os/blob/master/src/lesson02/src/kernel.c#L10)

```
    int el = get_el();
    printf("Exception level: %d \r\n", el);
```

如果你重复这个实验，你应该能在屏幕上看到 `Exception level: 3` 。

### 改变当前异常等级

在 ARM 架构中，如果没有已运行在更高等级的异常级别下的软件的参与，程序根本不可能提升自己的异常级别。这非常合乎道理：无论如何，任何软件都不能逃脱它被分配的异常级别，也不能访问其他程序的数据。只有当一个异常产生的时候，当前异常级别才能被改变。这种异常可能在程序执行非法指令的时候产生(比如，尝试访问一个不存在的内存地址，或者被0整除)。 应用也可以用 `svc` 指令来故意产生一个异常。硬件所产生的中断也会被当作是一种特殊的异常来处理。每当异常产生的时候，就会执行以下步骤序列(在这段表述中，我假定异常是在 EL1、 EL2、 EL3中被处理的)。

1. 当前指令地址被保存在 `ELR_ELn` 寄存器中。(寄存器被称为 `异常链接寄存器(Exception link register`))
1. 当前处理器状态被保存在 `SPSR_ELn` 寄存器 (`程序状态保存寄存器(Saved Program Status Register`))
1. 执行某个异常处理器，并执行它需要做的全部任务。
1. 异常处理器调用 `eret` 指令。这个指令从 `SPSR_ELn` 寄存器中回复处理器的状态，并从保存在 `ELR_ELn` 寄存器中的地址的地方重新开始执行。

事实上程序要更复杂一点，因为异常处理器还要保存所有通用寄存器的状态，并在处理完后恢复状态，不过我们下一节课才详细讨论这些细节。现在，我们只需要明白大概的流程，以及记住  `ELR_ELm` 和 `SPSR_ELn` 寄存器的含义。

需要知道一件重要的事情就是，异常处理器没有义务返回到异常产生的地方。 `ELR_ELm` 和 `SPSR_ELn` 两个寄存器都是可写的，只要是异常处理器想，它就能修改这两个寄存器的值。当我们在代码中试着从 EL3 切换到 EL1 的时候，我们会用到这个技术。

### 切换到 EL1

严格来说，我们的操作系统没有义务去切换到 EL1，不过使用 EL1 对我们来说是一个很自然的选择，因为这个级别正好有实现所有常用操作系统任务的权限集合。看看在运行中如何切换异常级别也是一个很有趣的练习。我们来看看 [做这事的源代码](https://github.com/s-matyukevich/raspberry-pi-os/blob/master/src/lesson02/src/boot.S#L17).

```
master:
    ldr    x0, =SCTLR_VALUE_MMU_DISABLED
    msr    sctlr_el1, x0        

    ldr    x0, =HCR_VALUE
    msr    hcr_el2, x0

    ldr    x0, =SCR_VALUE
    msr    scr_el3, x0

    ldr    x0, =SPSR_VALUE
    msr    spsr_el3, x0

    adr    x0, el1_entry        
    msr    elr_el3, x0

    eret                
```

如你所见，代码主要是对几个系统寄存器的配置(操作)组成的。我们来逐个查看这些寄存器。为了查看这些寄存器，我们需要先下载 [AArch64参考手册](https://developer.arm.com/docs/ddi0487/ca/arm-architecture-reference-manual-armv8-for-armv8-a-architecture-profile)。这篇文档包含了 `ARM.v8` 架构的特定细节。 

#### SCTLR_EL1，系统控制寄存器 (EL1), AArch64参考手册 第 2654 页

```
    ldr    x0, =SCTLR_VALUE_MMU_DISABLED
    msr    sctlr_el1, x0        
```

这里我们给 `sctlr_el1` 系统寄存器设置值。当处于 EL1 状态下操作的时候， `sctlr_el1` 负责配置处理器的不同参数。比如，它控制缓存是否启用，对我们来说最重要的一点是，是否开启 MMU (内存映射单元)。 `sctlr_el1` 对大于等于 EL1 的所有异常级别来说都是可访问的(你可以从 `_el1` 后缀推断出来)。 

`SCTLR_VALUE_MMU_DISABLED` 常量定义在 [这里](https://github.com/s-matyukevich/raspberry-pi-os/blob/master/src/lesson02/include/arm/sysregs.h#L16)。其值的每一位定义如下：

* `#define SCTLR_RESERVED                  (3 << 28) | (3 << 22) | (1 << 20) | (1 << 11)` 在 `sctlr_el1` 寄存器的描述中一些位被标记为 `RES1`。这些位是为将来使用而保留的，应该被初始化为 `1`。
* `#define SCTLR_EE_LITTLE_ENDIAN          (0 << 25)` 异常 [字节序(Endianness)](https://en.wikipedia.org/wiki/Endianness)。这个字段控制 EL1 中显式数据访问的字节序。我们将把处理器配置成只工作在 `小端法(字节序)` 模式下。
* `#define SCTLR_EOE_LITTLE_ENDIAN         (0 << 24)` 和前一个字段相似，不过这个字段控制的是 EL1 中显式数据访问的字节序，而不是 EL0。 
* `#define SCTLR_I_CACHE_DISABLED          (0 << 12)` 禁用指令缓存。为简便起见，我们将禁用全部缓存。你可以在 [这里](https://stackoverflow.com/questions/22394750/what-is-meant-by-data-cache-and-instruction-cache) 获取关于数据和指令缓存的更多相关信息。
* `#define SCTLR_D_CACHE_DISABLED          (0 << 2)` 禁用数据缓存。
* `#define SCTLR_MMU_DISABLED              (0 << 0)` 禁用 MMU(内存映射单元)。在第 6 节课我们准备使用页表和开始使用虚拟内存来工作以前，我们必须禁用 MMU。

#### HCR_EL2, 虚拟化配置寄存器 (EL2), AArch64参考手册 第 2487 页 

```
    ldr    x0, =HCR_VALUE
    msr    hcr_el2, x0
```

我们不准备实现我们自己的 [虚拟化](https://en.wikipedia.org/wiki/Hypervisor)。不过我们还是需要使用这些寄存器，因为在其它设置中，它控制 EL1 的执行状态。执行状态必须是 `AArch64` 而不是 `AArch32`。这个配置是在 [这里](https://github.com/s-matyukevich/raspberry-pi-os/blob/master/src/lesson02/include/arm/sysregs.h#L22)。 

#### SCR_EL3, 安全配置寄存器 (EL3), AArch64参考手册 第 2648 页

```
    ldr    x0, =SCR_VALUE
    msr    scr_el3, x0
```

这个寄存器负责配置安全方面的设置。例如，所有更低的级别是被执行在安全还是非安全状态下。它还控制 EL2 下的执行状态。 [在这里](https://github.com/s-matyukevich/raspberry-pi-os/blob/master/src/lesson02/include/arm/sysregs.h#L26) 我们设置 EL2 执行于 `AArch64` 状态下，以及所有更低的级别是非安全的。 

#### SPSR_EL3, 程序状态保存寄存器 (EL3), AArch64参考手册 第 389 页

```
    ldr    x0, =SPSR_VALUE
    msr    spsr_el3, x0
```

这个寄存器对你来说肯定早就很熟悉了 - 我们在讨论程序改变异常级别的时候就提到过它。 `spsr_el3` 包含了我们执行 `eret` 指令后处理器所要恢复的状态。有必要稍微解释下什么是处理器状态。处理器状态包括以下信息：

* **Condition Flags** 这些标志包含前一个执行操作的信息：结果为负(N 标志)，零(A 标志)，无符号溢出(C 标志)或者有符号溢出(V 溢出)。这些标志值可用于条件分支指令。比如， `b.eq` 指令在上一个比较操作结果为0的的时候才会跳转到给定的标签。处理器通过测试 Z 标志是否被设置为1来检测判定。

* **Interrupt disable bits** 这些位的值可用来启用/禁用不同类型的中断。

* 异常被处理完成后完全恢复处理器执行状态所需的其他信息。

通常，当 EL3 发生异常时，`spsr_el3` 会被自动保存。不过这个寄存器是可写入的，所以我们利用这个特性来手动准备处理器状态。`SPSR_VALUE` 在 [这里](https://github.com/s-matyukevich/raspberry-pi-os/blob/master/src/lesson02/include/arm/sysregs.h#L35) 被准备好，同时我们还初始化如下字段：

* `#define SPSR_MASK_ALL        (7 << 6)` 在将 EL 改成 EL1 后，所有类型的中断将会被屏蔽(换句话说就是被禁用)。
* `#define SPSR_EL1h        (5 << 0)` 在 EL1 下，我们可以使用自己的专有栈指针，也可以用 EL0 栈指针。 `EL1h` 模式表示我们使用的是专有栈指针。 

#### ELR_EL3, 异常链接寄存器 (EL3),  AArch64参考手册 第 351 页

```
    adr    x0, el1_entry        
    msr    elr_el3, x0

    eret                
```

`elr_el3` 持有在 `eret` 指令被执行后我们将要返回的地址。在这里，我们把这个地址设为 `el1_entry` 标签所在的位置。

### 结论

这点非常重要：当我们进入到 `el1_entry` 函数的时候，执行一定是已经处于 EL1 模式下。去尝试一下吧! 

##### 上一页

1.5 [内核初始化：练习](../../docs/lesson01/exercises.md)

##### 下一页

2.2 [处理器初始化： Linux](../../docs/lesson02/linux.md)
