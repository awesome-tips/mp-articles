| 原文链接：https://juejin.im/post/5c78906df265da2de97092ad

昨天朋友圈被一篇文章（以下简称“coobjc介绍文章”）刷屏了：**刚刚，阿里开源 iOS 协程开发框架 coobjc！**(https://mp.weixin.qq.com/s/hXmkd0TqTrCD-4kYlZcqvQ)。可能大部分 iOS 开发者都直接懵逼了：

* 什么是协程？
* 协程的作用是什么？
* 为什么要使用它？

因此笔者想给大家普及普及协程的知识，运行一下 coobjc 的 Example，顺便分析一下 coobjc 源码。

## 分析

协程的维基百科在这里：协程。引用里面的解释如下：

> 协程是计算机程序的一类组件，推广了非抢先多任务的子程序，允许执行被挂起与被恢复。相对子例程而言，协程更为一般和灵活，但在实践中使用没有子例程那样广泛。协程源自 Simula 和 Modula-2 语言，但也有其他语言支持。协程更适合于用来实现彼此熟悉的程序组件，如合作式多任务、异常处理、事件循环、迭代器、无限列表和管道。根据高德纳的说法, 马尔文·康威于1958年发明了术语 coroutine 并用于构建汇编程序。

对，还是一知半解。但最起码我们了解到

* 协程的英文是“coroutine”，因此我们能理解阿里的库起名为 `coobjc` 的含义。那么这个词又是怎么来的呢？笔者再深挖一下，协程(coroutine)顾名思义就是“协作的例程”（co-operative routines）。
* 协程是和进程或者线程有一定关系的
* 协程的历史还是比较悠久的，只是 Objective-C 不支持。笔者经过查阅，发现很多现代语言都支持协程。比如 Python 以及 swift，甚至C语言也是支持协程的。

协程的作用其实在 coobjc 介绍文章中有提及，是为了优化 iOS 中的异步操作。解决了如下问题：

* "嵌套地狱"
* 错误处理复杂和冗长
* 容易忘记调用 completion handler
* 条件执行变得很困难
* 从互相独立的调用中组合返回结果变得极其困难
* 在错误的线程中继续执行
* 难以定位原因的多线程崩溃
* 锁和信号量滥用带来的卡顿、卡死

听起来是有点强大，最明显的好处是可以简化代码；并且在 coobjc 介绍文章也说道，性能也有所保障：当线程的数量级大于1000以上时，coobjc 的优势就会非常明显。为了证明文章的结论，我们就来运行一下 coobjc 源码好了。这里（https://github.com/alibaba/coobjc）下载 coobjc 源码。发现目录结构如下：

![1](http://)

从目录结构看还是比较清晰的，根据 coobjc 介绍文章中提到的，coobjc 不但提供了基础的异步操作还提供了基于 UIKit 的封装。目录中

* `cokit` 及其子目录提供的是基于UIKit层的coobjc封装
* `coobjc` 目录是coobjc的Objective-C版实现的源代码
* `coswift` 目录是coobjc的Swift版实现的源代码
* `Example` 下有两个目录，一个是Objective-C的实现，一个是Swift版的实现的Demo

我们先分析一下 coobjcBaseExample 工程：打开项目，pod update 一下即可运行,运行结果如下：

![2](http://)

可以看到是个简单的列表页。

> Tips：打开 podfile 可以发现里面有库 coobjc 以外，还有 Specta、Expecta 以及 OCMock。这三个库这里不多做介绍了，大家只需要知道这是用于单元测试的。

我们先看一下这个列表的实现逻辑是什么样的。我们不难定位到页面位于 KMDiscoverListViewController 中，其网络请求（这里是电影列表）代码如下：

```objc
- (void)requestMovies
{
    co_launch(^{
        NSArray *dataArray = [[KMDiscoverSource discoverSource] getDiscoverList:@"1"];
        [self.refreshControl endRefreshing];
        
        if (dataArray != nil)
        {
            [self processData:dataArray];
        }
        else
        {
            [self.networkLoadingViewController showErrorView];
        }
    });
}
```

这里很容易理解代码

```objc
NSArray *dataArray = [[KMDiscoverSource discoverSource] getDiscoverList:@"1"];
```

是请求网络数据的，其实现如下：

```objc
- (NSArray*)getDiscoverList:(NSString *)pageLimit;
{
    NSString *url = [NSString stringWithFormat:@"%@&page=%@", [self prepareUrl], pageLimit];
    id json = [[DataService sharedInstance] requestJSONWithURL:url];
    NSDictionary* infosDictionary = [self dictionaryFromResponseObject:json jsonPatternFile:@"KMDiscoverSourceJsonPattern.json"];
    return [self processResponseObject:infosDictionary];
}
```

以上代码也能猜出，

```objc
id json = [[DataService sharedInstance] requestJSONWithURL:url];
```

这一行是做了网络请求，但是我们再点击进入类 DataService 看 requestJSONWithURL 方法的实现的时候，发现已经看不懂了：

```objc
- (id)requestJSONWithURL:(NSString*)url CO_ASYNC{
    SURE_ASYNC
    return await([self.jsonActor sendMessage:url]);
}
```

好吧。既然看不懂了，我们就从头开始学习，协程的含义以及使用。继而对 coobjc 源码进行分析。

## 协程入门

coobjc介绍文章中有提到

* 第一种：利用 glibc 的 ucontext组件(云风的库)。
* 第二种：使用汇编代码来切换上下文(实现C协程)，原理同 ucontext。
* 第三种：利用 C 语言语法 switch-case 的奇淫技巧来实现（Protothreads)。
* 第四种：利用了 C 语言的 setjmp 和 longjmp。
* 第五种：利用编译器支持语法糖。

经过筛选最终选择了第二种。那我们来一个个分析，为什么 coobjc 摒弃了其他的方式。首先我们看第一种，coobjc 介绍文章中提到 ucontext 在 iOS 中被废弃了，那如果不废弃，我们如何去使用 ucontext 呢？如下的一个 Demo 可以解释一下 ucontext 的用法：

```objc
#include <stdio.h>
#include <ucontext.h>
#include <unistd.h>
 
int main(int argc, const char *argv[]){
    ucontext_t context;
    getcontext(&context);
    puts("Hello world");
    sleep(1);
    setcontext(&context);
    return 0;
}
```

> 注：示例代码来自维基百科.

保存上述代码到 example.c，执行编译命令：

```sh
gcc example.c -o example
```

想想程序运行的结果会是什么样？

```sh
kysonzhu@ubuntu:~$ ./example 
Hello world
Hello world
Hello world
Hello world
Hello world
Hello world
^C
kysonzhu@ubuntu:~$
```

上面是程序执行的部分输出，不知道是否和你想得一样呢？我们可以看到，程序在输出第一个 “Hello world" 后并没有退出程序，而是持续不断的输出 “Hello world”。其实是程序通过 getcontext 先保存了一个上下文，然后输出 “Hello world”，在通过 setcontext 恢复到 getcontext 的地方，重新执行代码，所以导致程序不断的输出 “Hello world”，在我这个菜鸟的眼里，这简直就是一个神奇的跳转。那么问题来了，ucontext 到底是什么？

这里笔者不多做介绍了，推荐一篇文章，讲的比较详细：**ucontext-人人都可以实现的简单协程库**。这里我们只需要知道，所谓 coobjc 介绍文章中提到的使用汇编语言模拟 ucontext，其实就是模拟的上面例子中的 setcontext 及 getcontext 等函数。为了证明笔者的猜想，笔者打开了coobjc源码库，发现里面的唯一的汇编文件 coroutine_context.s

![3](http://)

查看该文件，发现了这么几个函数：

* _coroutine_getcontext
* _coroutine_begin
* _coroutine_setcontext

果然验证了笔者的想法。这三个方法被暴露在文件 coroutine_context.h 中，供后序调用：

```objc
extern int coroutine_getcontext (coroutine_ucontext_t *__ucp);
extern int coroutine_setcontext (coroutine_ucontext_t *__ucp);
extern int coroutine_begin (coroutine_ucontext_t *__ucp);
```

接下来说另外一个函数

```c
int  setcontext(const ucontext_t *cut)
```

该函数是设置当前的上下文为 cut，setcontext 的上下文 cut 应该通过 getcontext 或者 makecontext 取得，如果调用成功则不返回。如果上下文是通过调用 getcontext() 取得，程序会继续执行这个调用。如果上下文是通过调用makecontext取得，程序会调用 makecontext 函数的第二个参数指向的函数，如果 func 函数返回，则恢复 makecontext 第一个参数指向的上下文第一个参数指向的上下文 context_t 中指向的 uc_link。如果 uc_link 为 NULL，则线程退出。

我们画个表类比一下 ucontext 和 coobjc 的函数：

![4](http://)

这么一来，我们之前的程序可以改写成如下：

```c
#import <coobjc/coroutine_context.h>
int main(int argc, const char *argv[]) {
    coroutine_ucontext_t context;
    coroutine_getcontext(&context);
    puts("Hello world");
    sleep(1);
    coroutine_setcontext(&context);
    return 0;
}
```

返回的结果仍然不变，一直打印“hello world”。

## 深入协程

**(1)目录分析**

![5](http://)

上图是 coobjc 的目录结构，其中

* **core 目录** 提供了核心的协程函数
* **api目录** 是 coobjc 基于 Objective-C 的封装
* **csp 目录** 从库 libtask 引入，提供了一些链式操作
* **objc** 提供了 coobjc 对象声明周期管理的一些类

下面的文章，笔者会先从核心的 core 目录开始研究，后面的大家理解起来也就不复杂了。

**(2)协程的构成**

上面我们只简单的介绍了 coobjc，也了解到 coobjc 基本都是参考了 ucontext。那下面的例子中，笔者尽可能先介绍 ucontext，然后再应用到 coobjc 对应的方法中。

我们继续讨论上文提到的几个函数，并说明一下其作用：

```c
int  getcontext(ucontext_t *uctp)
```

这个方法是，获取当前上下文，并将上下文设置到 uctp 中，uctp 是个上下文结构体，其定义如下：

```c
_STRUCT_UCONTEXT
{
	int                     uc_onstack;
	__darwin_sigset_t       uc_sigmask;     /* signal mask used by this context */
	_STRUCT_SIGALTSTACK     uc_stack;       /* stack used by this context */
	_STRUCT_UCONTEXT        *uc_link;       /* pointer to resuming context */
	__darwin_size_t	        uc_mcsize;      /* size of the machine context passed in */
	_STRUCT_MCONTEXT        *uc_mcontext;   /* pointer to machine specific context */
#ifdef _XOPEN_SOURCE
	_STRUCT_MCONTEXT        __mcontext_data;
#endif /* _XOPEN_SOURCE */
};

/* user context */
typedef _STRUCT_UCONTEXT	ucontext_t;     /* [???] user context */
```	

以上是 ucontext 的数据结构，其内部的几个属性介绍一下：

* 当当前上下文(如使用 makecontext 创建的上下文）运行终止时系统会恢复 uc_link 指向的上下文；
* uc_sigmask 为该上下文中的阻塞信号集合；
* uc_stack 为该上下文中使用的栈；
* uc_mcontext 保存的上下文的特定机器表示，包括调用线程的特定寄存器等。

其实还蛮好理解的，ucontext 其实就存放一些必要的数据，这些数据还包括拯救成功或者失败的情况需要的数据。

相比较而言，coobjc 的定义和 ucontext 有一定区别：

```c
/**
     The structure store coroutine's context data.
     */
struct coroutine {
    coroutine_func entry;                   // Process entry.
    void *userdata;                         // Userdata.
    coroutine_func userdata_dispose;        // Userdata's dispose action.
    void *context;                          // Coroutine's Call stack data.
    void *pre_context;                      // Coroutine's source process's Call stack data.
    int status;                             // Coroutine's running status.
    uint32_t stack_size;                    // Coroutine's stack size
    void *stack_memory;                     // Coroutine's stack memory address.
    void *stack_top;                    // Coroutine's stack top address.
    struct coroutine_scheduler *scheduler;  // The pointer to the scheduler.
    int8_t   is_scheduler;                  // The coroutine is a scheduler.
    
    struct coroutine *prev;
    struct coroutine *next;
    
    void *autoreleasepage;                  // If enable autorelease, the custom autoreleasepage.
    bool is_cancelled;                      // The coroutine is cancelled
};
typedef struct coroutine coroutine_t;
```

其中

```c
    struct coroutine *prev;
    struct coroutine *next;
```

表明其是一个链表结构。既然是链表，那么就会有添加元素，以及删除某个元素的方法，果然我们在 coroutine.m 中发现了对应的链表操作方法：

```c
// add routine to the queue
void scheduler_add_coroutine(coroutine_list_t *l, coroutine_t *t) {
    if(l->tail) {
        l->tail->next = t;
        t->prev = l->tail;
    } else {
        l->head = t;
        t->prev = nil;
    }
    l->tail = t;
    t->next = nil;
}

// delete routine from the queue
void scheduler_delete_coroutine(coroutine_list_t *l, coroutine_t *t) {
    if(t->prev) {
        t->prev->next = t->next;
    } else {
        l->head = t->next;
    }
    
    if(t->next) {
        t->next->prev = t->prev;
    } else {
        l->tail = t->prev;
    }
}
```

其中 coroutine_list_t 是为了标识链表的头尾节点：

```c
/**
 Define the linked list of scheduler's queue.
 */
struct coroutine_list {
    coroutine_t *head;
    coroutine_t *tail;
};
typedef struct coroutine_list coroutine_list_t;
```

为了管理所有的协程状态，还设置了一个调度器：

```c
/**
 Define the scheduler.
 One thread own one scheduler, all coroutine run this thread shares it.
 */
struct coroutine_scheduler {
    coroutine_t         *main_coroutine;
    coroutine_t         *running_coroutine;
    coroutine_list_t     coroutine_queue;
};
typedef struct coroutine_scheduler coroutine_scheduler_t;
```

看命名就大概能猜到，main_coroutine 中包含了主协程（可能是即将设置数据的协程，或者即将使用的协程）；running_coroutine 是当前正在运行的协程。

**(3)协程的操作**

协程拥有和线程一样类似的操作，例如创建，启动，出让控制权，恢复，以及死亡。对应的，我们在 coroutine.h 看到了如下的几个函数声明：

```c
//关闭一个协程如果它已经死亡
void coroutine_close_ifdead(coroutine_t *co);
//添加协程到调度器，并且立刻启动
void coroutine_resume(coroutine_t *co);
//添加协程到调度器
void coroutine_add(coroutine_t *co);
//出让控制权
void coroutine_yield(coroutine_t *co);
```

为了更好的控制各个操作中的数据，coobjc 还提供了以下两个方法：

```c
void coroutine_setuserdata(coroutine_t *co, void *userdata, coroutine_func userdata_dispose);
void *coroutine_getuserdata(coroutine_t *co);
```

至此，coobjc 的核心代码都分析完成了。

**(4)协程的Objective-C层面的封装**

我们再次回到文章开头的例子 `- (void)requestMovies` 方法的实现中，第一步就是调用一个 co_launch() 的方法，这个方法最终会调用到

```c
+ (instancetype)coroutineWithBlock:(void(^)(void))block onQueue:(dispatch_queue_t _Nullable)queue stackSize:(NSUInteger)stackSize {
    if (queue == NULL) {
        queue = co_get_current_queue();
    }
    if (queue == NULL) {
        return nil;
    }
    COCoroutine *coObj = [[self alloc] initWithBlock:block onQueue:queue];
    coObj.queue = queue;
    coroutine_t  *co = coroutine_create((void (*)(void *))co_exec);
    if (stackSize > 0 && stackSize < 1024*1024) {   // Max 1M
        co->stack_size = (uint32_t)((stackSize % 16384 > 0) ? ((stackSize/16384 + 1) * 16384) : stackSize/16384);        // Align with 16kb
    }
    coObj.co = co;
    coroutine_setuserdata(co, (__bridge_retained void *)coObj, co_obj_dispose);
    return coObj;
}

- (void)resumeNow {
    [self performBlockOnQueue:^{
        if (self.isResume) {
            return;
        }
        self.isResume = YES;
        coroutine_resume(self.co);
    }];
}
```

这两个方法。其实代码已经很容易理解了，第一个方法是创建一个协程，第二个是启动。最后我们在说一下文章开头提到的 await 方法，其实最终就交给 chan 去处理了：

```objc
- (COActorCompletable *)sendMessage:(id)message {
    COActorCompletable *completable = [COActorCompletable promise];
    dispatch_async(self.queue, ^{
        COActorMessage *actorMessage = [[COActorMessage alloc] initWithType:message completable:completable];
        [self.messageChan send_nonblock:actorMessage];
    });
    return completable;
}
```

所有的操作虽然丢到了同一个线程中，但其实最终是通过 chan 来调度了。关于 chan 就不在本文讨论范围了，后面如果有时间，笔者会再进行对 chan 的分析。

## 总结

本文介绍了协程的概念，通过对比 ucontext 以及 coobjc 来说明协程的用法，并分析了 coobjc 的源代码，希望对大家有所帮助。

### 扩展阅读 

* iOS单元测试：Specta + Expecta + OCMock + OHHTTPStubs + KIF
  https://blog.csdn.net/colorapp/article/details/47007431
* 我所理解的ucontext族函数
  https://www.jianshu.com/p/dfd7ac1402f0
* 一个“蝇量级” C 语言协程库
  https://coolshell.cn/articles/10975.html
* 协程（Coroutine）并不是真正的多线程
  https://www.cnblogs.com/wonderKK/p/4062591.html
* ucontext-人人都可以实现的简单协程库
  https://blog.csdn.net/qq910894904/article/details/41911175


