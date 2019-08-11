> objc的很多设计，从底层实现上都不完全是线程安全的，这也导致在一些极端的并发情况下，会引起竞争导致的内存访问错误问题。之前分析过_weak的设计不是多线程安全的，最近又踩坑了_block，发现这个居然也不是线程安全。

当然这也不是说 `__block`, `__weak` 这些不要用了，而是说在比较频繁创建释放且有多线程使用的情况下，不要用 `__block`, `__weak` 修饰，因为他们的确不是线程安全的。

关于 `__weak` 的问题原文 *不安全的weak*<sup>[1]</sup>

## 一、问题

最近线上新版本发布后，由于框架大改版导致一个以前几乎没有暴露出来的crash突然暴露出来了，虽然总量和频率也不高，但是每天也能有几十次，这个crash还有很有特色的。

crash堆栈如下：

```sh
Exception Type: SIGTRAP
Exception Codes: 0 at 0x000000019cfb6e8c
Crashed Thread: 11

Thread 11 Crashed: 
0  libsystem_blocks.dylib         0x000000019cfb6e8c _Block_object_dispose + 284
1  mttlite                        0x0000000104bbe114 -[MttSpaceClearManager extractSimilarImages:inOperation:] (MttSpaceClearManager.mm:611)
2  mttlite                        0x0000000104bbcc04 -[MttSpaceClearManager getSimilarImagesWithDataSource:inOperation:] (MttSpaceClearManager.mm:0)
3  mttlite                        0x0000000104bbc6e0 __66-[MttSpaceClearManager getSimilarImagesWithDataSource:completion:]_block_invoke (MttSpaceClearManager.mm:0)
4  Foundation                     0x000000019dfae82c ___NSBLOCKOPERATION_IS_CALLING_OUT_TO_A_BLOCK__ +  16
 +  16
5  Foundation                     0x000000019deb6a28 -[NSBlockOperation main] +  72
6  Foundation                     0x000000019deb5efc -[__NSOperationInternal _start:] +  740
7  Foundation                     0x000000019dfb0700 ___NSOQSchedule_f +  272
8  libdispatch.dylib              0x000000019cf596c8 __dispatch_call_block_and_release +  24
9  libdispatch.dylib              0x000000019cf5a484 __dispatch_client_callout +  16
10 libdispatch.dylib              0x000000019cf30e14 __dispatch_continuation_pop$VARIANT$armv81 +  404
11 libdispatch.dylib              0x000000019cf304f8 __dispatch_async_redirect_invoke +  592
12 libdispatch.dylib              0x000000019cf3cafc __dispatch_root_queue_drain +  344
13 libdispatch.dylib              0x000000019cf3d35c __dispatch_worker_thread2 +  116
14 libsystem_pthread.dylib        0x000000019d13c17c _pthread_wqthread + 460
1  libsystem_pthread.dylib        0x000000019d13ecec _start_wqthread +  4

Binary Images:
0x10230c000 - 0x105713fff +mttlite arm64 <336143a1676d3cc18550136117ee2a60> /var/containers/Bundle/Application/AB7143CF-079B-49FA-99E9-AAE55D78491A/mttlite.app/mttlite
0x19cfb6000 - 0x19cfb6fff  libsystem_blocks.dylib arm64 <3e987452dc8b3ad9ab242066ddb75743> /usr/lib/system/libsystem_blocks.dylib
```

对应实际代码大概如下

```objc
__block NSMutableArray *images = [NSMutableArray array]; 

 //并发遍历数组groups
    [groups enumerateObjectsWithOptions:NSEnumerationConcurrent usingBlock:^(NSArray * _Nonnull group, NSUInteger idx, BOOL * _Nonnull stop) {
        //各种dispatch_async再到其它线程，同时也捕获使用了这个__block的images数组。
        
        [images addObject:xxx];
}
```

而crash就发生在enumerateObjectsWithOptions:执行完后，一直会报SIGTRAP的问题。

## 二、分析问题

### SIGTRAP

SIGTAP 类问题一般都是系统主动触发brk指令，说明有逻辑或数据异常，系统抛异常了。

crash 发生在 `_Block_object_dispose` 地址 `0x000000019cfb6e8c`，代码和分析结果如下图：

![1](http://)

`_Block_object_dispose` 函数如果走入 `SIGTRAP` 逻辑，则必定是其引用计数出现问题了，那到底怎么出现了问题，我们需要结合 `lib_closure` 源码以及反汇编来分析一下。

首先根据 crash 地址找到 crash 地址 `0x0000000104bbe114` 在我们app二进制的实际位置，计算如下：

```
相对macho偏移地址=0x0000000104bbe114-0x10230c000;
macho静态地址=0x0100000000 + 相对macho偏移地址;
```

根据macho静态地址，打开Hopper得到如下结果：

![2](http://)

就是在调用 `_Block_object_dispose` 方法的时候发生了 `crash`，`x0` 传入的时该 `__block` 内存，`x1` 比较特殊等于 8，

### 分析源码

这里我们打开 `lib_closure` 的源码查看 `_Block_object_dispose` 大概干了什么。

```c
//https://opensource.apple.com/source/libclosure/libclosure-53/
void _Block_object_dispose(const void *object, const int flags) {
    switch (osx_assumes(flags & BLOCK_ALL_COPY_DISPOSE_FLAGS)) {
      case BLOCK_FIELD_IS_BYREF | BLOCK_FIELD_IS_WEAK:
      case BLOCK_FIELD_IS_BYREF:
        // get rid of the __block data structure held in a Block
        _Block_byref_release(object);
        break;
      case BLOCK_FIELD_IS_BLOCK:
        _Block_destroy(object);
        break;
      case BLOCK_FIELD_IS_OBJECT:
        _Block_release_object(object);
        break;
      case BLOCK_BYREF_CALLER | BLOCK_FIELD_IS_OBJECT:
      case BLOCK_BYREF_CALLER | BLOCK_FIELD_IS_BLOCK:
      case BLOCK_BYREF_CALLER | BLOCK_FIELD_IS_OBJECT | BLOCK_FIELD_IS_WEAK:
      case BLOCK_BYREF_CALLER | BLOCK_FIELD_IS_BLOCK  | BLOCK_FIELD_IS_WEAK:
        break;
      default:
        break;
    }
}

enum {
    // see function implementation for a more complete description of these fields and combinations
    BLOCK_FIELD_IS_OBJECT   =  3,  // id, NSObject, __attribute__((NSObject)), block, ...
    BLOCK_FIELD_IS_BLOCK    =  7,  // a block variable
    BLOCK_FIELD_IS_BYREF    =  8,  // the on stack structure holding the __block variable
    BLOCK_FIELD_IS_WEAK     = 16,  // declared __weak, only used in byref copy helpers
    BLOCK_BYREF_CALLER      = 128, // called from __block (byref) copy/dispose support routines.
};
```


根据源码推测 `x1=8` 代表这是一个 `__block` 对象，也就是这里是在执行一个 `__block` 对象的释放，释放时发现其引用计数 <=0，从而触发了异常。

## __block内存结构

一个 `__block` 声明的对象/数据，编译后就不再是一个简单的栈上的内存数据了，而会变成一个堆上的数据，其数据结构大概如下

```c
struct Block_byref {
    void *isa;
    struct Block_byref *forwarding;
    volatile int flags; // contains ref count
    unsigned int size;
    void (*byref_keep)(struct Block_byref *dst, struct Block_byref *src);
    void (*byref_destroy)(struct Block_byref *);
    // long shared[0];
};
```

此处以 `__block NSMutableArray *images` 为例，其编译后，大概内存结构如下:

```c
struct Block_byref_xxx {
    void *isa;
    struct Block_byref *forwarding;
    volatile int flags; // contains ref count
    unsigned int size;
    void *images;//这个位置是数据images指针
};
```

### __block 引用计数管理

`__block` 对象引用计数管理大概规则如下：

当 `__block` 对象被 block 捕获使用后，第一次时会执行一次拷贝，把内存从栈上拷贝到堆上，并将引用计数增加`+2`(**__block比较特殊，引用计数增加一次是+2，而不是+1，源码上是这么写的**)，其对应的 flags 的第 1 到 11 位存储了其引用计数值；当执行第一次从栈到堆的拷贝时引用计数值 +4 (即引用计数为2)，其后每次拷贝/retain，则调用`_Block_byref_assign_copy` 函数触发引用计数值+2，当 release 时调用 `_Block_byref_release` 触发引用计数 -2，直到<=0则触发内存释放（前提是这个是堆内存）；

这里 `__block` 对象的引用计数不像 ARC 中 NSObject 一样有一个 `SideTables` 并配合加锁解锁来辅助存储每个指针对应的引用计数，而是由其自身内存 `flags` 字段存储了引用计数，本来这也没有什么大的问题，但是在这里其操作 `flags` 字段的接口并不完全是线程安全的，导致其可能 `flags` 值修改不一致。源码如下：

```c
static void _Block_byref_release(const void *arg) {
    struct Block_byref *shared_struct = (struct Block_byref *)arg;
    unsigned int refcount;

    // dereference the forwarding pointer since the compiler isn't doing this anymore (ever?)
    shared_struct = shared_struct->forwarding;
    
    // To support C++ destructors under GC we arrange for there to be a finalizer for this
    // by using an isa that directs the code to a finalizer that calls the byref_destroy method.
    if ((shared_struct->flags & BLOCK_NEEDS_FREE) == 0) {
        return; // stack or GC or global
    }
    refcount = shared_struct->flags & BLOCK_REFCOUNT_MASK;
    osx_assert(refcount);
    if (latching_decr_int_should_deallocate(&shared_struct->flags)) {
        if (shared_struct->flags & BLOCK_HAS_COPY_DISPOSE) {
            (*shared_struct->byref_destroy)(shared_struct);
        }
        _Block_deallocator((struct Block_layout *)shared_struct);
    }
}

//__block设计上是使用latching_decr_int_should_deallocate基于原子交互来确保数据竞争安全问题
//引用计数-- release
static bool latching_decr_int_should_deallocate(volatile int *where) {
    while (1) {
        int old_value = *where;
        if ((old_value & BLOCK_REFCOUNT_MASK) == BLOCK_REFCOUNT_MASK) {
            return false; // latched high
        }
        if ((old_value & BLOCK_REFCOUNT_MASK) == 0) {
            return false;   // underflow, latch low
        }
        int new_value = old_value - 2;
        bool result = false;
        if ((old_value & (BLOCK_REFCOUNT_MASK|BLOCK_DEALLOCATING)) == 2) {
            new_value = old_value - 1;
            result = true;
        }
        if (OSAtomicCompareAndSwapInt(old_value, new_value, where)) {
            return result;
        }
    }
}

//引用计数++ retain
static int latching_incr_int(volatile int *where) {
    while (1) {
        int old_value = *where;
        if ((old_value & BLOCK_REFCOUNT_MASK) == BLOCK_REFCOUNT_MASK) {
            return BLOCK_REFCOUNT_MASK;
        }
        if (OSAtomicCompareAndSwapInt(old_value, old_value+2, where)) {
            return old_value+2;
        }
    }
}
```

通过源码查看，我们发现其对应的相关 release 和 reatin 方法里会频繁直接访问或修改 flags 字段，而未完全加锁，虽然在某些关键修改处它使用了原子交互来解决线程竞争的问题，但从时机真机表现来看，iOS 系统上带的 `libsystem_blocks.dylib` 库依旧会出现极个别并发情况下，flags 引用计数竞争的问题。这时就可能会导致引用计数管理出错，从而触发其异常逻辑；

## 结论

综上，我们发现 `__block` 的引用计数管理在 iOS 上并不是完全的线程安全的，导致当一个 `__block` 对象频繁多线程并发 retain,release后，其引用计数 flags 由于数据竞争导致出错，从而触发 `libblocks` 的异常。所以 `__block` 修饰符，并不适合有大量多线程并发场景下使用；但一般情况下继续使用 `__block` 则基本不会有什么太大的问题。

### 参考链接

1. https://www.jianshu.com/p/ea9353697e16

