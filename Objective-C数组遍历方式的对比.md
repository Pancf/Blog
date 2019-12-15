## Objective-C数组遍历方式的对比

在Objective-C中，对NSArray或者NSMutableArray遍历有三种方式，分别为

方式一

```objective-c
NSArray<NSNumber *> *arr = @[@1, @2, @3];
for (int i = 0; i < arr.count; ++i) {
  //do something with arr[i]
}
```

方式二

```objective-c
for (NSNumber *obj in arr) {
	//do something with obj
}
```

方式三

```objective-c
[arr enumerateObjectsUsingBlock:];
[arr enumerateObjectsWithOptions: usingBlock:];
[arr enumerateObjectsAtIndexes: options: usingBlock:];
```

第一种方式就是从C继承而来的，每次循环移动下标`i`，通过下标来访问数组元素。

第二种方式其实是编译器提供的语法糖，通过`clang -rewrite-objc`可以看到重写为c++代码后语法糖的真实面貌。

```c++
struct __objcFastEnumerationState {
	unsigned long state;
	void **itemsPtr; //暂存的数组元素数组（部分）
	unsigned long *mutationsPtr;
	unsigned long extra[5];
};
NSNumber * obj;
//快速遍历用到的context state
	struct __objcFastEnumerationState enumState = { 0 };
	id __rw_items[16];
	id l_collection = (id) arr;
//获取数组的个数，同时会修改enumState、__rw_items，16是单次从数组取元素的个数
	_WIN_NSUInteger limit =
		((_WIN_NSUInteger (*) (id, SEL, struct __objcFastEnumerationState *, id *, _WIN_NSUInteger))(void *)objc_msgSend)
		((id)l_collection,
		sel_registerName("countByEnumeratingWithState:objects:count:"),
		&enumState, (id *)__rw_items, (_WIN_NSUInteger)16);
	if (limit) {
	unsigned long startMutations = *enumState.mutationsPtr;
	do {
		unsigned long counter = 0;
		do {
			if (startMutations != *enumState.mutationsPtr)
				objc_enumerationMutation(l_collection);
			obj = (NSNumber *)enumState.itemsPtr[counter++]; {
        //写在for-in花括号内的statements就会出现在这里
        };
	__continue_label_1: ;
		} while (counter < limit);
	} while ((limit = ((_WIN_NSUInteger (*) (id, SEL, struct __objcFastEnumerationState *, id *, _WIN_NSUInteger))(void *)objc_msgSend)
		((id)l_collection,
		sel_registerName("countByEnumeratingWithState:objects:count:"),
		&enumState, (id *)__rw_items, (_WIN_NSUInteger)16)));
	obj = ((NSNumber *)0);
	__break_label_1: ;
	}
	else
    //数组为空的分支
		obj = ((NSNumber *)0);
```

这个方式的背后的实现是每次会取出固定数量（编译器版本不同这个数字可能会有区别）的元素，通过内层的do-while循环来遍历取出来的元素，然后再取一次，直到取完为止。

方式二不便之处在于有时候需要用到元素下标，这时候就需要调用`[arr indexOfObject:]`。如果每次都需要调用这个方法，遍历大规模数组时相比第一种方式就会多出一点开销。

第三种方式就是block形式的封装，通过修改`*stop`的值来代替前两种方式的break关键字作用，同时提供了元素和元素对应的下标，对处理元素来说是最便利的。

性能上，三种方式在遍历100万大小的数组时（花括号内为NSLog打印内容），耗时分别为43.9479秒、43.2685秒、44.5855秒（MBP 16-inch，2.6 GHz 六核Intel Core i7)。差距不大，大家可以选择自己喜欢的使用方式，我比较喜欢后两者，因为当你定义数组的时候都写范型参数，这两种方法可以让你在花括号中使用正确类型的元素。