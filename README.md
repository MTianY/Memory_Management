### copy

拷贝的目的: 产生一个副本对象,跟原对象互不影响.

- `copy`: 不可变拷贝, 产生不可变副本.
- `mutableCopy`: 可变拷贝, 产生可变副本.

1.创建一个不可变字符串

```objc
NSString *str1 = [NSString stringWithFormat:@"test"];
// 返回不可变字符串 NSString
NSString *str2 = [str1 copy];

// 返回可变字符串 NSMutableString
NSMutableString *str3 = [str1 mutableCopy];

// 如 str3 可以拼接其他字符串
[str3 appendString:@"123"];

// 这样 str3 就是 test123
```

2.创建一个可变的字符串

```objc
NSMutableString *str1 = [NSMutableString stringWithFormat:@"test"];
// 返回一个不可变字符串
NSString *str2 = [str1 copy];
// 返回一个可变的字符串
NSMutableString *str3 = [str1 mutableCopy];
```

所以得出结论: 

- 调用 `copy` 返回的就是不可变的.
- 调用 `mutableCopy` 返回的就是可变的.

### copy 内存管理

`MRC`环境下,`copy`后要记得`release`操作.

### 深拷贝、浅拷贝

##### 深拷贝:

- 内容拷贝, 产生新的对象

##### 浅拷贝:

- 指针拷贝, 没有产生新的对象

##### 1. NSString 和 NSMutableString 为例:

1. 先定义一个不可变字符串

```objc
NSString *string1 = [NSString stringWithFormat:@"test"];
```

```objc
// copy
// 返回一个不可变字符串
// 浅拷贝
NSString *string2 = [string1 copy];

// mutableCopy
// 返回一个可变字符串
// 深拷贝
NSMutableString *string3 = [string1 mutableCopy];
```

内存图:
![](https://lh3.googleusercontent.com/-dDUQdGv4X7w/W9fjm91elOI/AAAAAAAAAQM/5qxLAReH-q8FP8JV4TavJMhRzL5tWDaLgCHMYCw/I/15408751568928.jpg)



2. 定义一个可变字符串

```objc
NSMutableString *string1 = [NSMutableString stringWithFormat:@"test"];
```

```objc
// copy
// 返回一个不可变字符串
// 深拷贝
NSString *string2 = [string1 copy];

// mutableCopy
// 返回一个可变字符串
// 浅拷贝
NSMutableString *string3 = [string1 mutableCopy];
```

内存图:

![](https://lh3.googleusercontent.com/-CotZm8O0gX0/W9fj9M19ngI/AAAAAAAAAQU/JkLux0ZHG0guz7CuJauuW3qj8fD3s_6uACHMYCw/I/15408752455424.jpg)


##### 总结:

- 如果一开始就是不可变的, 那么执行`copy`操作,会返回一个不可变,既然都是不可变的,那么就拷贝指针就好,不用另开辟内存去存同一个不可变的东西,因为不可变别人本来也不能改变它.所以是浅拷贝,指针拷贝
- 深拷贝,内容拷贝,拷贝后在内存另开辟了一个空间存新拷贝的东西.


##### 2. NSArray 、 NSMutableArray 为例(NSDictionary 和 NSMutableDictionary 与这个都类似)

```objc
NSArray *array1 = [[NSArray alloc] initWithObjects:@"a", @"b", nil];

// 浅拷贝
NSArray *arr2 = [array1 copy];

// 深拷贝
NSMutableArray *arr3 = [array1 mutableCopy];
```

## @property 的 修饰

都是对 set 方法的管理不一样

- `assign`

直接返回值

- `retain`

在 set 先 release 掉之前的值,然后 retain 新的值.

- `copy`

举例

```objc
// TYPerson 有个属性
@property (nonatomic, copy) NSArray *data;

TYPerson *p = [[TYPerson alloc] init];
p.data = @[@"jack", @"ros"];

那么其本质就是
- (void)setData:(NSArray *)data {
    if(_data != data) {
        [_data release];
        _data = [data copy];
    }
}

- (void)dealloc {
    self.data = nil;
    [super dealloc];
}
```

所以用下面这个属性定义,如果使用 set 方法就会报错

```objc
@property (nonatomic, copy) NSMutableArray *array;
```

如果使用这个 `array` 的 `set` 方法,那么会先 `release`, 然后 `copy`, 为一个不可变的数组,如果再往里加东西,直接崩,找不到 `addObject:` 的方法.

### 引用计数

从`arm64`开始,对`isa`有了优化,`引用计数`就存储在这个`isa`指针中.

- `isa`中只有19位.如果不够存储的话,那么就会存到 `SideTable`类中.里面有个散列表,用来存

### weak 指针的原理

程序运行过程中,将弱引用存在哈希表中,将来销毁时取出销毁,然后将当前对象置为 nil.

weak 需要 runtime 的.

### Autorelease 什么时候释放?

`autoreleasePool` 本质是:

- 开始调用一个 `objc_autoreleasePoolPush()` 
- 并在结束时调用 `objc_autoreleasePoolPop(atautoreleasepoolobj)`

调用了`autorelease` 的对象最终都是通过 `AutoreleasePoolPage` 对象来管理的.

简化后的 `AutoreleasePoolPage` 对象如下:

```C++
class AutoreleasePoolPage {
    magic_t const magic;
    id *next;
    pthread_t const thread;
    AutoreleasePoolPage * const parent;
    AutoreleasePoolPage *child;
    unit32_t const depth;
    unit32_t hiwat;
}   
```

- `AutoreleasePoolPage` 对象占 `4096` 个字节内存,除了存放自己的成员变量,剩下的空间存 `autorelease` 对象的地址.
- 所有的`AutoreleasePoolPage` 对象通过双向链表的形式连接在一起.
- 调用`push`方法会将一个`POOL_BOUNDARY`入栈, 并且返回其存的内存地址.
- 调用`pop`时会传入一个`POOL_BOUNDARY`的内存地址,从最后一个入栈的对象开始,发送 `release`消息,直到遇到这个`POOL_BOUNDARY`.
- `id *next` 指向了下一个能存放 `autorelease` 对象地址的区域.

#### 问: autoreleasePool 对象什么时候执行 release 操作?

如果是下面这种被 `@autoreleasepool` 包住的,那么就是当大括号结束了就释放.

```objc
@autoreleasepool {
    
}
```

- 如果用`autoreleasePool` 来释放,那么就与 `RunLoop` 有关.

    - 因为 `iOS` 默认在主线程的 `RunLoop` 中注册了 `2个Observer`.
    - 第一个`Observer`监听了 `KCFRunLoopEntry` 事件,会调用 `objc_autoreleasePoolPush()`.
    - 第二个`Observer`监听`KCFRunLoopBeforeWaiting (即将休眠)` 事件,这时会调用 `objc_autoreleasePoolPop()`, `objc_autoreleasePoolPush()`操作.
    - 同时第二个`Observer`也监听了`KCFRunLoopBeforeExit(即将退出)`事件,会调用`objc_autoreleasePoolPop()` .

    
所以,如果问局部变量什么时候释放? 那要看`ARC(现在都是 ARC)`是用`autoreleasePool` 技术还是`release`技术,如果是前者,那么要看 RunLoop, 如果是后者,大括号结束就立即释放.xin


