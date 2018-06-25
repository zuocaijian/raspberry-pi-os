## 1.3: 内核构建系统。

看完了 Linux 内核项目的组织架构，现在是时候花点时间来研究一下怎么编译和运行了。 Linux 同样也是使用 `make` 工具来构建内核，不过 Linux 的 makefile 要复杂的多。在阅读 Linux 的 makefile 之前，我们先来学习一下关于 Linux 系统构建的一些重要概念，它被称为 "kbuild"。

### 关于 kbuild 的几个基本概念

* 使用 kbuild 变量可以自定义构建过程。这些变量定义在 `Kconfig` 文件当中。 这个文件中你可以定义变量并给它们赋默认值。这些变量有不同的类型，包括字符串型、布尔型以及整型。在 Kconfig 文件中，你也可以定义变量之间的依赖关系(比如，你可以说，如果变量x被选中，那么变量y将被隐式选中)。 举个例子，你可以去看看 [arm64 Kconfig file](https://github.com/torvalds/linux/tree/v4.14/arch/arm64/Kconfig)。这个文件定义了针对于 `arm64` 体系架构所有的变量。 `Kconfig` 的功能并不是 `make` 工具的标准组成，它只是 Linux 中makefile的一种具体实现。 `Kconfig` 中定义的变量暴露在内核源代码以及嵌套的 makfile 文件中。这内核配置这一步中，变量可以被赋值(比如，如果你键入 `make menuconfig` ，一个图形化的命令行将会出现。这个图形化的命令行可以让你对所有的内核变量自定义赋值，并且将值储存在 `.config` 文件中。使用 `make help` 命令可以查看内核配置相关的所有可选项)

* Linux 使用递归构建。也就是说每一个 Linux 内核的子文件夹可以定义它们自己的 `Makefile` 和 `Kconfig`。大部分嵌套的 Makefile 文件都非常简单，仅仅是定义了需要被编译的目标对象文件。通常，这些定义的格式如下：

  ```
  obj-$(SOME_CONFIG_VARIABLE) += some_file.o
  ```

  这个定义表明 `some_file.c` 文件只在 `SOME_CONFIG_VARIABLE` 变量被设置的时候才会被编译并且链接到内核中。如果你需要无条件的编译和链接一个文件，你需要把上面的定义改成如下所示的样子。

  ```
  obj-y += some_file.o
  ```

  在 [这里](https://github.com/torvalds/linux/tree/v4.14/kernel/Makefile) 可以找到嵌套 makefile 的示例。

* 在进行下一步研究之前，你需要了解一下基本的 make 编写规则和结构，并且熟悉相关术语。通用规则和结构如下图所示。 

  ```
  targets : prerequisites
          recipe
          …
  ```
    * `targets` 是一些使用空格分割的文件名。目标文件是由构建规则执行后所产生的。通常，一条规则只有一个目标文件。
    * `prerequisites` 是依赖文件列表， `make` 通过跟踪这个文件列表来确定是否需要重新生成目标文件。
    * `recipe` 是一个命令行命令。当依赖文件列表有变化时，Make 就会调用这个命令行命令。这个命令负责生成目标文件。
    * targets 和 prerequisites 都可以使用通配符(`%`)。在脚本中使用通配符时，脚本会分别将每个匹配上的依赖文件执行一遍。在这个示例中，你可以使用 `$<` 和 `$@` 两个变量来引用依赖文件列表和目标文件。我们在 [RPi OS makefile](https://github.com/s-matyukevich/raspberry-pi-os/blob/master/src/lesson01/Makefile#L14)中早就用到了这个特性。关于 make 规则的更多信息，请参阅 [官方文档](https://www.gnu.org/software/make/manual/html_node/Rule-Syntax.html#Rule-Syntax)

* `make` 非常善于侦测是否有依赖文件发生了更改，以及在需要的时候重新编译生成目标文件。但是，如果命令是动态更新的， `make` 就没办法检测到这些改变。 怎么会这样子呢?非常简单。 一个很好的例子就是当你改变一些配置变量时，就会导致命令中增加一个附加选项。默认情况下，这时候 `make` 不会重新编译生成目标文件，因为依赖文件没有发生变化，仅仅是命令更新。为了应付这种情况，Linux 引入了 [if_changed](https://github.com/torvalds/linux/blob/v4.14/scripts/Kbuild.include#L264) 的方法。想了解它的工作原理，请看下面这个示例。

  ```
  cmd_compile = gcc $(flags) -o $@ $<

  %.o: %.c FORCE
      $(call if_changed,compile)
  ```

  在这里我们通过调用带有 `compile` 实参的 `if_changed` 函数使每一个 `.c` 文件都生成对应的 `.o` 文件。 `if_changed` 函数寻找 `cmd_compile` 这个变量 (这个函数给它的首个参数添加了 `cmd_` 前缀)，并且检查这个变量值在上一次执行之后是否有更新，以及其他依赖是否发生了变化。如果有更新或者是变化 - `cmd_compile` 这条命令将被执行，重新生成目标文件。我们的示例规则中包含了2个依赖： `.c` 源文件和 `FORCE`。 `FORCE` 是一个特殊的依赖，它会在每一次 `make` 命令被调用的时候强制调用命令行命令。如果没有 `FORCE` 这个依赖，那么命令行命令只会在 `.c` 文件发生了变化的时候才被调用。你可以在 [这里](https://www.gnu.org/software/make/manual/html_node/Force-Targets.html)  阅读更多关于 `FORCE` 依赖的信息。  

### 构建内核

现在，我们已经学完了关于 Linux 构建系统的一些重要的概念，让我们来试着弄清楚当你键入 `make` 命令后到底发生了什么事。这个过程非常复杂，包含了非常多的细节，不过我们这其中很多细节我们都将跳过。我们的目标就是要弄清楚下面两个问题。

1. 源文件究竟是怎么编译成目标文件的?
1. 目标文件是怎么被链接到操作系统镜像文件中的?

我们首先来解决第二个问题。

#### 链接阶段 

* 就像你应该在 `make help` 命令输出中看到过的那样，默认的目标，是用来构建内核的，这个默认目标被称为 `vmlinux`

* `vmlinux` 目标的定义可以在 [这里](https://github.com/torvalds/linux/blob/v4.14/Makefile#L1004) 找到，看起来像是下面这样子的：

  ```
  cmd_link-vmlinux =                                                 \
      $(CONFIG_SHELL) $< $(LD) $(LDFLAGS) $(LDFLAGS_vmlinux) ;    \
      $(if $(ARCH_POSTLINK), $(MAKE) -f $(ARCH_POSTLINK) $@, true)

  vmlinux: scripts/link-vmlinux.sh vmlinux_prereq $(vmlinux-deps) FORCE
      +$(call if_changed,link-vmlinux)
  ```

  这个目标使用了我们熟悉的 `if_changed` 函数。无论什么依赖发生了更新， `cmd_link-vmlinux` 这条命令都将被执行。这条命令会执行 [scripts/link-vmlinux.sh](https://github.com/torvalds/linux/blob/v4.14/scripts/link-vmlinux.sh) 这个脚本(注意，在 `cmd_link-vmlinux` 命令中自动变量 [$<](https://www.gnu.org/software/make/manual/html_node/Automatic-Variables.html) 的用法)。 它同时还执行了特定架构下的 [postlink script](https://github.com/torvalds/linux/blob/v4.14/Documentation/kbuild/makefiles.txt#L1229) 这个脚本，但我们并不关系这个脚本。

* 当 [scripts/link-vmlinux.sh](https://github.com/torvalds/linux/blob/v4.14/scripts/link-vmlinux.sh) 脚本被执行的时候，它认为所有需要的对象文件都已经构建好，并且这些对象文件的位置(地址)被保存在3个变量中： `KBUILD_VMLINUX_INIT`, `KBUILD_VMLINUX_MAIN`, `KBUILD_VMLINUX_LIBS`。  

* `link-vmlinux.sh` 首先为所有的对象文件创建 `thin archive` 。 `thin archive` 是一个包含了一系列对象文件(归档成员文件)以及符号索引表的引用。这是在 [archive_builtin](https://github.com/torvalds/linux/blob/v4.14/scripts/link-vmlinux.sh#L56) 函数中完成的。为了创建 `thin archive` ，这个函数使用了 [ar](https://sourceware.org/binutils/docs/binutils/ar.html) 工具。 生成好的 `thin archive` 被保存为 `built-in.o` 文件，其文件格式可以被链接器理解识别，所以它可以和任何其他对象文件一样被使用。

* 接下来 [modpost_link](https://github.com/torvalds/linux/blob/v4.14/scripts/link-vmlinux.sh#L69) 被调用。这个函数会调用链接器并生成 `vmlinux.o` 对象文件。我们需要这个这个对象文件来执行 [Section mismatch analysis](https://github.com/torvalds/linux/blob/v4.14/lib/Kconfig.debug#L308)。这个分析是由 [modpost](https://github.com/torvalds/linux/tree/v4.14/scripts/mod) 程序执行的，具体是在 [这](https://github.com/torvalds/linux/blob/v4.14/scripts/link-vmlinux.sh#L260) 行被触发。

* 然后生成内核符号表。这个符号表包含了 `vmlinux` 二进制文件中全部的函数、全局变量以及位置(地址)信息。这些工作主要是在 [kallsyms](https://github.com/torvalds/linux/blob/v4.14/scripts/link-vmlinux.sh#L146) 函数中完成的。这个函数首先使用 [nm](https://sourceware.org/binutils/docs/binutils/nm.html) 从 `vmlinux` 二进制文件中获取全部的符号。然后使用 [scripts/kallsyms](https://github.com/torvalds/linux/blob/v4.14/scripts/kallsyms.c)  工具生成一个包含了全部符号且能被 Linux 内核解析识别的特殊汇编文件。接下来，这个汇编文件被编译，并与原始的二进制文件在一起被链接在一起。这个过程将重复进行多次，因为在最终链接完成前，一些符号的地址可能会变。内核符号表中的信息用来在运行时生成 `/proc/kallsyms` 文件。

* 最后 `vmlinux` 二进制文件有了， `System.map`  也构建好了。 `System.map` 包含了和 `/proc/kallsyms` 一样的信息，不过 `System.map` 是一个静态文件，不像 `/proc/kallsyms` 那样是在运行时生成的。 `System.map` 一般是用来在 [kernel oops](https://en.wikipedia.org/wiki/Linux_kernel_oops) 的时候获取符号的地址。 `nm` 这个工具也还被用来构建生成 `System.map`。这个过程是在 [这里](https://github.com/torvalds/linux/blob/v4.14/scripts/mksysmap#L44) 完成的。

#### 构建阶段 

* 现在我们再回过头来看一下源代码文件是如何被编译成对象文件的。你可能还记得生成 `vmlinux` 目标文件的依赖之一就是 `$(vmlinux-deps)` 这个变量。让我们拷贝几行 Linux makefile 文件中相关的代码来演示下这个变量是如何构建得到的。 

  ```
  init-y        := init/
  drivers-y    := drivers/ sound/ firmware/
  net-y        := net/
  libs-y        := lib/
  core-y        := usr/

  core-y        += kernel/ certs/ mm/ fs/ ipc/ security/ crypto/ block/

  init-y        := $(patsubst %/, %/built-in.o, $(init-y))
  core-y        := $(patsubst %/, %/built-in.o, $(core-y))
  drivers-y    := $(patsubst %/, %/built-in.o, $(drivers-y))
  net-y        := $(patsubst %/, %/built-in.o, $(net-y))

  export KBUILD_VMLINUX_INIT := $(head-y) $(init-y)
  export KBUILD_VMLINUX_MAIN := $(core-y) $(libs-y2) $(drivers-y) $(net-y) $(virt-y)
  export KBUILD_VMLINUX_LIBS := $(libs-y1)
  export KBUILD_LDS          := arch/$(SRCARCH)/kernel/vmlinux.lds

  vmlinux-deps := $(KBUILD_LDS) $(KBUILD_VMLINUX_INIT) $(KBUILD_VMLINUX_MAIN) $(KBUILD_VMLINUX_LIBS)
  ```

  这几行代码都是以 `init-y`, `core-y` 等这样的变量开头，这些变量包含了可编译源代码的Linux内核的所有子文件夹。然后 `built-in.o` 被追加到全部子文件夹名字的后面，比如 `drivers/` 就变成了 `drivers/built-in.o`。然后 `vmlinux-deps` 聚合了所有的这些值。这就解释了为什么 `vmlinux` 实际上是依赖全部的 `build-in.o` 文件。

* 下一个问题是 `built-in.o` 这些对象是怎么被创建的? 同样的，让我们再次拷贝几行相关的代码来解释这个过程是怎么完成的。

  ```
  $(sort $(vmlinux-deps)): $(vmlinux-dirs) ;

  vmlinux-dirs    := $(patsubst %/,%,$(filter %/, $(init-y) $(init-m) \
               $(core-y) $(core-m) $(drivers-y) $(drivers-m) \
               $(net-y) $(net-m) $(libs-y) $(libs-m) $(virt-y)))

  build := -f $(srctree)/scripts/Makefile.build obj               #Copied from `scripts/Kbuild.include`

  $(vmlinux-dirs): prepare scripts
      $(Q)$(MAKE) $(build)=$@

  ```

  第一行代码表示 `vmlinux-deps` 依赖于 `vmlinux-dirs`。接下来，我们看到 `vmlinux-dirs` 这个变量包含了根目录下全部以 `/` 字符结尾的子文件夹。这其中最重要的一行是构建生成 `$(vmlinux-dirs)` 目标文件的命令行命令。如果把所有的变量都替换掉，那么这个命令看起来就像下面这样(这里我们用 `drivers` 文件夹为例，实际上这个命令将会作用于根目录下的全部子文件夹)

  ```
  make -f scripts/Makefile.build obj=drivers
  ```

  这行代码只是调用了另一个 makefile 文件 ([scripts/Makefile.build](https://github.com/torvalds/linux/blob/v4.14/scripts/Makefile.build))，并且传递一个包含待编译文件夹的 `obj` 变量。 

* 接着我们来看下一个构建逻辑的流程 [scripts/Makefile.build](https://github.com/torvalds/linux/blob/v4.14/scripts/Makefile.build)。这个脚本被执行后发生的第一件重要的事就是那些定义在当前目录以及在 `Makefile` 和 `Kbuild` 这些文件中的变量。当前目录是指 `obj` 变量所引用的目录。这个过程由 [这三行](https://github.com/torvalds/linux/blob/v4.14/scripts/Makefile.build#L43-L45) 代码完成。

  ```
  kbuild-dir := $(if $(filter /%,$(src)),$(src),$(srctree)/$(src))
  kbuild-file := $(if $(wildcard $(kbuild-dir)/Kbuild),$(kbuild-dir)/Kbuild,$(kbuild-dir)/Makefile)
  include $(kbuild-file)
  ```
  嵌套的那些 makefile 文件大部分用来负责初始化像 `obj-y` 这样的变量。 快速回顾： `obj-y` 变量应该包含位于当前目录下的全部源代码文件列表。另一个被嵌套 makefile 文件初始化的重要变量是 `subdir-y`。这个变量包含了在构建当前目录源代码前应该被检查的全部子文件夹的一个列表。 `subdir-y` 用于实现完成对子文件夹的递归。

* 当调用不带特殊目标的 `make` 命令时(比如在执行 `scripts/Makefile.build` 的情况下)，它会把第一个目标当作目标。 `scripts/Makefile.build` 的第一个目标是 `__build`，可以在 [这里](https://github.com/torvalds/linux/blob/v4.14/scripts/Makefile.build#L96) 找到。让我们来看一下它。

  ```
  __build: $(if $(KBUILD_BUILTIN),$(builtin-target) $(lib-target) $(extra-y)) \
       $(if $(KBUILD_MODULES),$(obj-m) $(modorder-target)) \
       $(subdir-ym) $(always)
      @:
  ```

  如你所见， `__build` 目标没有命令行命令，但是他依赖于一大串其它的目标。我们只对 `$(builtin-target)` 感兴趣 - 它负责创建 `built-in.o` 文件，以及 `$(subdir-ym)` - 负责递归进入到嵌套子目录。

* 我们来看一下 `subdir-ym`。这个变量在 [这里](https://github.com/torvalds/linux/blob/v4.14/scripts/Makefile.lib#L48) 被初始化，它就是 `subdir-y` 和 `subdir-m` 两个变量的的级联。 (`subdir-m` 和 `subdir-y` 相似，但 `subdir-m` 含有需要被单独的 [kernel module(内核模块)](https://en.wikipedia.org/wiki/Loadable_kernel_module) 所包含的子文件夹。现在我们先跳过对 模块 的讨论，保持注意力。) 

*  `subdir-ym` 目标定义在 [这里](https://github.com/torvalds/linux/blob/v4.14/scripts/Makefile.build#L572)，对你而言，它看起来应该比较熟悉。

  ```
  $(subdir-ym):
      $(Q)$(MAKE) $(build)=$@
  ```

  这个目标就是在其中一个嵌套子文件夹中触发执行 `scripts/Makefile.build`。

* 现在是时候看一下 [builtin-target](https://github.com/torvalds/linux/blob/v4.14/scripts/Makefile.build#L467) 这个目标了。同样，我把相关的几行代码拷贝到这里。

  ```
  cmd_make_builtin = rm -f $@; $(AR) rcSTP$(KBUILD_ARFLAGS)
  cmd_make_empty_builtin = rm -f $@; $(AR) rcSTP$(KBUILD_ARFLAGS)

  cmd_link_o_target = $(if $(strip $(obj-y)),\
                $(cmd_make_builtin) $@ $(filter $(obj-y), $^) \
                $(cmd_secanalysis),\
                $(cmd_make_empty_builtin) $@)

  $(builtin-target): $(obj-y) FORCE
      $(call if_changed,link_o_target)
  ```

  这个目标依赖于 `$(obj-y)` 目标，而 `obj-y` 是当前文件夹下需要被构建的全部对象文件的列表。当这些文件全部准备好之后， `cmd_link_o_target` 命令就会被执行。如果调用 `cmd_make_empty_builtin` 时 `obj-y` 变量为空，那么就只创建一个空的 `built-in.o` 文件。此外， `cmd_make_builtin` 命令也被执行；它使用我们熟悉的 `ar` 工具来创建 `built-in.o` thin archive。

* 最终我们来到了需要编译的地方。你还记得我们前面还没研究的依赖项是 `$(obj-y)`， `obj-y` 就是一个对象文件列表。这个目标将定义在 [这里](https://github.com/torvalds/linux/blob/v4.14/scripts/Makefile.build#L313) 的全部 `.c` 文件编译成对应的对象文件。我们来看一下理解这个目标所需要用到的全部代码。

  ```
  cmd_cc_o_c = $(CC) $(c_flags) -c -o $@ $<

  define rule_cc_o_c
      $(call echo-cmd,checksrc) $(cmd_checksrc)              \
      $(call cmd_and_fixdep,cc_o_c)                      \
      $(cmd_modversions_c)                          \
      $(call echo-cmd,objtool) $(cmd_objtool)                  \
      $(call echo-cmd,record_mcount) $(cmd_record_mcount)
  endef

  $(obj)/%.o: $(src)/%.c $(recordmcount_source) $(objtool_dep) FORCE
      $(call cmd,force_checksrc)
      $(call if_changed_rule,cc_o_c)
  ```

  这脚本命令中这个目标调用了 `rule_cc_o_c`。这条规则负责很多工作，比如检查源代码中的一些常见错误(`cmd_checksrc`)，启用导出模块符号的版本控制(`cmd_modversions_c`)，使用 [objtool](https://github.com/torvalds/linux/tree/v4.14/tools/objtool) 验证生成的对象文件的某些方面，以及构造一个对 `mcount` 函数的调用列表以便于 [ftrace](https://github.com/torvalds/linux/blob/v4.14/Documentation/trace/ftrace.txt) 可以快速找到它们。 不过最重要的就是它调用了 `cmd_cc_o_c` 命令来将全部 `.c` 文件编译成对象文件。 

### 结论

还真是在内核构建系统内的一次漫长之旅! 当然，我们跳过了许多的细节，对于那些想学习更多关于这一块内容的人，我推荐你们去读这篇 [文档](https://github.com/torvalds/linux/blob/v4.14/Documentation/kbuild/makefiles.txt)，然后继续读 Linux makefile 的源代码。现在让我们来巩固一下几个重要的知识点，这些知识点也是你应该从本章中学到的核心内容。

1. `.c` 是如何编译成对象文件的。
1. 对象文件是如何合并成 `build-in.o` 文件的。
1. 如何递归构建获取全部的 `build-in.o` 子文件并将他们合并成一个。
1. How `vmlinux` is linked from all top-level `build-in.o` files.

我们的主要目标是通过学习本章，你能对以上知识点有一个大概的认识。

##### 上一页

1.2 [内核初始化： Linux 项目结构](../../../docs/lesson01/linux/project-structure.md)

##### 下一页

1.4 [内核初始化： Linux 启动流程](../../../docs/lesson01/linux/kernel-startup.md)
