## 1.2: Linux项目结构

这是我们第一次正式谈论到Linux。我的想法是首先在我们自己的内核中完成一些小步骤，然后来看看同样的东西在Linux中是如何工作的。尽管到目前为止我们只完成了一点点：那就是我们刚刚实现的第一个裸板程序 hello world，但是，我们还是能够找到 RPi 操作系统和 Linux 操作系统之间的一些相似之处。现在我们就来探究一下这些相似之处。 

### 项目结构

无论何时你准备去研究任何一个大型软件项目，快速浏览一下项目的架构总是非常有用的。这样你可以知道项目是由哪些模块组成，以及项目的上层结构是什么样的，这一点非常重要。所限，让我们先来快速看一眼Linux内核的项目结构。

首先，你需要在本地克隆一份Linux的仓库代码

```
git clone https://github.com/torvalds/linux.git 
cd linux
git checkout v4.14
```

我们准备使用 `v4.14` 这个版本的Linux源代码，这也是在写这篇教程的时候最新的Linux源代码。对Linux源码的全部引用都是使用的这个特定版本。

接下来，让我们来看看Linux仓库中我们都能找到哪些文件夹。我们并不打算把他们全部过一遍，而是只关注我认为在刚开始的时候费用重要的那一部分。

* [arch](https://github.com/torvalds/linux/tree/v4.14/arch) 这个文件夹下面包含了许多子文件夹，每一个子文件夹都对应一个特定体系架构的处理器。大部分时候我们都是。大部分时候我们都是使用 [arm64](https://github.com/torvalds/linux/tree/v4.14/arch/arm64) 这个文件夹 - 他能够兼容 ARM.v8 处理器。
* [init](https://github.com/torvalds/linux/tree/v4.14/init) 内核总是由特定架构相关的代码来引导启动，不过最后总是执行到 [start_kernel](https://github.com/torvalds/linux/blob/v4.14/init/main.c#L509) 这个函数处，这个函数负责初始化通用内核，并且是与体系结构无关的内核起始点。 `start_kernel` 函数，以及其它的一些初始化函数，都定义在 `init` 这个文件夹下。
* [kernel](https://github.com/torvalds/linux/tree/v4.14/kernel) 这是 Linux 内核的核心。几乎所有的主要内核子系统都是在这里实现的。
* [mm](https://github.com/torvalds/linux/tree/v4.14/mm) 与内存管理相关的所有数据结构和方法都定义在这里。 
* [drivers](https://github.com/torvalds/linux/tree/v4.14/drivers) 这是 Linux 内核中最大的一个文件夹。它包含了全部设备驱动的实现。
* [fs](https://github.com/torvalds/linux/tree/v4.14/fs) 在这里你可以看到不同文件系统的实现方式。

以上解读非常浅显，不过对现在的我们来说已经足够了。在下一章，我们将尝试研究一下 Linux 构建系统相关的一些内容。 

##### 上一页

1.1 [内核初始化：RPi 操作系统介绍， 以及裸板程序 "Hello, world!"](../../../docs/lesson01/rpi-os.md)

##### Next Page

1.3 [内核初始化：内核构建系统](../../../docs/lesson01/linux/build-system.md)
