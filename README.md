# 使用Linux内核和树莓派学习操作系统的开发

本项目涵盖一份详细的指南：教你如何从零开始实现一个简单的操作系统(OS)。我把这个操作系统称为 树莓派 操作系统，或简称 RPi 操作系统。 RPi 操作系统的大部分源码基于 [Linux 内核](https://github.com/torvalds/linux)，不过这个操作系统的功能十分有限，且仅支持 [Raspberry PI 3](https://www.raspberrypi.org/products/raspberry-pi-3-model-b/)。 

每一节课程教学都遵循如下设计：首先解释 RPi 操作系统实现了哪些内核特性和功能，接着演示该功能在 Linux 内核中是如何工作的。在源代码 [src](https://github.com/s-matyukevich/raspberry-pi-os/tree/master/src) 目录下有对应的每节课的文件夹, 里面包含了本次课程结束时 RPi 操作系统的一份源代码快照。这样可以保证由浅入深逐步引入新的专业概念，并且有助于读者清晰的看到 RPi 操作系统是如何演进的。理解这份教学指南不要求掌握任何特别的操作系统开发技术。

关于本项目更多的目的和历史信息, 请参阅 [Introduction](docs/Introduction.md)。目前本项目仍处于开发状态, 如果你有兴趣参与 - 请参阅 [Contribution guide](docs/Contributions.md)。

<a href="https://twitter.com/RPi_OS" target="_blank">
  <img src="https://raw.githubusercontent.com/s-matyukevich/raspberry-pi-os/master/images/twitter.png" alt="Follow @RPi_OS on twitter" height="34" >
</a>

<a href="https://www.facebook.com/groups/251043708976964/" target="_blank">
  <img src="https://raw.githubusercontent.com/s-matyukevich/raspberry-pi-os/master/images/facebook.png" alt="Follow Raspberry Pi OS on facebook" height="34" >
</a>

<a href="https://www.producthunt.com/upcoming/raspberry-pi-os" target="_blank">
  <img src="https://raw.githubusercontent.com/s-matyukevich/raspberry-pi-os/master/images/subscribe.png" alt="Subscribe for updates" height="34" >
</a>

## 目录

* **[引言](docs/Introduction.md)**
* **[贡献指南](docs/Contributions.md)**
* **[先决条件](docs/Prerequisites.md)**
* **第 一 节课: 内核初始化** 
  * 1.1 [RPi 操作系统介绍， 以及裸板程序 "Hello, world!"](docs/lesson01/rpi-os.md)
  * Linux
    * 1.2 [项目结构](docs/lesson01/linux/project-structure.md)
    * 1.3 [内核构建系统](docs/lesson01/linux/build-system.md) 
    * 1.4 [开机流程](docs/lesson01/linux/kernel-startup.md)
  * 1.5 [练习](docs/lesson01/exercises.md)
* **第 二 节课: 处理器初始化**
  * 2.1 [RPi 操作系统](docs/lesson02/rpi-os.md)
  * 2.2 [Linux](docs/lesson02/linux.md)
  * 2.3 [练习](docs/lesson02/exercises.md)
* **第 三 节课: 中断处理**
  * 3.1 [RPi 操作系统](docs/lesson03/rpi-os.md)
  * Linux
    * 3.2 [低级异常处理](docs/lesson03/linux/low_level-exception_handling.md) 
    * 3.3 [中断控制器](docs/lesson03/linux/interrupt_controllers.md)
    * 3.4 [时钟](docs/lesson03/linux/timer.md)
  * 3.5 [练习](docs/lesson03/exercises.md)
* **第 四 节课: 进程调度器**
  * 4.1 [RPi 操作系统](docs/lesson04/rpi-os.md) 
  * Linux
    * 4.2 [任务调度器基本结构](docs/lesson04/linux/basic_structures.md)
    * 4.3 [拷贝一个任务](docs/lesson04/linux/fork.md)
    * 4.4 [任务调度器](docs/lesson04/linux/scheduler.md)
  * 4.5 [练习](docs/lesson04/exercises.md)
* **第 五 节课: 用户进程和系统调用** 
  * 5.1 [RPi 操作系统](docs/lesson05/rpi-os.md) 
  * 5.2 [Linux](docs/lesson05/linux.md)
  * 5.3 [练习](docs/lesson05/exercises.md)
* **第 六 节课: 虚拟内存管理**
  * 6.1 [RPi 操作系统](docs/lesson06/rpi-os.md) 
  * 6.2 Linux (进行中)
  * 6.3 [练习](docs/lesson06/exercises.md)
* **第 七 节课: 信号和中断/等待** (待完成)
* **第 八 节课: 文件系统** (待完成)
* **第 九 节课: 可执行文件 (ELF)** (待完成)
* **第 十 节课: 驱动** (待完成)
* **第 十一 节课: 网络** (待完成)

