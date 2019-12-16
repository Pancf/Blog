## Autorelease Pool

```objective-c
@autoreleasepool {
  //code could use autorelease here
}
```

Autorelease Pool得以用上述方式使用的核心其实就是C++的RAII（resource acquisition is initialization）。上面的代码是Objective-C的语法糖，用C++脱掉糖衣后就是这个样子。

```c++
extern "C" __declspec(dllimport) void * objc_autoreleasePoolPush(void);
extern "C" __declspec(dllimport) void objc_autoreleasePoolPop(void *);

struct __AtAutoreleasePool {
  __AtAutoreleasePool() {atautoreleasepoolobj = objc_autoreleasePoolPush();}
  ~__AtAutoreleasePool() {objc_autoreleasePoolPop(atautoreleasepoolobj);}
  void * atautoreleasepoolobj;
};

/* @autoreleasepool */ { __AtAutoreleasePool __autoreleasepool;

}
```

`__autoreleasepool`作为自动变量，定义的时候会调用默认构造函数，然后把hot page（就是当前使用的autorelease page）存在TLS（thread local storage）。花括号中调用过`[obj autorelease]`的对象最终会调用`AutoreleasePage::autorelease`把自己添加到hot page中，离开花括号的时候自动变量`__autoreleasepool`的析构函数被调用，向pool中保存的每个对象都发送`-release`消息。

所以如果你了解C++的话，这应该不算很高级的操作。

另外Autorelease Pool其实是由Autorelase Page为结点的双向链表，每个Page的大小为VM（虚拟内存）的页大小，在x86中为4KB。为了保证这一点，Autorelease Pool重写了class-specifical operator new，实现里按字节对齐方式分配4KB大小。除去Autorelease Page这个类的成员变量，紧随其后的内存空间保存的都是autorelease对象。**这个内存分布的方式和struct实现变长数组差不多。**

```c++
class AutoreleasePoolPage {
    magic_t const magic; //校验使用
    id *next; //指向存放下一个对象的地址
    pthread_t const thread; //保存线程ID，每个线程一个autorelease pool
    AutoreleasePoolPage * const parent;
    AutoreleasePoolPage *child;
    uint32_t const depth;
    uint32_t hiwat;
    // SIZE-sizeof(*this) bytes of contents follow
}
```

Draveness文章中有个疑问点

```c++
// 他当时不清楚为什么要根据当前页的不同状态来kii不同child的页面
if (page->lessThanHalfFull()) {
   page->child->kill();
} else if (page->child->child) {
   page->child->child->kill();
}
```

其实就是跟实现动态容量数组的原理一样，当向数组中添加元素至size == capcity时，数组会动态扩容，一般是capcity  *= 2。反过来当从数组中移除元素时，只有size < x * capcity时才会缩小capcity，释放一部分内存空间。

网络上这三篇关于Autorelease Pool的文章都是还不错的--雷纯峰的[Objective-C Autorelease Pool的实现原理](https://blog.leichunfeng.com/blog/2015/05/31/objective-c-autorelease-pool-implementation-principle/)，sunnyxx的[黑幕背后的AutoRelease](https://blog.sunnyxx.com/2014/10/15/behind-autorelease/)，Draveness的[自动释放池的前世今生--深入解析autoreleasepool](https://draveness.me/autoreleasepool)。我写这篇的目的就是补充一下我认为值得补充的东西。

