## Autoreleasepool

### 原理   

先看一下程序的入口`main.m`文件,clang 编译后在`main.cpp`文件中看到   

```objc
int main(int argc, char * argv[]) {
    /* @autoreleasepool */ { __AtAutoreleasePool __autoreleasepool; 
        return UIApplicationMain(argc, argv, __null, NSStringFromClass(((Class (*)(id, SEL))(void *)objc_msgSend)((id)objc_getClass("AppDelegate"), sel_registerName("class"))));
    }
}
```   
可以看到`@autoreleasepool`被转换为一个`__AtAutoreleasePool` 结构体。继续来看`__AtAutoreleasePool`结构体。   

```objc
struct __AtAutoreleasePool {
  __AtAutoreleasePool() {atautoreleasepoolobj = objc_autoreleasePoolPush();}
  ~__AtAutoreleasePool() {objc_autoreleasePoolPop(atautoreleasepoolobj);}
  void * atautoreleasepoolobj;
};
```   
结构体初始化的时候调用`objc_autoreleasePoolPush()`   
析构的时候调用`objc_autoreleasePoolPop()`   
实际上`@autoreleasepool{}` 实际是转换成了如下的代码：   

```objc
void *context = objc_autoreleasePoolPush();
// {} 中的代码
objc_autoreleasePoolPop(context);
```   

在`NSObjcet.mm`中查找`objc_autoreleasePoolPush`和`objc_autoreleasePoolPop`方法。如下：

```objc
void *
objc_autoreleasePoolPush(void)
{
    return AutoreleasePoolPage::push();
}
void
objc_autoreleasePoolPop(void *ctxt)
{
    AutoreleasePoolPage::pop(ctxt);
}
```   
可以看到这两个方法实际上就是对`AutoreleasePoolPage`对应`push`,`pop`方法的封装。那么核心的地方就在`AutoreleasePoolPage`这个类中。   
#### AutoreleasepoolPage

```objc
    magic_t const magic;
    id *next;
    pthread_t const thread;
    AutoreleasePoolPage * const parent;
    AutoreleasePoolPage *child;
    uint32_t const depth;
    uint32_t hiwat;
```   

- `magic`用于校验当前`AutoreleasepoolPage`的完整性   
- `thread`指向当前线程
- `Autoreleasepool`没有单独的结构，是由若干个`AutoreleasepoolPage`以双向链表的形式组合而成
- 每一个`AutoreleasepoolPage`对象会开辟4096字节内存(虚拟内存一页的大小)。除了上述实例变量所占空间，剩余的都用来存储autorelease对象的地址。   
- 一个`AutoreleasepoolPage`空间占满的时候，会创建一个新的`AutoreleasepoolPage`对象，连接链表，新加入的autorelease对象加入新的page。   
- `id *next`指针作为游标指向栈顶最新add进来的autorelease对象的下个位置   

![](https://github.com/drunkbread/Doc/blob/master/resources/autoreleasepool_1.jpg)

> 向一个对象发送`autorelease`消息的时候，就是将这个对象加入到当前`AutoreleasepoolPage`的栈顶`next`指针指向的位置。   

#### 哨兵对象   
每当调用一次`AutoreleasePoolPush()`的时候，会在栈顶加入一个哨兵对象(就是一个nil), `AutoreleasePoolPush()`的返回值就是这个哨兵对象的地址，被`objc_autoreleasePoolPop()`作为参数。    
在`objc_autoreleasePoolPop()`调用时：   

- 根据传入的哨兵对象地址找到哨兵对象所在的page   
- 从最新加入的对象一直向前清理，可以跨越多个page，直到哨兵对象所在的page。并且把`next`指针回位。   

![](https://github.com/drunkbread/Doc/blob/master/resources/autoreleasepool_2.jpg)    
![](https://github.com/drunkbread/Doc/blob/master/resources/autoreleasepool_3.jpg)  

> 对于嵌套的`autoreleasepool`，pop时总会清理到上次push的位置。多层pool就是多个哨兵对象。   

### 释放时机   
调用`objc_autoreleasePoolPop()`的时机




## 参考摘录

[黑幕背后的Autorelease](https://blog.sunnyxx.com/2014/10/15/behind-autorelease/)   
[Objective-C Autorelease Pool 的实现原理](http://blog.leichunfeng.com/blog/2015/05/31/objective-c-autorelease-pool-implementation-principle/)   
[自动释放池的前世今生](https://draveness.me/autoreleasepool)


