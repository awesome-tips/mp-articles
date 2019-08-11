## 为什么要重构？

h2这边的缓存层问题诟病很多。很早之前就想整体规划一下。

目前的主要问题有以下几点：

* 主要存储使用plist文件存储
	* 存储和读取的相应速度都慢
	* 如果没有文件，需要提前创建文件，徒增性能消耗
	* 存储数据容易破解。plist可以直接打开
	* 存储数据没有任何压缩，徒增缓存文件大小
* 轻量级存储主要使用了NSUserDefaults
	* 实质也是plist文件
	* 如果需要实时存储数据，需要手动调用同步
	* 都存在于一个文件中，存的越多，读取更新写入的成本越大
	* 容易滥用，造成不必要的浪费
* 还有之前前辈封装的基于sqlite的数据库

结合H2主要的业务。几乎都是轻量级存储，只有个别的计划详情，课程详情，首页数据相对多一些，并且存的场景多一下。所以一直想利用一个NoSQL，替换现有方案。在再三斟酌之后，决定使用LevelDB。

相比于之前的方案，LevelDB主要优点有以下几个方面：

1. 高可靠。避免数据存储失败的情况
2. 顺序写。写速度大大提高
3. k-v存储。API简单方便
4. 数据压缩。减小存储量，减小APP的数据大小
5. 比文件存储方式，数据相对不可见
6. 删除操作不再是文件式的覆盖写入，或者删除文件本身，而是直接处理数据，提升性能

总而言之，整体性能提高了100倍左右。对于卡顿，使用内存，启动耗时都有相应的提升。

## 层级划分

![1](http://)

而我在接入的过程中，使用了两层结构，即一层业务数据格式化，一层数据库操作。主要原因如下：

1. 方便替换之前的plist存储
2. 方便后续替换底层数据库，固定上层的业务API
3. 存储在LevelDB是NSData，上层业务可以传入NSDictionary，NSArray，NSString，NSNumber的基础数据，而不能直接传入一个对象，目的是提高数据格式化的效率。所以在上层进行数据格式化。

## 具体实现

首先，我们来看一下LevelDB的封装。

**LevelDB.h**

```objc
#import <Foundation/Foundation.h>

@interface LevelDB : NSObject

+ (instancetype)share;

- (void)set:(NSString *)key value:(id)value;
- (id)get:(NSString *)key;
- (void)del:(NSString *)key;

@end
```

对上层提供增/改，删，查的基础API。

**LevelDB.m**

```objc
#import "LevelDB.h"

#include "db.h"
#include "status.h"
#include "options.h"
#include "slice.h"

#define SliceFromString(_string_) (Slice((char *)[_string_ UTF8String], [_string_ lengthOfBytesUsingEncoding:NSUTF8StringEncoding]))
#define StringFromSlice(_slice_)  ([[NSString alloc] initWithBytes:_slice_.data() length:_slice_.size() encoding:NSUTF8StringEncoding])

static leveldb::DB *db;
static leveldb::Options options;

using namespace leveldb;
static Slice SliceFromObject(id object) {
    NSData *data = nil;
    if ([object isKindOfClass:[NSArray class]]
        || [object isKindOfClass:[NSDictionary class]]) {
        @try {
            NSError *error;
            data = [NSJSONSerialization dataWithJSONObject:object options:NSJSONWritingPrettyPrinted error:&error];
        } @catch(NSException * exception) {
            NSLog(@"%@", exception);
        }
    } else if ([object isKindOfClass:[NSString class]]) {
        data = [object dataUsingEncoding:NSUTF8StringEncoding];
    } else if ([object isKindOfClass:[NSNumber class]]) {
        data = [[NSString stringWithFormat:@"%@", object] dataUsingEncoding:NSUTF8StringEncoding];
    }
    return Slice((const char *)[data bytes], (size_t)[data length]);
}

static id ObjectFromSlice(Slice v) {
    NSData *data = [NSData dataWithBytes:v.data() length:v.size()];
    NSError *error;
    id object = [NSJSONSerialization JSONObjectWithData:data options:NSJSONReadingAllowFragments error:&error];
    if (!object) {
        object = [[NSString alloc]initWithData:data encoding:NSUTF8StringEncoding];
    }
    return object;
}

@implementation LevelDB

+ (instancetype)share {
    static LevelDB *dbManger = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        dbManger = [[LevelDB alloc]init];
        NSArray *directoryPaths = NSSearchPathForDirectoriesInDomains(NSDocumentDirectory, NSUserDomainMask, YES);
        NSString *documentDirectory = [directoryPaths objectAtIndex:0];
        NSString *file = [documentDirectory stringByAppendingPathComponent:@"storage.ldb"];
        options.create_if_missing = true;
        leveldb::Status status = leveldb::DB::Open(options, [file UTF8String], &db);
        if (!status.ok()) {
        }
        NSLog(@"Problem creating LevelDB database: %s", status.ToString().c_str());
    });
    return dbManger;
}

- (void)set:(NSString *)key value:(id)value {
    if (!value) {
        [self del:key];
        return;
    }
    leveldb::Status status = db->Put(leveldb::WriteOptions(), SliceFromString(key), SliceFromObject(value));
    if (!status.ok()) {
        NSString *errorMessage = [NSString stringWithCString:(status.ToString().c_str()) encoding:[NSString defaultCStringEncoding]];
        NSLog(@"levelDB存储失败:%@\nreson:%@\n", key, errorMessage);
    }
}

- (id)get:(NSString *)key {
    std::string strValue;
    leveldb::Status status = db->Get(leveldb::ReadOptions(), SliceFromString(key), &strValue);

    if (!(status.ok())) {
        NSString *errorMessage = [NSString stringWithCString:(status.ToString().c_str()) encoding:[NSString defaultCStringEncoding]];
        NSLog(@"levelDB获取失败:%@\nreson:%@\n",key,errorMessage);
        return nil;
    }
    return ObjectFromSlice(strValue);
}

- (void)del:(NSString *)key {
    std::string strValue;
    leveldb::Status status = db->Delete(leveldb::WriteOptions(), SliceFromString(key));
    if (!status.ok()) {
        NSString *errorMessage = [NSString stringWithCString:(status.ToString().c_str()) encoding:[NSString defaultCStringEncoding]];
        NSLog(@"levelDB删除失败:%@\nreson:%@\n",key,errorMessage);
    }
}

@end
```

其中，SliceFromObject方法和ObjectFromSlice，可以将OC数据和LevelDB需要的存储数据互相转化。

之前，尝试过 解档归档 的方式来格式化数据，好处是可以传入任意对象来进行存储，但是问题在于，效率太低，数据量增大的情况下，甚至比文件存储的效率更低，瓶颈主要在 归档 这里。

所以，最后确定为只进行NSString，NSNumber，NSArray，NSDictionary的基础数据处理，使用Json进行数据转化，效率明显提升。不过，我想，这部分还有更多的优化控件，还在思考中。

YGCacheHandler提供业务方封装。这里提供一个例子，其他不再赘述：

```objc
+ (YGUser *)user {
    NSDictionary *userInfo = [[LevelDB share]get:@"YGUser"];
    if (!userInfo) {
        NSString *path = [[YGCacheHandler rootDirectoryPath] stringByAppendingPathComponent:@"YGUser.plist"];
        userInfo = [NSDictionary dictionaryWithContentsOfFile:path];
        if (!userInfo || 0 == userInfo.allKeys.count) {
            return nil;
        }
    }
    YGUser *user = [YGUser yy_modelWithDictionary:userInfo];
    return user;
}

+ (void)updateUser:(YGUser *)user {
    NSDictionary *userInfo = [user yy_modelToJSONObject];
    [[LevelDB share]set:@"YGUser" value:userInfo];
}
```

其中，读取方法，需要注意要兼容之前版本plist文件存储的数据。

而，写入方法中，需要将YGUser的model转为基础数据，再直接写入到LevelDB。

可以看出，代码量少了很多，更加清爽简单。


