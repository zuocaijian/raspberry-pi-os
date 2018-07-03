## 2.2: 处理器初始化 (Linux)

我们在 [stext](https://github.com/torvalds/linux/blob/v4.14/arch/arm64/kernel/head.S#L116) 函数处停止了对 Linux 内核的探索，也就是进入 `arm64` 架构入口点的地方。这一次，我们将更深入一点，找到一些我们在上一节课中已经实现了的相似的代码。 

你可能会觉得这一章有点无聊，因为这一章主要就是讨论不同的 arm 系统寄存器以及它们在 Linux 内核中是如何被使用的。不过考虑到以下原因，我仍然觉得这些内容是非常重要的：

1. 了解硬件为软件提供了哪些接口是十分有必要的。很多情况下，只有当你知道这些接口你才能明白特定的内核的特性是如何实现的，以及软硬件是如何协作来实现这个特性。
1. 系统寄存器的不同选线通常与启用/禁用不同的硬件特性有关。如果你了解一个 ARM 处理器有哪些不同的系统寄存器，那你就能知道这个处理器支持什么功能了。

好了，让我们来继续研究 `stext` 函数。

```
ENTRY(stext)
    bl    preserve_boot_args
    bl    el2_setup            // Drop to EL1, w0=cpu_boot_mode
    adrp    x23, __PHYS_OFFSET
    and    x23, x23, MIN_KIMG_ALIGN - 1    // KASLR offset, defaults to 0
    bl    set_cpu_boot_mode_flag
    bl    __create_page_tables
    /*
     * The following calls CPU setup code, see arch/arm64/mm/proc.S for
     * details.
     * On return, the CPU will be ready for the MMU to be turned on and
     * the TCR will have been set.
     */
    bl    __cpu_setup            // initialise processor
    b    __primary_switch
ENDPROC(stext)
``` 

### 保护引导启动参数(preserve_boot_args)

[preserve_boot_args](https://github.com/torvalds/linux/blob/v4.14/arch/arm64/kernel/head.S#L136) 函数负责保存由引导启动加载器传递给内核的那些参数。 

```
preserve_boot_args:
    mov    x21, x0                // x21=FDT

    adr_l    x0, boot_args            // record the contents of
    stp    x21, x1, [x0]            // x0 .. x3 at kernel entry
    stp    x2, x3, [x0, #16]

    dmb    sy                // needed before dc ivac with
                        // MMU off

    mov    x1, #0x20            // 4 x 8 bytes
    b    __inval_dcache_area        // tail call
ENDPROC(preserve_boot_args)
```

根据 [内核引导启动协议(kernel boot protocol)](https://github.com/torvalds/linux/blob/v4.14/Documentation/arm64/booting.txt#L150)，存放在 `x0 - x3`. `x0` 寄存器中传递给内核的参数，包含了 RAM 体系中的设备树(`.dtb`)的物理地址。 `x1 - x3` 被保留供以后使用。这个函数把 `x0 - x3` 寄存器中的内容拷贝到 [boot_args](https://github.com/torvalds/linux/blob/v4.14/arch/arm64/kernel/setup.c#L93) 数组，然后使数据缓存中响应的缓存行失效 [invalidate](https://developer.arm.com/products/architecture/a-profile/docs/den0024/latest/caches/cache-maintenance)。多核理器系统中的缓存维护本身就是一个非常大的课题，现在我们不准备聊这些。那些对这个主题感性的人，我可以推荐你们去读一读 [缓存(Caches)](https://developer.arm.com/products/architecture/a-profile/docs/den0024/latest/caches) 以及 `ARM 编程指南(ARM Programmer’s Guide)` 中 [多核处理器(Multi-core processors)](https://developer.arm.com/products/architecture/a-profile/docs/den0024/latest/multi-core-processors) 这一章。

### el2 设置(el2_setup)

根据 [内核引导启动协议(kernel boot protocol)](https://github.com/torvalds/linux/blob/v4.14/Documentation/arm64/booting.txt#L159)，内核既可以在 EL1 下被启动，也可以在 EL2 下被启动。在第二种情况下，内核可以访问虚拟化扩展，并且充当一个宿主操作系统。如果我们有幸在 EL2 下启动内核，那么 [el2_setup](https://github.com/torvalds/linux/blob/v4.14/arch/arm64/kernel/head.S#L386) 函数将被调用。这个函数负责配置不同的参数，只能在 EL2 下访问，并降级到 EL1。现在我把这个函数分为好几小的部分，然后逐个解释它们。

```
    msr    SPsel, #1            // We want to use SP_EL{1,2}
``` 

在 EL1 和 EL2 下都能使用专有栈指针。另一个可选项是重用 EL0 下的栈指针。

```
    mrs    x0, CurrentEL
    cmp    x0, #CurrentEL_EL2
    b.eq    1f
```

只有在当前 EL 为 EL2 的时候才跳转到标签 `1` 这个地方，否则我们不能对 EL2 进行设置，且这个函数也没剩下多少要做的了。

```
    mrs    x0, sctlr_el1
CPU_BE(    orr    x0, x0, #(3 << 24)    )    // Set the EE and E0E bits for EL1
CPU_LE(    bic    x0, x0, #(3 << 24)    )    // Clear the EE and E0E bits for EL1
    msr    sctlr_el1, x0
    mov    w0, #BOOT_CPU_MODE_EL1        // This cpu booted in EL1
    isb
    ret
```

如果是在 EL1 下执行的时候发生的话， `sctlr_el1` 寄存器被更新，这样 CPU 要么工作在小端模式下，要么工作在大端模式下，这取决于 [CPU_BIG_ENDIAN](https://github.com/torvalds/linux/blob/v4.14/arch/arm64/Kconfig#L612) 配置中设置的值。然后我们从 `el2_setup` 函数中退出，并且返回常量 [BOOT_CPU_MODE_EL1](https://github.com/torvalds/linux/blob/v4.14/arch/arm64/include/asm/virt.h#L55)。根据 [ARM64 函数调用约定](http://infocenter.arm.com/help/topic/com.arm.doc.ihi0055b/IHI0055B_aapcs64.pdf) 返回值应该放在 `x0` 寄存器(或者像我们这种情况放在 `w0` 寄存器中。你可以把 `w0` 寄存器当作 `x0` 寄存器的前32位)

```
1:    mrs    x0, sctlr_el2
CPU_BE(    orr    x0, x0, #(1 << 25)    )    // Set the EE bit for EL2
CPU_LE(    bic    x0, x0, #(1 << 25)    )    // Clear the EE bit for EL2
    msr    sctlr_el2, x0
```

如果我们是在 EL2 下引导启动，我们也会对 EL2 做同样的事(注意，这时候我们是用 `sctlr_el2` 寄存器而不是 `sctlr_el1` 寄存器)

```
#ifdef CONFIG_ARM64_VHE
    /*
     * Check for VHE being present. For the rest of the EL2 setup,
     * x2 being non-zero indicates that we do have VHE, and that the
     * kernel is intended to run at EL2.
     */
    mrs    x2, id_aa64mmfr1_el1
    ubfx    x2, x2, #8, #4
#else
    mov    x2, xzr
#endif
```

如果通过配置 [ARM64_VHE](https://github.com/torvalds/linux/blob/v4.14/arch/arm64/Kconfig#L926) 变量启用了 [Virtualization Host Extensions (VHE)](https://developer.arm.com/products/architecture/a-profile/docs/100942/latest/aarch64-virtualization)，并且主机支持 `VHE`，`x2` 将被更新为非0值。`x2` 将在同样的函数中被用来检查 `VHE` 是否被启用。

```
    mov    x0, #HCR_RW            // 64-bit EL1
    cbz    x2, set_hcr
    orr    x0, x0, #HCR_TGE        // Enable Host Extensions
    orr    x0, x0, #HCR_E2H
set_hcr:
    msr    hcr_el2, x0
    isb
```

这里我们设置了 `hcr_el2` 寄存器。在 RPi 操作系统中我们使用同一个寄存器来为 EL1 设置 64 位扩展模式。这就是上述代码示例中第一行所做的事。同样的，如果 `x2 != 0` 意味着 VHE 是可用的，内核已被配置好使用 VHE，`hcr_el2` 同样也被用于启用 VHE。

```
    /*
     * Allow Non-secure EL1 and EL0 to access physical timer and counter.
     * This is not necessary for VHE, since the host kernel runs in EL2,
     * and EL0 accesses are configured in the later stage of boot process.
     * Note that when HCR_EL2.E2H == 1, CNTHCTL_EL2 has the same bit layout
     * as CNTKCTL_EL1, and CNTKCTL_EL1 accessing instructions are redefined
     * to access CNTHCTL_EL2. This allows the kernel designed to run at EL1
     * to transparently mess with the EL0 bits via CNTKCTL_EL1 access in
     * EL2.
     */
    cbnz    x2, 1f
    mrs    x0, cnthctl_el2
    orr    x0, x0, #3            // Enable EL1 physical timers
    msr    cnthctl_el2, x0
1:
    msr    cntvoff_el2, xzr        // Clear virtual offset

```

上述代码中的注释就已经做了很好的说明，我没有什么要补充说明的。

```
#ifdef CONFIG_ARM_GIC_V3
    /* GICv3 system register access */
    mrs    x0, id_aa64pfr0_el1
    ubfx    x0, x0, #24, #4
    cmp    x0, #1
    b.ne    3f

    mrs_s    x0, SYS_ICC_SRE_EL2
    orr    x0, x0, #ICC_SRE_EL2_SRE    // Set ICC_SRE_EL2.SRE==1
    orr    x0, x0, #ICC_SRE_EL2_ENABLE    // Set ICC_SRE_EL2.Enable==1
    msr_s    SYS_ICC_SRE_EL2, x0
    isb                    // Make sure SRE is now set
    mrs_s    x0, SYS_ICC_SRE_EL2        // Read SRE back,
    tbz    x0, #0, 3f            // and check that it sticks
    msr_s    SYS_ICH_HCR_EL2, xzr        // Reset ICC_HCR_EL2 to defaults

3:
#endif
```

接下来的代码片段旨在 GICv3 被启用且可用的时候才会被执行。GIC 表示通用中断控制器。 v3 版本的 GIC 规范添加了一些特性，这些特性在虚拟化上下文中特别有用。比如，GICv3使虚拟化上下文拥有LPIs(特殊局部外围设备中断)成为可能，这种中断通过消息总线被路由出去，并且这些中断的配置保存在内存中特殊的表结构中。  

上述代码用于启用 SRE(系统寄存器接口)，我们必须在使用 `ICC_*_ELn` 寄存器以及利用 GICv3 特性之前完成这一步。 

```
    /* Populate ID registers. */
    mrs    x0, midr_el1
    mrs    x1, mpidr_el1
    msr    vpidr_el2, x0
    msr    vmpidr_el2, x1
```

`midr_el1` 和 `mpidr_el1` 是标识寄存器组里的只读寄存器，它们提供关于处理器制造商各种各样的信息，比如处理器架构名字，处理器核心数量等等。对尝试在 EL1 下访问这些寄存器的所有访问者来说，寄存器中的信息可能会有变化。这里我们用 `vpidr_el2` 和 ` vmpidr_el2` 来保存来自 `midr_el1` 和 `mpidr_el1` 中的值，这样，这些信息不论我们是在 EL1 还是更高级别的异常下访问它都是相同的。

```
#ifdef CONFIG_COMPAT
    msr    hstr_el2, xzr            // Disable CP15 traps to EL2
#endif
```

当处理器实在32位执行模式下执行，有种叫做 "协处理器" 的说法。协处理器可以用来获取那些在64位模式下通过系统寄存器访问的信息。你可以从 [官方文档](http://infocenter.arm.com/help/index.jsp?topic=/com.arm.doc.ddi0311d/I1014521.html) 中获取关于更确切地使用协处理器访问的详细说明。 `msr    hstr_el2, xzr` 指令允许在低级别异常下使用协处理器。只有在启用兼容模式的时候这个指令才有意义(兼容模式下，64位内核可以运行32位用户应用)。 

```
    /* EL2 debug */
    mrs    x1, id_aa64dfr0_el1        // Check ID_AA64DFR0_EL1 PMUVer
    sbfx    x0, x1, #8, #4
    cmp    x0, #1
    b.lt    4f                // Skip if no PMU present
    mrs    x0, pmcr_el0            // Disable debug access traps
    ubfx    x0, x0, #11, #5            // to EL2 and allow access to
4:
    csel    x3, xzr, x0, lt            // all PMU counters from EL1

    /* Statistical profiling */
    ubfx    x0, x1, #32, #4            // Check ID_AA64DFR0_EL1 PMSVer
    cbz    x0, 6f                // Skip if SPE not present
    cbnz    x2, 5f                // VHE?
    mov    x1, #(MDCR_EL2_E2PB_MASK << MDCR_EL2_E2PB_SHIFT)
    orr    x3, x3, x1            // If we don't have VHE, then
    b    6f                // use EL1&0 translation.
5:                        // For VHE, use EL2 translation
    orr    x3, x3, #MDCR_EL2_TPMS        // and disable access from EL1
6:
    msr    mdcr_el2, x3            // Configure debug traps
```

这段代码用于配置 `mdcr_el2` (调试配置寄存器监视器(Monitor Debug Configuration Register (EL2)))。这个寄存器用于设置与虚拟化扩展有关的不同调试陷阱。我没打算解释这个代码块的细节，因为调试和追踪有点超出我们的讨论范围了。如果你对这些细节感兴趣，我推荐你去阅读  [AArch64 引用手册](https://developer.arm.com/docs/ddi0487/ca/arm-architecture-reference-manual-armv8-for-armv8-a-architecture-profile) 第 `2810` 页关于 `mdcr_el2` 寄存器的描述。

```
    /* Stage-2 translation */
    msr    vttbr_el2, xzr
```

如果你的操作系统用来当作 hypervisor(大概就是虚拟主机管理程序)，那它应该为它的客户端操作系统提供完全隔离的内存。第二阶段的内存转换就是为了这个目的：每一个客户端操作系统都认为自己独占全部系统内存，而事实却是通过第二阶段的内存转换，每份内存都被映射到物理内存。 `vttbr_el2` 持有用于第二季阶段内存转换的转换基地址。此刻，第二阶段内存转换还是被禁用的， `vttbr_el2` 应该被设为0。

```
    cbz    x2, install_el2_stub

    mov    w0, #BOOT_CPU_MODE_EL2        // This CPU booted in EL2
    isb
    ret
```

首先 `x2` 寄存器应该与 `0` 比较来检查 VHE 是否被启用。如果是 - 跳转到 `install_el2_stub` 标签处，否则记录 CPU 是在 EL2 下被引导启动并且从 `el2_setup` 函数中退出。接下来的情况，处理器继续在 EL2 模式下操作，并且 EL1 不会再被使用。

```
install_el2_stub:
    /* sctlr_el1 */
    mov    x0, #0x0800            // Set/clear RES{1,0} bits
CPU_BE(    movk    x0, #0x33d0, lsl #16    )    // Set EE and E0E on BE systems
CPU_LE(    movk    x0, #0x30d0, lsl #16    )    // Clear EE and E0E on LE systems
    msr    sctlr_el1, x0

```

如果到了这一步，就意味着我们不需要 VHE，并且准备切换到 EL1，所在这里需要完成 EL1 前期的初始化。这一段复制的代码段负责初始化 `sctlr_el1` (系统控制寄存器)。在RPi 操作系统中的 [这里](https://github.com/s-matyukevich/raspberry-pi-os/blob/master/src/lesson02/src/boot.S#L18) 我们早就做了同样的事情。

```
    /* Coprocessor traps. */
    mov    x0, #0x33ff
    msr    cptr_el2, x0            // Disable copro. traps to EL2
```

这段代码允许在 EL1 下访问 `cpacr_el1` 寄存器，也就是控制对追踪、浮点以及高级 SIMD 功能的访问。

```
    /* Hypervisor stub */
    adr_l    x0, __hyp_stub_vectors
    msr    vbar_el2, x0
```

我们暂时没打算现在使用 EL2，尽管一些功能需要用到它。我们需要 EL2，比如，在实现 [kexec](https://linux.die.net/man/8/kexec) 系统调用的时候，这个系统调用可以让我们在当前正在运行的内核上加载和引导启动另一个内核。 

[_hyp_stub_vectors](https://github.com/torvalds/linux/blob/v4.14/arch/arm64/kernel/hyp-stub.S#L33) 持有 EL2 全部异常处理程序的地址。下节课在讨论完有关中断和异常处理的细节后我们准备实现 EL1 的异常处理功能。

```
    /* spsr */
    mov    x0, #(PSR_F_BIT | PSR_I_BIT | PSR_A_BIT | PSR_D_BIT |\
              PSR_MODE_EL1h)
    msr    spsr_el2, x0
    msr    elr_el2, lr
    mov    w0, #BOOT_CPU_MODE_EL2        // This CPU booted in EL2
    eret
```

最后，我们需要在 EL1 下初始化处理器状态，并且切换异常级别。我们早在 [RPi 操作系统](https://github.com/s-matyukevich/raspberry-pi-os/blob/master/src/lesson02/src/boot.S#L27-L33) 完成了这一步，我准备解释一下这些代码的详细细节。 

这里唯一新的知识就是 `elr_el2` 是如何被初始化的方式。 `lr` 或者链接寄存器是 `x30` 的别名。无论何时当你执行 `br` (分支链接(Branch Link))指令，`x30` 自动填充当前指令的地址。这个特性经常被 `ret` 指令用到，这样 `ret` 指令就知道具体应该返回到哪里。像我们这种情况，`lr` 指向 [这里](https://github.com/torvalds/linux/blob/v4.14/arch/arm64/kernel/head.S#L119)，因为我们初始化 `elr_el2` 的方式，这也是在切换到 EL1 后执行将重新开始的地方。

### EL1 下的处理器初始化

现在我们回到 `stext` 函数。接下来的几行对我们来说不是很重要，不过为了完整性我还是想解释一下它们。

```
    adrp    x23, __PHYS_OFFSET
    and    x23, x23, MIN_KIMG_ALIGN - 1    // KASLR offset, defaults to 0
```
[KASLR](https://lwn.net/Articles/569635/)，或者说内核地址空间布局随机化，是允许内核随机放在内存地址中的一项技术，这是安全因素方面的要求。你可以阅读上面的链接获取更多信息。

```
    bl    set_cpu_boot_mode_flag
```

在这里 CPU 引导启动模式被保存到 [__boot_cpu_mode](https://github.com/torvalds/linux/blob/v4.14/arch/arm64/include/asm/virt.h#L74) 变量中。这行代码与我们前面研究的 `preserve_boot_args` 函数非常相似。  

```
    bl    __create_page_tables
    bl    __cpu_setup            // initialise processor
    b    __primary_switch
```

最后三个函数非常重要，但是它们都与虚拟内存管理相关，所以我们延迟到第六节课之后再来研究这些函数的细节。现在，我只想简短描述一下他们的含义。
* `__create_page_tables` 就像名字所指的那样，这个函数用于创建一个页表。
* `__cpu_setup` 初始化各种各样的处理器设置，主要是针对虚拟内存管理。
* `__primary_switch` 启用 MMU 并且跳转到 [start_kernel](https://github.com/torvalds/linux/blob/v4.14/init/main.c#L509) 函数，这个函数的起始地址是于特定架构相关的。

### 结论

这一章中，我们简单讨论了在 Linux 内核被引导启动的时候处理器是如何被初始化的。下一节课我们继续使用 ARM 处理器，并且研究一个对任何操作系统都至关重要的话题：中断处理。
 
##### 上一页

2.1 [处理器初始化：RPi 操作系统](../../docs/lesson02/rpi-os.md)

##### 下一页

2.3 [处理器初始化：练习](../../docs/lesson02/exercises.md)
