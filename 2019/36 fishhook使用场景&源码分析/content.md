## 不做伪学习者

上一篇我们一起分析了 **fishhook的实现原理**，但很多东西如果我们仅仅知道原理，其实距离真正吸收它并将其转化成自己的生产力还有很长的路。你得弄清楚别人是怎么利用这个原理去解决问题的，还要借鉴别人的设计思想，再结合我们自己的思考不断地实践和总结，才能真正让知识成为自己的生产力。

话不多说，进入今天的第一个正题。

## fishhook 使用场景

在上一篇里已经为大家演示了它的基本用法，使用很简单，这里就不展开了。它的使用场景正如其名: fishhook，主要用在安全防护领域。当然，大神级的逆向与安全防护专家咱们先不谈，那个级别的高手我相信也不会看到这篇文章，天下没有绝对的安全，黑与白永远都在博弈，所以希望大家不要钻牛角尖，至少咱们不能写出让菜鸟逆向就能轻松搞定的应用对吧？当然，后面咱们也会学习静态分析和汇编的知识，掌握更高级的逆向和防护技能，那都是打好基础的后话了。今天咱们的重点是源码分析，顺便温习下 c 的数据结构。

下面先来了解一下用 fishhook 防 HOOK 的基本思路：

1. 在基础的动态调试逆向中，最常见的就是定位到目标方法后，通过 runtime 中的几种方法交换方式，实现代码注入的目的。为你准备好了相关的文章：iOS代码注入+HOOK微信登录
2. 既然 fishhook 可以拦截系统外部的 C 函数，那自然就可以 HOOK 到 runtime 库中的所有方法。
3. 那我们就将所有可能用来篡改我们 OC 方法实现的 runtime API，都用 fishhook 拦截掉，使其无法用代码注入的方式成功 HOOK。

思路理清了，撸起袖子开始干。

* 为了方便演示，这里直接搞了一个分类，将 ViewController 的实例方法 `viewDidAppear:` 用 `method_exchangeImplementations(Method _Nonnull m1, Method _Nonnull m2)` 的方式与 `my_viewDidAppear`: 交换了实现，上代码：

![1](http://)

这时我们程序跑起来就可以看到如下输出：

![2](http://)

* 为了阻止其完成方法交换，我们要 hook `method_exchangeImplementations` 方法，拖入 fishhook 源文件，再添加一个分类并写好 hook `method_exchangeImplementations` 的代码：（如果成功 hook 了 `method_exchangeImplementations` ，那别人调用该方法时会进入我们的 `myExchange` ，然后顺便又把 `NSLog` hook 了一下，不要被这个绕晕了 😜）

![3](http://)

再次 Run 起来，咦？肿么肥四？你会发现 `method_exchangeImplementations` 并没有 HOOK 成功， `viewDidAppear:` 依然被篡改了实现，问题出在哪了呢？

对，聪明的你一定发现了问题所在：**是代码执行顺序的问题**

经过实践，我发现项目里参与编译的文件顺序就是其编译后被加载时的载入顺序（暂未找到官方的编译顺序说明，还请有研究的大佬指点），即此时 `ViewController+HOOKTest` 的 `load` 方法会早于 `ViewController+FishHook` 的 `load` 调用，所以 `method_exchangeImplementations` 的实现被我们 HOOK 发生在 `viewDidAppear:` 被别人交换之后，从而导致防护的失败：

![4](http://)

验证一下我们的想法：

![5](http://)

如上图所示，在调整了编译文件的顺序之后成功 HOOK 到了 `method_exchangeImplementations` 的调用，但实际开发中我们不可能采用这么笨的方法，也不可能通过这种方式决定文件的载入顺序，因此我们要想办法保证 fishhook 的代码必须最先执行才行。

那如何做到呢？由此前的 **dyld背后的故事&源码分析** 可以得知，本地的Framework中的类一定会早于后注入的库（动态库例外，非越狱设备是没有插入动态库的权限的）和可执行文件中的类进行初始化。所以我们将 fishhook 的 HOOK 操作代码移到自建的 Framework 中即可：

![6](http://)

至此，我们已经知道了 fishhook 反调试的基本思路，当然，上面的代码只是思路演示，实际开发中，像 `method_getImplementation`、`method_setImplementation` 等函数都需要用同样的方式一一 HOOK，同时，如果自己的项目中已经用到了这些函数，还需要设计相应的白名单方案，并且在检测到是被三方非法 HOOK 时通常直接调用 exit(0) 这类接口终止掉进程。这些细节以后还会详细讲，这里算是抛砖引玉吧。
 
那咱们进入第二个正题，源码分析。
 
## fishhook 源码分析

（一）： 在写 fishhook 的代码时，第一件事就是声明一个 `rebinding` 类型的结构体变量，其源码如下：

![7](http://)

命名很易读，其中第三个成员 `void **replaced` 是指向指针的指针，可以理解为一个存着另一个指针地址的指针，在上述示例中， `*replaced` 取出的就是一个指向共享库中 `method_exchangeImplementations` 函数实现的指针，再对其取值，`**replaced` 得到的就是共享库中 `method_exchangeImplementations` 函数实现的首地址，还不清楚的同学要自己去补补基础了😝。

（二）： 按结构体成员的类型写好声明和实现之后，一一赋值给结构体对应的成员，再把这些结构体放到一个数组中，然后调用重绑定符号函数 rebind_symbols(如果绑定成功返回 0，否则返回 -1)，并将结构体数组和数组长度作为参数传入：

![8](http://)

接下来咱们一步步的分析在这个函数里具体都做了些啥：

1) 第一件事，调用了这个函数-- prepend_rebindings，其具体实现如下：

![9](http://)

咦？传入的第一个参数 &_rebindings_head 是个啥东东？

![10](http://)

看源码：`_rebindings_head` 被声明为一个指向 `rebindings_entry` 类型结构体的静态指针变量，那 `&_rebindings_head` 就是取出这个指针的地址，再看该函数的参数声明 `struct rebindings_entry **` ，没错，这又是一个指向指针的指针。

结构体 `rebindings_entry` 的三个成员分别是：指向 `rebinding` 类型结构体的指针（用来指向传入结构体数组的首元素地址）、`rebindings_nel`：记录此次要重绑定的数量（用于开辟对应大小的空间）、指向下一个 `rebindings_entry` 类型的结构体（记录下一次需要重绑定的数据），这就是典型的数据结构——链表的一种实现。_rebindings_head 就是指向该链表的指针。

为了加深理解，我为你画了一张 prepend_rebindings 函数的作用示意图：

![11](http://)

一句话总结 `prepend_rebindings` 函数的目的：**将新加入的 rebindings 数组不断的添加到 _rebindings_head 这个链表的头部成为新的头节点。**

2) 第二件事，对已经载入的镜像文件（也就是库）逐一查找目标符号进行 hook。前面我们已经知道 fishhook 的代码执行时间非常早，所以第一次执行时要 hook 的库可能还没完成装载，因此这里如果是第一次调用会通过一个函数对库的装载完成注册监听和回调的方法：

```c
extern void _dyld_register_func_for_add_image(void (*func)(const struct mach_header* mh, intptr_t vmaddr_slide))    __OSX_AVAILABLE_STARTING(__MAC_10_1, __IPHONE_2_0);
```

源码注释如下图：

![12](http://)

当回调到 `_rebind_symbols_for_image` 时，会将存着待绑定函数信息的链表作为参数传入，用于符号查找和函数指针的交换，第二个参数 `header` 是当前 `image` 的头信息，第三个参数 `slide` 是 ASLR 的偏移：

![13](http://)

3) 第三件事，根据 fishhook 是如何根据字符串找到对应指针在符号表中的偏移值的 中的几个步骤，去找到目标符号所在的符号表以及偏移值：

![14](http://)

这个过程比较枯燥，无非就是按照规则计算各种表的地址和指针在表中的偏移量。

4) 最后一件事，根据算好的符号表地址和偏移量，找到在符号表中用于指向共享库目标函数的指针，然后将该指针的值（即目标函数的地址）赋值给我们的 `*replaced`，最后修改该指针的值为我们的 `replacement`（新的函数地址），`perform_rebinding_with_section` 的源码实现：

![15](http://)

## fishhook 源码之旅，告一段落

如果不算注释，fishhook 的源码实现一共 180+ 行，通过对其源码的分析，如果做到读懂它的每一行，我相信不管是对指针的理解和使用，还是对链表的数据结构和实现方式，你都会有更好的理解。当然，你对 MachO 的文件结构和加载机制，也更加了然于胸，同时还 get 了基本的安全防护技巧。总之，愿你不虚此行。

