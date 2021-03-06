## 树莓派 操作系统项目介绍以及如何高效学习操作系统开发?

几年以前，我第一次打开了 Linux 内核的源代码。那时候，我认为自己或多或少算是一个熟练的软件开发人员：我知道一些汇编和 C 的编程，对操作系统主要的概念有着较高层次的理解，比如进程调度器和虚拟内存管理。然而我的第一次尝试却彻彻底底的失败了 - 我几乎什么都不懂。

对于我接手过的其他软件项目而言，我有一个简单且行之有效的方法：找到程序的入口点，然后开始阅读源代码，尽可能的深入了解我感兴趣的所有细节。 这种方法很有效，但是并不适应像操作系统这样复杂的项目。这不仅是指我花了一个多星期才找到入口 - 更主要的问题是很快我就发现我处于这样的一个境地：对于我正在阅读的某几行代码，关于如何去寻找到它们究竟起到什么作用的线索，我毫无头绪。 对于低级汇编源代码来说，这一情况尤其正确，对于我尝试研究的操作系统源代码的其他部分，情况也好不到哪里去。 

然而，我不喜欢因为问题看起来开头比较复杂就放弃它。而且，我也相信没有真正复杂的问题 - 相反，有的只是一些我们不知道如何有效解决的问题。 所以我开始寻找学习通用操作系统，特别是学习Linux的有效方法。

### 学习开发操作系统的挑战

我知道有非常多关于 Linux 内核开发的书籍和文档，但是都没有我想要的学习经验。我发现这其中有一半的资料是我早就已经掌握了的高级内容(虚拟内存、进程调度等)。另一半的资料，存在着和我在阅读 Linux 内核源码的时候相似的问题：一旦解读源码过于深入，那么90%的细节都看起来和核心概念无关，而是一些安全性、性能或是历史遗留问题，以及 Linux 内核所支持的数百万个功能特性。所以，你不是在学习操作系统的核心概念，而是在挖掘实现这些功能特性的细枝末节。

你可能想知道当初我为什么想要学习操作系统的开发。于我而言，最主要的一个原因是，我总是对事情的背后究竟是怎样工作的感兴趣。 这并不仅仅是由于好奇心：而是当你处理的工作任务越复杂，你就越是经常能够追溯到来自操作系统层次的原因。如果你不理解底层是如何工作的，那么你就无法做出正确的技术决策。另一方面，如果你是真的喜欢挑战技术，那么开发操作系统这个任务肯定能让你兴奋起来。

下一个你想问的问题可能是，为什么是 Linux 操作系统呢? 其他操作系统可能更容易理解啊。 我的回答是，我希望我的知识至少在某种程度上是与我目前所做的事情相关的，或者我将来可能要做的事情有关。 从这方面来说，Linux 非常合适，因为如今从小型物联网设备到大型服务器都倾向于在 Linux 操作系统上运行。

在我说大部分关于 Linux 内核开发的书籍对于我没用的时候 - 我有点儿不太诚实。有一本书用实际的源代码解释了一些我完全能够理解清楚的基本概念，尽管我还是一个操作系统开发的新手。 这本书名为《Linux Device Drivers》， 是当之无愧的关于 Linux 内核的著名技术书籍之一。 它首先介绍了一个你可以编译并运行的简单驱动程序源代码。然后开始逐个介绍驱动程序相关的新概念，并且解释如何修改源代码以运用这些新概念。这正是我想要的 “有效的学习经验”。这本书唯一的问题是，他明确只关注驱动程序的开发，而很少涉及内核核心的实现细节。

那么为什么没有人去编写一本类似的关于内核开发的书呢? 我想可能是因为你没办法基于当前的 Linux 源代码来写你的书。由于 Linux 源代码非常复杂，以至于你根本没有一个函数、结构或者模块可以用来作为简单的起始点。你也没办法一次引入一个新概念，因为在源代码中，概念之间都是密切相关的。当我意识到这一点后，我萌生了一个想法：如果 Linux 内核太大太复杂，不能用作学习操作系统开发的起点，那我为什么不自己实现一个完全是为了学习的目的而设计的操作系统？这样的话，我可以使操作系统足够简单，以便于提供良好的学习体验。同时，假如这个操作系统主要是通过参考和简化 Linux 内核源代码的不同部分来实现的话，那么它也可以直接作为学习 Linux 内核的起点。除了操作系统之外，我还决定写一个系列的讲课，来讲解操作系统开发的一些主要概念，并且详细解释操作系统的源代码。

### 操作系统要求

基于以上的原因，我开始着手创建这个项目，也就是现在的 [RPi 操作系统](https://github.com/s-matyukevich/raspberry-pi-os)。 我必须要做的第一件事就是确定内核开发的哪些部分是我认为 "最基本的"，以及哪些是我认为不必要的、可以跳过的组件(至少在刚开始学习的是这样)。 在我的理解当中，每个操作系统都有两个基本目标：

1. 独立的运行用户进程。
1. 为每个用户进程提供一致的的机器硬件层。

为了满足第一个要求，RPi 操作系统需要有他自己的进程调度器。如果我想实现一个调度器，我还必须要去处理时钟中断。第二个要求意味着操作系统应该支持一些驱动程序，并且通过系统调用的方式暴露给用户程序。因为这是面向初学者的，所以我不想使用复杂的硬件，所以我唯一关心的驱动程序是，向屏幕写入数据以及从键盘读取用户输入。此外，这个操作系统还需要能够加载和运行用户程序，所以自然而然的它需要支持某种文件系统，并且能够解析某种可执行文件格式。如果这个操作系统能支持基本的网络当然是再好不过了，但是我并不想让它出现在初学者关注的点中。以上就是我认为"所有操作系统核心概念"的东西。

现在，让我们来看一下我想要忽略的一些东西：
1. **性能** 我不想在操作系统中使用任何复杂的算法。为了简单起见，我还将禁用所有的缓存和其他性能优化的技术。
1. **安全性** RPi 操作系统仅只有一个安全特性：虚拟内存。其他一切都可以忽略。
1. **多进程以及同步** 我非常开心我的操作系统运行在单个处理器核心上。 特别是这能够让我摆脱巨大源代码复杂性的来源 - 同步。 
1. **支持多种体系架构和不同的设备类型** 以后的课程将会涉及到更多这一方面的内容。
1. **其他成熟操作系统所支持的数以百万的功能和特性**

### 树莓派 操作系统怎么玩

我早就介绍过我不想让 RPi 操作系统支持多种计算机体系架构或者不同类型的设备。在研究了 Linux 内核驱动模型之后，这一想法更为强烈。看起来非常类似的设备，实现同样的功能也可能在细节上大相径庭。这使得在不同的设备类型上提出简单的抽象，并复用驱动程序源代码变得十分困难。在我看来，这可能就是导致 Linux 内核复杂性的主要来源之一，我当然要在 RPi 操作系统上避免出现这种情况。 好了，那么我要用哪种类型的计算机呢?我显然不想用我的笔记本电脑来测试我的裸板程序，因为我真的不敢保证我的笔记本电脑能够幸存下来。更重要的是，我不想让别人买一台昂贵的笔记本电脑来跟我学习操作系统的开发练习(当然我也不认为会有人这样做)。模拟器看起来或许是个不错的选择，但是我想在一台真正的设备上学习，因为这会让我感觉自己在做一些真实的事情，而不仅仅是在玩裸板程序。

我最终选择了树莓派，更确切的说，是 [树莓派三代B型](https://www.raspberrypi.org/products/raspberry-pi-3-model-b/)。使用这个设备看起来像是出于以下多种原因的理想选择：

1. 它大概只需要花费 35 美元。我觉得这应该是一个能负担得起的价格。
1. 这个设备是专门为了学习而设计的。它的内部结构足够简单，完全符合我的需要。
1. 设备使用了 ARM v8 架构。这是一个简单的 RISC(精简指令集) 架构，很适合操作系统开发者的需要，也没有太多的历史遗留问题方面的要求，比如说流行的 X86 架构。如果你不相信我，你可以对比一下 Linux内核中 `/arch/arm64` 和 `/arch/x86` 两个文件夹下面源代码的数量。

这个操作系统与老版本的树莓派不兼容，因为它们都不支持64位的ARM V8架构，而我认为适配所有的设备是一件十分琐碎的事情。

### 与社区协作

任何技术类书籍的一个主要缺点是，在版本发布不久后它们就过时了。现今技术发展的如此之快，以至于写书的人很难跟上技术发展的速度。这就是为什么我喜欢 “开源书籍” - 一本在互联网上可免费获取的书籍，并且鼓励读者参与内容创建和校验的一本书。这样的话，如果一本书的内容放在github上,那么任何读者都可以很容易的修复和开发新的示例代码，更新这本书的内容，以及参与到新章节的编写当中来。我明白现在这个项目还不完美，甚至在写这些的时候这个项目都还没有完结。但是我仍然想现在就将它发布出来，因为我希望在社区的帮助下，不但能够让我快速完成这个项目，而且能够使它比刚开始时更完美更实用。 

##### 上一页

[主页](https://github.com/s-matyukevich/raspberry-pi-os#learning-operating-system-development-using-linux-kernel-and-raspberry-pi)

##### 下一页

[向 树莓派 操作系统贡献](../docs/Contributions.md)