
## 前言

基于 AFNetworking 的二次封装网上蛮多的，比较好一点的就是 CTNetworking 和 YTKNetwork，但是看了一下源码过后发现都有一些不足的地方，或者说不太能满足我们的业务需求。考虑到 AFNetworking 本身就为网络层做了很多事情，二次封装并非是个复杂的事情，所以索性自己写了个便于拓展和维护 （代码完全脱敏）：

**代码地址和用法** : `YBNetwork` (https://github.com/indulgeIn/YBNetwork)

## 调研

Casa Taloyum 前辈的文章对笔者的架构思维有着深远的影响，记得两年多前入行不久，看得一知半解，近些时间要做架构方面的工作，又去重温了一下。

如何设计一个好的网络层架构，在 Casa Taloyum 的文章中已经说得比较全面了，不过似乎作者有点懒，文章和 CTNetworking 有些出入 😂。猿题库的 YTKNetwork 相对比较成熟，两份代码核心思想都是将代码归为集约处理部分和离散处理部分，在实现方式上有些差别。

没有什么技术难点，直接看了一遍两份开源代码，优点很多，这里罗列一下不足的地方（当然只是个人理解，并且笔者可能更多结合业务来考虑的）：

**CTNetworking 不足：**

- 使用 IOP 方式建立模块，化继承为组合。独立`CTServiceProtocol`协议类作为一个接口团队的公有配置，若针对一个接口团队的某一个特定接口需要特别处理，也就是需要专门定制`CTServiceProtocol`的某些协议方法，那就很棘手了（当然若接口方非常规范就没有这个顾虑）。
- 记录了一个 request 实例的所有 task，在 dealloc 中自动取消掉还未降落的网络请求，但是实际上网络请求任务会持有 request，所以自动取消策略不成立了（估计是作者未完善的，因为博客中有写）。

**YTKNetwork 不足：**

- 基于多态的设计思路，提供了很多供重载的方法，从设计来看，框架是可以实例化`YTKBaseRequest`子类 直接使用的，那么直接使用时无法重载这些方法专门定制（个人看来有些地方使用属性更灵活）；并且，当一个 reqeust 多次`start`发起请求就会调用多次这些重载方法，可能造成多余计算；
- 缓存策略使用一个`YTKBaseRequest`的子类`YTKRequest`来做，虽然这样看起来比较优雅，父类和子类各司其职，单一职责，但是缓存策略难免会更改父类的逻辑，如此就很难不违背开闭原则。框架的缓存只有一个失效时间控制，笔者想要拓展时发现要改的东西太多。
- 同一个 request 实例多次 start 调用网络请求时 (多个网络请求并发情况)，并未作出实际的处理策略，仅保留最新的`NSURLSessionTask`，而对旧的未结束的所有`NSURLSessionTask`丧失了控制权。
- 网络请求任务强持有所有 request 对象，在弱网环境下可能会有大量 request 对象无法释放，而界面降落点可能不存在了。

**共同不足：**

- 数据回调都是绑定在 request 上的，既然都未处理一个 request 重复并发请求的情况，那么多个网络请求落地时，request 上的数据会突变，业务方的处理方式是不可控的，既有可能在回调业务执行过程中发现数据变化了。

实际上针对团队的业务，架构上会有取舍，所以笔者列这些不足也可以说是比较片面的。

## 实现

**如何进行离散请求调用？**

在一个网络请求起飞到降落过程中，有一系列独有的配置始终能代表这一个网络请求。

那么思路就出来了，只要把一个针对某个接口的配置对象传递过去，让网络任务的闭包持有这个对象，然后在网络回调处理中，一直传递这个配置对象，像踢皮球一样，最终处理好后回调到业务类中。

怎么避免这个配置对象疯狂传递？实际上就可以把网络回调处理逻辑，放在这个配置对象中，就像`CTNetworking`的`CTAPIBaseManager`配置对象，只要安全落地就能命中对应的配置对象；也可以用一个全局容器把这些配置对象装起来，不用一直通过闭包传递，就像`YTKNetwork`的`YTKBaseRequest`配置对象。

所以笔者之前用了一个奇怪的思路：

```objc
Config config = Config.new;
[NetworkManager startWithConfig:config success:^{} failure:^{}];
```

实际上这和上面两个框架道理是一样的，笔者内部也会写逻辑去管理所有`config`，但是这么做不好对单独的网络请求进行管理，非要管理的话，又需要去持有这个`config`了。

实现代码类：

- YBNetworkManager : 负责组织数据发起网络请求，并且管理所有的 NSURLSessionTask
- YBNetworkCache : 负责缓存处理
- YBNetworkResponse : 回调响应结果
- YBBaseRequest : 负责离散数据配置、网络响应处理逻辑

### 集约/离散配置方式

为了更加灵活，并没有采用 IOP 方式来做配置管理，而是采用继承的方式来做，为了提高灵活性，定制几率大的配置使用属性实现，需要重载的方法使用分类提出来看起来保证清晰。

在开发中，需要针对不同的接口团队创建不同的`YBBaseRequest`子类集约配置，比如`DefaultServerRequest : YBBaseRequest`。在使用时，可以直接实例化`DefaultServerRequest`或者子类化`DefaultServerRequest`进行离散配置。

主要思路和 YTKNetwork 基本一样，当然像  CTNetworking 这样强制子类化来使用接口更好管理，但是有些时候显得有些繁琐。

笔者这种处理方式虽然需要子类化一些`YBBaseRequest`进行公共配置，但是也保证了每一个请求接口实例都可以任意的定制集约管理部分，防止接口抽风。


### 缓存处理

缓存处理专门提取一个类来包装逻辑，而调用逻辑仍然放在`YBBaseRequest`，实际上代码量很少，也好修改。

出于业务考虑，缓存支持的功能有：
- 内存/磁盘存储方式
- 缓存命中后是否继续发起网络请求
- 缓存的有效时长
- 定制缓存的 key
- 根据请求响应成功数据判断是否需要缓存（比如仅当 code=0 时数据有效允许缓存）

对于缓存命中的回调，笔者设置了专门的回调出口：

```objc
//Block
- (void)startWithCache:(nullable YBRequestCacheBlock)cache
success:(nullable YBRequestSuccessBlock)success
failure:(nullable YBRequestFailureBlock)failure;

//Delegate
- (void)request:(__kindof YBBaseRequest *)request cacheWithResponse:(YBNetworkResponse *)response;
```

对于 Block 方式 来说，独立的缓存回调闭包更好管理。
对于两种回调来说，设计一个专门的缓存回调能降低业务工程师的出错率。
对于网络及时数据和缓存数据往往在业务处理上有细微的差别，分开回调能避免出于疏忽而去写判断`if (isCache) {...} else {...}`（特别是当写业务的工程师并不知道这个 API 缓存策略是怎样的）。


### 重复网络请求处理

提供三种方式：

1. 允许重复网络请求
2. 取消最旧的网络请求
3. 取消最新的网络请求

实现比较简单，具体采用哪种策略还需要根据业务谨慎选择：

```objc
- (void)start {
    if (self.isExecuting) {
        switch (self.repeatStrategy) {
            case YBNetworkRepeatStrategyCancelNewest: return;
            case YBNetworkRepeatStrategyCancelOldest: {
                [self cancel];
            }
                break;
            default: break;
        }
    }
    ...
}
```


### 网络请求释放处理

提供三种方式：

1. 网络任务会持有`YBBaseRequest`实例，网络任务完成`YBBaseRequest`实例才会释放
2. 网络请求将随着`YBBaseRequest`实例的释放而取消
3. 网络请求和`YBBaseRequest`实例无关联

实现网络任务对 YBBaseRequest 弱持有 ，当`YBNetworkManager`发起请求时，让回调闭包捕获弱引用的`weakSelf`的就行了， 

```objc
__weak typeof(self) weakSelf = self;
taskID = [[YBNetworkManager sharedManager] startNetworkingWithRequest:weakSelf uploadProgress:^(NSProgress * _Nonnull progress) {
    __strong typeof(weakSelf) self = weakSelf;
    if (!self) return;
    [self requestUploadProgress:progress];
} downloadProgress:^(NSProgress * _Nonnull progress) {
    __strong typeof(weakSelf) self = weakSelf;
    if (!self) return;
    [self requestDownloadProgress:progress];
} completion:^(YBNetworkResponse * _Nonnull response) {
    __strong typeof(weakSelf) self = weakSelf;
    if (!self) return;
    YBN_IDECORD_LOCK([self.taskIDRecord removeObject:taskID];);
    [self requestCompletionWithResponse:response cacheKey:cacheKey fromCache:NO];
}];
```

而要让`YBBaseRequest`释放时自动取消网络请求只需要简单调用（不过在“网络请求和 YBBaseRequest 实例无关联”模式时是不能取消的）：

```objc
- (void)dealloc {
    ...
    if (self.releaseStrategy == YBNetworkReleaseStrategyWhenRequestDealloc) {
    [self cancel];
    }
}
```

### 回调处理

为了让重复网络请求时，每次回调的数据不相互影响，笔者思来想去还是额外定义了一个类，而不是直接让`YBBaseRequest`持有，并且同`CTNetworking`一样预定义了一个`YBResponseErrorType`，包含三种类型：超时、取消、无网。

至于为什么要单独定义一个类，而不是直接回调一个`id respondsObject`，因为有些业务中还需要其它数据，比如头部信息，那么单独定义一个类便于拓展回调内容，并且也降低了框架内部数据流通过程中的成本（传递一个对象总比传递一堆对象好处理吧）。

## 后语

大体思路就是如此，至于线程安全啥的细节就不多说了，主要是在加锁的时候注意避免同一线程重复获取锁导致死锁就行了。

一个看似简单的二次封装也能有这么多值得思考的地方，精益求精并不是一件容易的事。

### 参考

* iOS应用架构谈 网络层设计方案 
	https://casatwy.com/iosying-yong-jia-gou-tan-wang-luo-ceng-she-ji-fang-an.html
* YTKNetwork
	https://github.com/yuantiku/YTKNetwork 
* CTNetworking 
	https://github.com/casatwy/CTNetworking






