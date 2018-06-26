## 先决条件

### 1. [树莓派三代B型](https://www.raspberrypi.org/products/raspberry-pi-3-model-b/)

较早版本的树莓派不适用于本教程，因为所有的课程都是基于支持ARMv8架构的64位处理器而设计，而这种处理器只在树莓派三代上可用。更新的树莓派版本，包括 [树莓派三代B+](https://www.raspberrypi.org/products/raspberry-pi-3-model-b-plus/) 应该也能兼容，不过目前为止我还没有对它做过测试。

### 2. [USB转TTL线](https://www.amazon.com/s/ref=nb_sb_noss_2?url=search-alias%3Daps&field-keywords=usb+to+ttl+serial+cable&rh=i%3Aaps%2Ck%3Ausb+to+ttl+serial+cable) 

当你拿到串口转接线，你需要测试它是否可以连接。如果你以前从来没有接触过这些，我推荐你去阅读 [这篇指南](https://cdn-learn.adafruit.com/downloads/pdf/adafruits-raspberry-pi-lesson-5-using-a-console-cable.pdf)。它详细的描述了通过串口线连接树莓派的详细过程。 

这篇指南同时也描述了如何通过串口线给树莓派供电。 RPi 操作系统在这种设置下运作良好，但是，在这种设置下你必须在插上串口线之后立刻运行你的终端模拟器。关于这个问题的详细信息可以看 [这里](https://github.com/s-matyukevich/raspberry-pi-os/issues/2)...
 
### 3. 安装了 [树莓派操作系统](https://www.raspberrypi.org/downloads/raspbian/) 的 [SD 卡](https://www.raspberrypi.org/documentation/installation/sd-cards.md) 

我们需要树莓派来测试USB转TTL串口线的连接初始化。此外，在安装完成后，我们也需要树莓派来正确格式化SD卡。

### 4. Docker

严格来说, Docker 并不是必须的。它只是方便我们构建编译每节课的源代码，特别是对于Mac和Windows用户来说。每节课都有一个 `build.sh` 脚本 (对Windows用户来说是 `build.bat` )，这个脚本使用了Docker来构建每节课的源代码。关于如何安装使用Docker的介绍请参考 [docker 官方网站](https://docs.docker.com/engine/installation/)。  

如果你不想使用Docker，你可以安装 [make utility](http://www.math.tau.ac.il/~danha/courses/software1/make-intro.html) 或者 `aarch64-linux-gnu` 工具链。如果你使用的是Ubuntu，你只需要安装 `gcc-aarch64-linux-gnu` 和 `build-essential` 两个软件包。

##### 上一页

[贡献指南](../docs/Contributions.md)

##### 下一页

1.1 [RPi 操作系统介绍， 以及裸板程序 "Hello, world!"](../docs/lesson01/rpi-os.md)
