在之前这篇 *iOS应用脚本重签*<sup>[1]</sup> 名中，我们对脱壳的微信安装包进行重签名，并成功在真机上运行起来，完成了iOS逆向的准备工作。这一篇我们将通过演示如何HOOK微信登录事件并获取到用户密码，把iOS代码注入的几种方式串起来做个简单地概述。不管做逆向还是正向开发，这些都能为你提供一些在应用安全攻防方面的思路。

当拿到别人的脱壳包，想要HOOK别人的方法做些小插件，首先需要程序执行你写的代码，你才有机会利用runtime的运行时机制去做自己的事情，关于方法混淆的注意事项请参考 *这一篇*<sup>[2]</sup>。让程序执行我们写的代码就需要修改MachO文件，关于MachO我在 *这一篇*<sup>[3]</sup> 里详细讲解了，这篇主要讲代码注入的事儿：


## Framework注入

添加自己的Framework：

![1](http://)

写好测试代码，在 *上一篇*<sup>[1]</sup> 重签名脚本的基础上加一行修改MachO加载路径的代码：

```
yololib "$TARGET_APP_PATH/$APP_BINARY" "Frameworks/SharonFramework.framework/SharonFramework"
```

Framework文件名为你自己刚刚添加的。直接Run！

![2](http://)

大功告成。

## Dylib注入

添加自己的Dylib：

![3](http://)

要注意的是这样添加的MacOS的Dylib需要将BuildSetting-->Base SDK改为iOS，BuildSetting-->CODE SIGN IDENTITY改为iPhone Developer即可在iPhone上运行。

![4](http://)

另外，与Framework不同的是它需要手动添加关联库：

同样在重签名脚本中加一行修改MachO可执行文件路径的代码：

```
yololib "$TARGET_APP_PATH/$APP_BINARY" "Frameworks/libSharonLibrary.dylib"
```

dylib文件名为你自己刚刚添加的。直接Run！

![4](http://)

至此我们已经完成了代码注入的第一步，让别人的应用在运行时执行我们写的代码，这个过程中你可能会碰到签名不成功等各种各样的奇葩问题，不要慌，静下心分析，实在不行你可以留言^_^ ~，接下来我们要尝试HOOK微信的登录按钮事件。

同步几个共识：

* +load 方法的调用发生在类或分类被 runtime 加载（编译后的可执行文件被装载到内存中）时，只调用1次。
* 子类的 +load 方法会在它的所有父类的 +load 方法之后执行，而分类的 +load 方法会在它的主类的 +load 方法之后执行。
* 如果子类没有实现 +load 方法，那么当它被加载时 runtime 是不会去调用父类的 +load 方法的。同理，当一个类和它的分类都实现了 +load 方法时，两个方法都会被调用。
* 不同的类之间的 +load 方法的调用顺序是不确定的。
* 基于+load方法的上述特点，它是实现方法混淆的最佳入口。

## 通过viewDebug+头文件分析目标Method

![5](http://)

如上图所示，我们很快定位到登录按钮的target为WCAccountLoginControlLogic，action为onFirstViewLogin，我们在通过头文件分析一下，class-dump怎么用相信你Google一下就搞得定，这里就不赘述啦，拿到微信的所有头文件丢到sublime里全局搜索：

![6](http://)

果然，找到了目标文件，点击进入头文件查看Method列表：

![7](http://)

验证了我们的分析是正确的。
用同样的方式我们定位账号密码输入页登录按钮的target为WCAccountMainLoginViewController，action为onNext：

![8](http://)

我们将通过HOOK登录按钮点击事件获取密码输入框里的内容。

## MethodSwizzling的几种姿势

**class_replaceMethod**

![9](http://)

class_replaceMethod本身会尝试调用class_addMethod和method_setImplementation，所以直接调用class_replaceMethod就可以了。

**class_getInstanceMethod & method_setImplementation**

![10](http://)

**method_exchangeImplementations**

心细的同学一定会发现，在这个场景下，如果直接写个OC方法然后用method_exchangeImplementations交换新旧方法的实现有问题：

![11](http://)

因为my_next中的self是WCAccountMainLoginViewController，调用my_next会找不到方法。解决方案是手动为WCAccountMainLoginViewController添加my_next方法。

![12](http://)

由此我们也发现，method_exchangeImplementations在分类或子类中对主/父类重载的方法进行交换时更方便些，不会出现上述问题。所以在逆向中一般不直接使用method_exchangeImplementations，更倾向于前两种方式。

### 参考

1. https://juejin.im/post/5c460908f265da6167209b87
2. https://juejin.im/entry/5a1fceddf265da43310d9985
3. https://juejin.im/post/5c67e7efe51d45164c75993b

