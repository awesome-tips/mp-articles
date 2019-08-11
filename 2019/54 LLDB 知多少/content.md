## 重识 LLDB

### LLDB 是什么？

“如果调试是删除 bug 的过程，那么编程就是引入 bug 的过程。”（Edsger W. Dijkstra）

对于苹果开发者而言，LLDB 是无人不知的调试工具，然而此知非彼知，相信有相当规模的开发者对 LLDB 的了解仍然停留于几个基础命令的使用，今天让我们来重新认识一下既熟悉又陌生的 LLDB，看看它那些你不曾用过的强大功能，以及如何提高我们的开发效率。

开始把玩其功能之前，先搞清楚 LLDB 是什么，简言之，LLDB 是一个有着 REPL 的特性和 C++ 、Python 插件的开源调试器。

> LLDB is a next generation, high-performance debugger. It is built as a set of reusable components which highly leverage existing libraries in the larger LLVM Project, such as the Clang expression parser and LLVM disassembler.
> LLDB is the default debugger in Xcode on Mac OS X and supports debugging C, Objective-C and C++ on the desktop and iOS devices and simulator.

以上摘自官方文档中的一段简短的介绍，更多相关信息请参阅 LLDB **官方文档**。

## LLDB 命令结构
 
知道了 LLDB 是什么，还需要了解其命令结构及语法，这样才能告别死记命令，开启压榨 LLDB 之路了。LLDB 通用结构的形式如下：


```sh
<command> [<subcommand> [<subcommand>...]] <action> [-options [option-value]] [argument [argument...]]
```

其中：

* command、subcommand：LLDB调试命令的名称。命令和子命令按层级结构来排列：一个命令对象为跟随其的子命令对象创建一个上下文，子命令又为其子命令创建一个上下文，依此类推。
* action：命令操作，想在前面的命令序列的上下文中执行的一些操作。
* options：命令选项，行为修改器(action modifiers)。通常带有一些值。
* argument：命令参数，根据使用的命令的上下文来表示各种不同的东西。
* []：表示命令是可选的，可以有也可以没有。

举个例子：

命令：`breakpoint set -n main` 对应到上面的语法就是：

* command：breakpoint 断点命令
* action：set 设置断点
* option：-n 表根据方法 name 设置断点
* arguement：main 表示方法名为 main

关于**原始命令**：

LLDB支持不带命令选项的原始命令，原始命令会将命令后面的所有东西当做参数(arguement)处理。但很多原始命令也可以带命令选项，当你使用命令选项的时候，需要在命令选项后面加 `--` 区分命令选项和参数。
 
如：`expression` （就是 `p/print/call`）、`expression -o`（就是 po），打印一个 UIView 对象地址：

![1](http://)

前者是计算其地址的值，后者调用了对象的discraption方法，其中体现了唯一匹配原则：假如根据前n个字母已经能唯一匹配到某个命令，则只写前n个字母等效于写下完整的命令。再用设置断点的命令举例，下面两条命令等效：

![2](http://)

更多命令结构的介绍及用法，请参考 `LLDB 入门`。接下来介绍点 LLDB 最实用又最容易被忽略的用法。

## LLDB 常用命令总结

### 辅助记忆：apropos

当我们并不能完全记得某个命令的时候，使用 **apropos** 通过命令中的某个关键字就可以找到所有相关的命令信息。

比如: 我们想使用stop-hook的命令，但是已经不记得stop-hook命令是啥样了：

![3](http://)

实在记不起来也不要慌，这里有 `命令list` https://lldb.llvm.org/lldb-gdb.html。

### 一.断点设置
 
关于断点设置，多数人都习惯用图形界面去做，但在调试中有些场景仅仅靠图形界面还是不够的，比如：如何通过断点实现类似 KVO 那样对成员变量变化的监听呢？（别跟我说你要加代码重写set方法...即使这样也不靠谱）。下面一一罗列那些好用的断点命令：

* breakpoint list：查看所有断点列表
* breakpoint delete：删除所有断点（可跟组号删除指定组）
* breakpoint disable/enable：禁用 启用指定断点
* breakpoint set -r some：遍历整个项目中包含 some 这个字符的所有方法并设置断点
* breakpoint 支持按文件名、函数名、行数、正则等各种条件筛选设置断点，请结合语法并参考官方文档
* watchpoint set expression 0x10cc64d50：在内存中为地址为0x10cc64d50的对象设置内存断点
* watchpoint set variable xxoo：为当前对象的变量 xxoo 设置内存断点
* target stop-hook add -o "frame variable"：添加每次程序 stop 时都希望执行的命令：frame variable（打印当前栈内的所有变量）
* target stop-hook、watchpoint 的增删改查命令与 breakpoint 的基本相同
* 更多变态断点玩法需自定义插件支持，迫不及待的你请快进此文 》》》

### 二.流程控制

这两幅图你一定不陌生：

![4](http://)

![5](http://)

* 第一个按钮：continue/c 继续执行
* 第二个按钮：
	* 图一： thread step-over/next/n 当前线程下一步（以一个完整子函数为一步）
	* 图二：  thread step-inst-over/ni 当前线程下一步（以一个汇编函数为一步）
* 第三个按钮：
	* 图一： thread step-in/step/s 当前线程下一步（遇到子函数就进入并且继续单步执行）
	* 图二： thread step-inst-over/si 当前线程下一步（遇到汇编函数就进入并且继续单步执行汇编指令）
* 第四个按钮：thread step-out/finish 退出当前帧栈

其他命令：

* thread return：它有一个可选参数，在执行时它会把可选参数加载进返回寄存器里，然后立刻执行返回命令，跳出当前栈帧。这意味这函数剩余的部分不会被执行。这会给 ARC 的引用计数造成一些问题，或者会使函数内的清理部分失效。但是在函数的开头执行这个命令，是个非常好的隔离次函数、伪造返回值的方式。

### 三.可执行文件&共享库查询命令

这些命令在逆向及定位时使用频率非常高。

1. image list：列出主要的可执行文件和所有依赖的共享库。
2. image lookup --address 0x1ec4：在可执行文件或任何共享库中查找原始地址信息。
3. image lookup -v --address 0x1ec4：查找完整的源代码行信息。
4. image lookup --type NSString：根据名称查找对应（NSString）类型的信息。

### 四.其他常用命令模板

1. register read：显示当前线程的通用寄存器。
2. register write rax 123：将一个新的十进制值“123”写入当前线程寄存器“rax”。
3. memory read --size 4 --format x --count 4 0xbffff3c0：从地址0xbffff3c0读取内存，并显示4个十六进制uint32_t值。

## 加强版 LLDB —— 修改 .lldbinit 文件 & 插件安装

前面所列的命令在 这里 都能找到官方说明，更多命令用法有兴趣的建议自己去细细探索，接下来我们要学会站在巨人的肩膀上，用 高手们专门为 LLDB 写的插件去深入挖掘它的潜力。

**推荐插件一： facebook 开源的 LLDB 插件 chisel**

`brew install chisel` 的安装过程这里就不赘述了，安装成功后，在~/目录下的 `.lldbinit` 文件中引入对应文件路径，增加一行：`command script import /usr/local/opt/chisel/libexec/fblldb.py` 后保存， 重启 Xcode即可使用。它提供的快捷 命令清单及说明 这里也不赘述了。截个图感受下它的强大吧：

![6](http://)

**推荐插件二：DerekSelander/LLDB**

该插件与 chisel 都是用 Python 写的，其安装需要手动下载仓库，然后将仓库中 `dslldb.py` 文件的路径用与上述同样的方式添加到  `.lldbinit` 中，具体用法也很简单粗暴，就不在这粘贴了，请至 README 领略。

## 总结：

看到这，你收获的只有暂时记忆，其实等于毫无所获...而且还浪费了宝贵的几分钟，这也是我极不希望看到的，而避免其成为事实的唯一方式就是，请你打开 Xcode，运行一个项目，参照着文中涉及到的说明文档，试着敲一敲每个命令，体会一下它们的用法与区别。最后，为你的 LLDB 配好插件，然后去感受它的蜕变，相信我，你的开发效率提升的可不止一点点。学习是一种能力，拒绝操作手册式灌输，分享者多半是在总结学习收获时为读者提供一些思路或方向，这也是我为你保留一丝探索余地的初衷，愿你有所收获。

### 参考

* 官方文档 http://lldb.llvm.org/index.html



