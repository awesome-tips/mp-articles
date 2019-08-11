# iOS面向切面的TableView-AOPTableView

这个是公司很久之前的开源项目，一个大牛写的，在项目中一直有在用，今天有空发了点时间看下如何实现，看了之后感觉挺有收获，故撰此文，分享给需要的同学。

该库的开源地址：MeetYouDevs/IMYAOPTableView

## 概览

### WHY AOP TableView

关于为何使用AOP，在MeetYouDevs/IMYAOPTableView这个库的简介中已经有提及到了，主要是针对在我们数据流中接入广告的这种场景，最原始的方法就是分别请求数据以及广告，根据规则合并数据，分别处理业务数据和广告数据的展示这个流程如下图所示。这种方案的弊端就是有很明显的耦合，广告和正常的业务耦合在一起了，同时也违反了设计原则中的单一职责原则，所以这种方式是做的不够优雅的，后期的维护成本也是比较大的。

![1](http://)

那么如何解决这个问题呢？如何使用一种不侵入业务的方式优雅的去解决这个问题呢？答案就是使用AOP，让正常的业务和广告并行独立滴处理，下图就是使用AOP方式处理数据流中接入广告流程图

![2](http://)

### HOW DESIGN AOP TableView

该如何设计一个可用AOP的TableView呢？设计中提到的一点是没有什么问题是通过添加一个层解决不了的，不行的话就在添加一个层！。AOP TableView中同样是存在着这个处理层的，承担着如下的职责：1、注入非业务的广告内容；2、转发不同的业务到不同的处理者；3、处理展示、业务、广告之间的转换关系；另外还有一些辅助的方法。

下面这张图是AOPTableView设计类图，IMYAOPTableViewUtils该类就是这一层，为了更加符合设计中的单一职责原则，通过分类的方式，这个类的功能被拆分在多个不同的模块中，比如处理delegate转发的IMYAOPTableViewUtils (UITableViewDelegate)、处理dataSource转发的IMYAOPTableViewUtils (UITableViewDataSource)，主要完成如下事务处理

* 注入广告内容对应的位置
* 设置AOP
* 作为TableView的真实Delegate/DataSource
* 处理转发Delegate/DataSource方法到业务或者广告
* 处理delegate转发 ->IMYAOPTableViewUtils (UITableViewDelegate)
* 处理dataSource转发->IMYAOPTableViewUtils (UITableViewDataSource)

![3](http://)

## 设置AOP

![4](http://)

AOP设置的时序图如上图所示，以下是对应的代码，创建了IMYAOPTableViewUtils对象之后，需要注入 aop class ，主要的步骤如下：

* 保存业务的Delegate/DataSource ->injectTableView方法处理
* 设置TableView的delegate/dataSource为IMYAOPBaseUtils -> injectFeedsView方法处理
* 动态创建TableView的子类 -> makeSubclassWithClass方法处理
* 并设置业务的TableView的isa指针 -> bindingFeedsView方法处理
* 设置动态创建TableView的子类的aop方法 -> setupAopClass方法处理

特别地：动态创建子类以及给动态创建的子类添加aop的方法，最终该子类型的处理方法会在 _IMYAOPTableView 类中，下面会讲到  _IMYAOPTableView 类的用途

```objc
- (void)injectTableView {
    UITableView *tableView = self.tableView;

    _origDataSource = tableView.dataSource;
    _origDelegate = tableView.delegate;

    [self injectFeedsView:tableView];
}


#pragma mark - 注入 aop class

- (void)injectFeedsView:(UIView *)feedsView {
    // 设置TableView的delegate为IMYAOPBaseUtils
    // 设置TableView的dataSource为IMYAOPBaseUtils
    struct objc_super objcSuper = {.super_class = [self msgSendSuperClass], .receiver = feedsView};
    ((void (*)(void *, SEL, id))(void *)objc_msgSendSuper)(&objcSuper, @selector(setDelegate:), self);
    ((void (*)(void *, SEL, id))(void *)objc_msgSendSuper)(&objcSuper, @selector(setDataSource:), self);
    
    self.origViewClass = [feedsView class];
    // 动态创建TableView的子类
    Class aopClass = [self makeSubclassWithClass:self.origViewClass];
    if (![self.origViewClass isSubclassOfClass:aopClass]) {
        // isa-swizzle: 设置TableView的isa指针为创建的TableView子类
        [self bindingFeedsView:feedsView aopClass:aopClass];
    }
}

/**
 isa-swizzle: 设置TableView的isa指针为创建的TableView子类
 这里需要注意的是KVO使用的也是isa-swizzle，设置了isa-swizzle之后需要把设置的KVO重新添加回去
 */
- (void)bindingFeedsView:(UIView *)feedsView aopClass:(Class)aopClass {
    id observationInfo = [feedsView observationInfo];
    NSArray *observanceArray = [observationInfo valueForKey:@"_observances"];
    ///移除旧的KVO
    for (id observance in observanceArray) {
        NSString *keyPath = [observance valueForKeyPath:@"_property._keyPath"];
        id observer = [observance valueForKey:@"_observer"];
        if (keyPath && observer) {
            [feedsView removeObserver:observer forKeyPath:keyPath];
        }
    }
    object_setClass(feedsView, aopClass);
    ///添加新的KVO
    for (id observance in observanceArray) {
        NSString *keyPath = [observance valueForKeyPath:@"_property._keyPath"];
        id observer = [observance valueForKey:@"_observer"];
        if (observer && keyPath) {
            void *context = NULL;
            NSUInteger options = 0;
            @try {
                Ivar _civar = class_getInstanceVariable([observance class], "_context");
                if (_civar) {
                    context = ((void *(*)(id, Ivar))(void *)object_getIvar)(observance, _civar);
                }
                Ivar _oivar = class_getInstanceVariable([observance class], "_options");
                if (_oivar) {
                    options = ((NSUInteger(*)(id, Ivar))(void *)object_getIvar)(observance, _oivar);
                }
                /// 不知道为什么，iOS11 返回的值 会填充8个字节。。 128
                if (options >= 128) {
                    options -= 128;
                }
            } @catch (NSException *exception) {
                IMYLog(@"%@", exception.debugDescription);
            }
            if (options == 0) {
                options = (NSKeyValueObservingOptionOld | NSKeyValueObservingOptionNew);
            }
            [feedsView addObserver:observer forKeyPath:keyPath options:options context:context];
        }
    }
}

#pragma mark - install aop method
/**
 动态创建TableView的子类
 */
- (Class)makeSubclassWithClass:(Class)origClass {
    NSString *className = NSStringFromClass(origClass);
    NSString *aopClassName = [kAOPFeedsViewPrefix stringByAppendingString:className];
    Class aopClass = NSClassFromString(aopClassName);

    if (aopClass) {
        return aopClass;
    }
    aopClass = objc_allocateClassPair(origClass, aopClassName.UTF8String, 0);

    // 设置动态创建的子类的aop方法，真实处理方法是在_IMYAOPTableView类中的aop_前缀的方法
    [self setupAopClass:aopClass];

    objc_registerClassPair(aopClass);
    return aopClass;
}

/**
  设置动态创建的子类的aop方法，这里做了省略
 */
- (void)setupAopClass:(Class)aopClass {
    ///纯手动敲打
    [self addOverriteMethod:@selector(class) aopClass:aopClass];
    [self addOverriteMethod:@selector(setDelegate:) aopClass:aopClass];
    // ....
    
    ///UI Calling
    [self addOverriteMethod:@selector(reloadData) aopClass:aopClass];
    [self addOverriteMethod:@selector(layoutSubviews) aopClass:aopClass];
    [self addOverriteMethod:@selector(setBounds:) aopClass:aopClass];
    // ....
    ///add real reload function
    [self addOverriteMethod:@selector(aop_refreshDataSource) aopClass:aopClass];
    [self addOverriteMethod:@selector(aop_refreshDelegate) aopClass:aopClass];
    // ....

    // Info
    [self addOverriteMethod:@selector(numberOfSections) aopClass:aopClass];
    [self addOverriteMethod:@selector(numberOfRowsInSection:) aopClass:aopClass];
    // ....

    // Row insertion/deletion/reloading.
    [self addOverriteMethod:@selector(insertSections:withRowAnimation:) aopClass:aopClass];
    [self addOverriteMethod:@selector(deleteSections:withRowAnimation:) aopClass:aopClass];
    // ....

    // Selection
    [self addOverriteMethod:@selector(indexPathForSelectedRow) aopClass:aopClass];
    [self addOverriteMethod:@selector(indexPathsForSelectedRows) aopClass:aopClass];
    // ....

    // Appearance
    [self addOverriteMethod:@selector(dequeueReusableCellWithIdentifier:forIndexPath:) aopClass:aopClass];
}

- (void)addOverriteMethod:(SEL)seletor aopClass:(Class)aopClass {
    NSString *seletorString = NSStringFromSelector(seletor);
    NSString *aopSeletorString = [NSString stringWithFormat:@"aop_%@", seletorString];
    SEL aopMethod = NSSelectorFromString(aopSeletorString);
    [self addOverriteMethod:seletor toMethod:aopMethod aopClass:aopClass];
}

- (void)addOverriteMethod:(SEL)seletor toMethod:(SEL)toSeletor aopClass:(Class)aopClass {
    // 这里的这个implClass在AOPTableViewUtils中为_IMYAOPTableView
    Class implClass = [self implAopViewClass];
    Method method = class_getInstanceMethod(implClass, toSeletor);
    if (method == NULL) {
        method = class_getInstanceMethod(implClass, seletor);
    }
    const char *types = method_getTypeEncoding(method);
    IMP imp = method_getImplementation(method);
    // 添加aopClass也就是创建的子类型kIMYAOP_UITableView的处理方法，真实处理方法是在_IMYAOPTableView类中的
    class_addMethod(aopClass, seletor, imp, types);
}
```

_IMYAOPTableView的职责是在业务端直接使用TableView对应的方法的时候，把业务的规则转换为真实列表的规则，比如下面的业务端调用了cellForRowAtIndexPath这个方法，会走到如下的方法中，这里的indexPath是业务自己的indexPath，比如在列表可见的第五个位置，但是前面是有两个广告，在业务端的逻辑中该indexPath对应的位置是在第三个位置的，所以需要进行修正，返回正确的IndexPath，获取到对应位置的Cell，这样才不会有问题

```objc
- (UITableViewCell *)aop_cellForRowAtIndexPath:(NSIndexPath *)indexPath {
    AopDefineVars;
    if (aop_utils) {
        // 修复业务使用的indexPath为真实的indexPath
        indexPath = [aop_utils feedsIndexPathByUser:indexPath];
    }
    aop_utils.isUICalling += 1;
    UITableViewCell *cell = AopCallSuperResult_1(@selector(cellForRowAtIndexPath:), indexPath);
    aop_utils.isUICalling -= 1;
    return cell;
}
```

## 使用AOP

### 非业务数据插入

IMYAOPBaseUtils类提供了两个方法用于非业务数据的处理

```objc
///插入sections 跟 indexPaths
- (void)insertWithSections:(nullable NSArray<__kindof IMYAOPBaseInsertBody *> *)sections;
- (void)insertWithIndexPaths:(nullable NSArray<__kindof IMYAOPBaseInsertBody *> *)indexPaths;

// 实现
- (void)insertWithIndexPaths:(NSArray<IMYAOPBaseInsertBody *> *)indexPaths {
    NSArray<IMYAOPBaseInsertBody *> *array = [indexPaths sortedArrayUsingComparator:^NSComparisonResult(IMYAOPBaseInsertBody *_Nonnull obj1, IMYAOPBaseInsertBody *_Nonnull obj2) {
        return [obj1.indexPath compare:obj2.indexPath];
    }];

    NSMutableDictionary *insertMap = [NSMutableDictionary dictionary];
    [array enumerateObjectsUsingBlock:^(IMYAOPBaseInsertBody *_Nonnull obj, NSUInteger idx, BOOL *_Nonnull stop) {
        NSInteger section = obj.indexPath.section;
        NSInteger row = obj.indexPath.row;
        NSMutableArray *rowArray = insertMap[@(section)];
        if (!rowArray) {
            rowArray = [NSMutableArray array];
            [insertMap setObject:rowArray forKey:@(section)];
        }
        while (YES) {
            BOOL hasEqual = NO;
            for (NSIndexPath *inserted in rowArray) {
                if (inserted.row == row) {
                    row++;
                    hasEqual = YES;
                    break;
                }
            }
            if (hasEqual == NO) {
                break;
            }
        }
        NSIndexPath *insertPath = [NSIndexPath indexPathForRow:row inSection:section];
        [rowArray addObject:insertPath];
        obj.resultIndexPath = insertPath;
    }];
    self.sectionMap = insertMap;
}
```

调用insertWithIndexPaths插入非业务的广告数据，这里插入的数据是位置

```objc
///简单的rows插入
- (void)insertRows {
    NSMutableArray<IMYAOPTableViewInsertBody *> *insertBodys = [NSMutableArray array];
    ///随机生成了5个要插入的位置
    for (int i = 0; i < 5; i++) {
        NSIndexPath *indexPath = [NSIndexPath indexPathForRow:arc4random() % 10 inSection:0];
        [insertBodys addObject:[IMYAOPTableViewInsertBody insertBodyWithIndexPath:indexPath]];
    }
    ///清空 旧数据
    [self.aopUtils insertWithSections:nil];
    [self.aopUtils insertWithIndexPaths:nil];

    ///插入 新数据, 同一个 row 会按数组的顺序 row 进行 递增
    [self.aopUtils insertWithIndexPaths:insertBodys];

    ///调用tableView的reloadData，进行页面刷新
    [self.aopUtils.tableView reloadData];

    dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(1 * NSEC_PER_SEC)), dispatch_get_main_queue(), ^{
        NSLog(@"%@", self.aopUtils.allModels);
    });
}
```

在demo中使用了如上的代码调用，sectionMap中保存的数据如下，key为section，value是对应section下所有插入数据的IndexPath数组，sectionMap数据会用于处理真实数据和业务数据之间的映射

![5](http://)

userIndexPathByFeeds方法使用sectionMap处理真实indexPath和业务indexPath之间的变换

```objc
// 获取业务对应的indexPath，该方法的作用是进行indexPath，比如真实的indexPath为(0-5)，前面插入了两个广告，会把indexPath修复为业务的indexPath，也就是(0-3)，如果该位置是广告的位置，那么返回nil空值  
- (NSIndexPath *)userIndexPathByFeeds:(NSIndexPath *)feedsIndexPath {
    if (!feedsIndexPath) {
        return nil;
    }
    NSInteger section = feedsIndexPath.section;
    NSInteger row = feedsIndexPath.row;

    NSMutableArray<NSIndexPath *> *array = self.sectionMap[@(section)];
    NSInteger cutCount = 0;
    for (NSIndexPath *obj in array) {
        if (obj.row == row) {
            cutCount = -1;
            break;
        }
        if (obj.row < row) {
            cutCount++;
        } else {
            break;
        }
    }
    if (cutCount < 0) {
        return nil;
    }
    ///如果该位置不是广告， 则转为逻辑index
    section = [self userSectionByFeeds:section];
    NSIndexPath *userIndexPath = [NSIndexPath indexPathForRow:row - cutCount inSection:section];
    return userIndexPath;
}
```

### AOP代理方法回调

![6](http://)

如上图所示，IMYAOPTableViewUtils作为中间层承担了作为TableView的delegate和dataSource的职责，在改类中处理对应事件的转发到具体的处理者：业务端或者是非业务的广告端

比如下面的获取cell的代理方法tableView:cellForRowAtIndexPath:，首先会进行indexPath的修复，然后判断是业务的还是非业务的，然后使用不同的dataSource进行相应的处理，代码段有做了注释，详情参加注释的解释

```objc
- (UITableViewCell *)tableView:(UITableView *)tableView cellForRowAtIndexPath:(NSIndexPath *)indexPath {
    kAOPUICallingSaved;
    kAOPUserIndexPathCode;
    UITableViewCell *cell = nil;
    if ([dataSource respondsToSelector:@selector(tableView:cellForRowAtIndexPath:)]) {
        cell = [dataSource tableView:tableView cellForRowAtIndexPath:indexPath];
    }
    if (![cell isKindOfClass:[UITableViewCell class]]) {
        cell = [UITableViewCell new];
        if (dataSource) {
            NSAssert(NO, @"Cell is Nil");
        }
    }
    kAOPUICallingResotre;
    return cell;
}

// 宏定义的代码段，用户是判断该位置是否是业务使用的IndexPath，是的话返回业务的DataSource->origDataSource，否则返回非业务的DataSource->dataSource  
#define kAOPUserIndexPathCode                                           \
    NSIndexPath *userIndexPath = [self userIndexPathByFeeds:indexPath]; \
    id<IMYAOPTableViewDataSource> dataSource = nil;                     \
    if (userIndexPath) {                                                \
        dataSource = (id)self.origDataSource;                           \
        indexPath = userIndexPath;                                      \
    } else {                                                            \
        dataSource = self.dataSource;                                   \
        isInjectAction = YES;                                           \
    }                                                                   \
    if (isInjectAction) {                                               \
        self.isUICalling += 1;                                          \
    }

// 获取业务对应的indexPath，该方法的作用是进行indexPath，比如真实的indexPath为(0-5)，前面插入了两个广告，会把indexPath修复为业务的indexPath，也就是(0-3)，如果该位置是广告的位置，那么返回nil空值  
- (NSIndexPath *)userIndexPathByFeeds:(NSIndexPath *)feedsIndexPath {
    if (!feedsIndexPath) {
        return nil;
    }
    NSInteger section = feedsIndexPath.section;
    NSInteger row = feedsIndexPath.row;

    NSMutableArray<NSIndexPath *> *array = self.sectionMap[@(section)];
    NSInteger cutCount = 0;
    for (NSIndexPath *obj in array) {
        if (obj.row == row) {
            cutCount = -1;
            break;
        }
        if (obj.row < row) {
            cutCount++;
        } else {
            break;
        }
    }
    if (cutCount < 0) {
        return nil;
    }
    ///如果该位置不是广告， 则转为逻辑index
    section = [self userSectionByFeeds:section];
    NSIndexPath *userIndexPath = [NSIndexPath indexPathForRow:row - cutCount inSection:section];
    return userIndexPath;
}
```

## 结束

就先写到这了，如果不妥之处敬请赐教

