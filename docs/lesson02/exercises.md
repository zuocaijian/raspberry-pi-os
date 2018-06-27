## 2.3: 练习

1. 不直接从 EL3 跳到 EL1，试着先跳到 EL2 然后再跳到 EL1。 
1. 在这节课上我遇到的一个问题是，如果使用 FP/SIMD 寄存器，那么在 EL3 下一切运行正常，但是你一旦切换到 EL1 下，那么打印函数就失效了。这就是我在编译器可选项中添加 [-mgeneral-regs-only](https://github.com/s-matyukevich/raspberry-pi-os/blob/master/src/lesson02/Makefile#L3) 参数的原因。现在我想让你移除参数去复现这个问题。接下来你可以使用 `objdump` 工具查看在没有 `-mgeneral-regs-only` 标志的情况下 gcc 究竟是怎么使用 FP/SIMD 寄存器的。接下来，我想让你通过 'cpacr_el1' 来允许使用 FP/SIMD 寄存器。
1. 将第二节课的内容为qemu(模拟器)进行适配。 查看 [这个](https://github.com/s-matyukevich/raspberry-pi-os/issues/8) 问题作为参考。

##### 上一页

2.2 [处理器初始化： Linux](../../docs/lesson02/linux.md)

##### 下一页

3.1 [中断处理： RPi 操作系统](../../docs/lesson03/rpi-os.md)
