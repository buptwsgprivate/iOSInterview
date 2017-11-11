# 大标题
## 问答题
### 说说ARC和MRC的区别
ARC: 自动引用计数， MRC：手动引用计数  
在MRC时代，开发者通过retain, release, autorelease这些函数，手动的控制引用计数。  
而ARC将开发者从这个工作中解放出来，在编译期间，自动插入这些函数调用。  
ARC是一个编译器的特性。  
ARC引入了一些新的修饰关键字，如strong, weak。  
不管是ARC还是MRC，内存管理的方式并没有改变。
  

### 线程同步工具都有哪些？
主要有：Atomic Operations, Lock和Condition。  GCD中的group, barrier, semaphore也是用来在GCD中做同步的。
#### 1. Atomic Operations  
系统提供了一些原子性的数学运算和逻辑运算函数，声明在/usr/include/libkern/OSAtomic.h中。  原子操作的性能比锁要高。  
这些函数分为以下类别：  
Add: Adds two integer values together and stores the result in one of the specified variables.  
Increment: Increments the specified integer value by 1.  
Decrement: Decrements the specified integer value by 1.  
Logical OR: Performs a logical OR between the specified 32-bit value and a 32-bit mask.     
Logical AND: Performs a logical AND between the specified 32-bit value and a 32-bit mask.   
Logical XOR: Performs a logical XOR between the specified 32-bit value and a 32-bit mask.  
Compare and swap:    
Test and set:   
Test and clear:    

#### 2. Using Locks  
* Using a POSIX Mutex Lock   

```
pthread_mutex_t mutex;
void MyInitFunction()
{
    pthread_mutex_init(&mutex, NULL);
}
 
void MyLockingFunction()
{
    pthread_mutex_lock(&mutex);
    // Do work.
    pthread_mutex_unlock(&mutex);
}
```
* Using the NSLock Class  
基本的互斥锁，除了标准的加锁和解锁外，还提供了非阻塞的tryLock，设置超时的lockBeforeDate:  

```
BOOL moreToDo = YES;
NSLock *theLock = [[NSLock alloc] init];
...
while (moreToDo) {
    /* Do another increment of calculation */
    /* until there’s no more to do. */
    if ([theLock tryLock]) {
        /* Update display used by all threads. */
        [theLock unlock];
    }
}
```

* Using @synchronized  
一个使用互斥锁的便利方式，使用括号中的对象作为锁的token。性能是最差的。 
@synchronized 指令实现锁的优点就是我们不需要在代码中显式的创建锁对象，便可以实现锁的机制，但作为一种预防措施，@synchronized 块会隐式的添加一个异常处理例程来保护代码，该处理例程会在异常抛出的时候自动的释放互斥锁。所以如果不想让隐式的异常处理例程带来额外的开销，你可以考虑使用锁对象。     

```
- (void)myMethod:(id)anObj
{
    @synchronized(anObj)
    {
        // Everything between the braces is protected by the @synchronized directive.
    }
}
```
* NSRecursiveLock  
递归锁，可以被同一线程加锁多次，而不会导致死锁问题。递归锁会追踪被加锁多少次，每次成功的加锁都得匹配一次解锁。只有加锁和解锁的次数相同，锁才会被释放，其它的线程才可以加锁成功。
该锁经常使用在递归函数中。   

```
NSRecursiveLock *theLock = [[NSRecursiveLock alloc] init];
 
void MyRecursiveFunction(int value)
{
    [theLock lock];
    if (value != 0)
    {
        --value;
        MyRecursiveFunction(value);
    }
    [theLock unlock];
}
 
MyRecursiveFunction(5);
```
* NSConditionLock  
该类型的锁可以使用一个特定的值去加锁和解锁，一个典型的例子就是一个线程生产数据，另一个消费数据。  
lockWhenCondition: 1 是指当前condition为1时，才能加锁成功。  
unlockWithCondition: 1 是指解锁后，将condition置为1.

```
//生产者
id condLock = [[NSConditionLock alloc] initWithCondition:NO_DATA];
 
while(true)
{
    [condLock lock];
    /* Add data to the queue. */
    [condLock unlockWithCondition:HAS_DATA];
}
```


```
//消费者
while (true)
{
    [condLock lockWhenCondition:HAS_DATA];
    /* Remove data from the queue. */
    [condLock unlockWithCondition:(isEmpty ? NO_DATA : HAS_DATA)];
 
    // Process the data locally.
}
```

#### 3. Using Conditions  
一种特殊类型的锁，主要是用来同步操作执行的顺序。  
A condition object acts as both a lock and a checkpoint in a given thread. The lock protects your code while it tests the condition and performs the task triggered by the condition. The checkpoint behavior requires that the condition be true before the thread proceeds with its task. While the condition is not true, the thread blocks. It remains blocked until another thread signals the condition object.  
The semantics for using an NSCondition object are as follows:  
1. Lock the condition object.  
2. Test a boolean predicate. (This predicate is a boolean flag or other variable in your code that indicates whether it is safe to perform the task protected by the condition.)  
3. If the boolean predicate is false, call the condition object’s wait or waitUntilDate: method to block the thread. Upon returning from these methods, go to step 2 to retest your boolean predicate. (Continue waiting and retesting the predicate until it is true.)  
4. If the boolean predicate is true, perform the task.  
5. Optionally update any predicates (or signal any conditions) affected by your task.  
6. When your task is done, unlock the condition object.  

Using a Cocoa condition：  

```
[cocoaCondition lock];
while (timeToDoWork <= 0)
    [cocoaCondition wait];
 
timeToDoWork--;
 
// Do real work here.
 
[cocoaCondition unlock];
```

Signaling a Cocoa condition:   

```
[cocoaCondition lock];
timeToDoWork++;
[cocoaCondition signal];
[cocoaCondition unlock];
```
一次signal调用只能唤醒一个线程，而broadcoast则能唤醒所有的线程。

### Lock与Condition的区别是什么？  
一个Lock只能由一个线程加锁成功，其它的线程必须等待，直到其它线程释放锁。
Condition相当于Lock + Condition，可以由多个线程加锁成功，加锁成功以后还需要检查condition是否满足，不满足的话需要一直wait。

### AutoreleasePool
每个线程都需要自动释放池，否则无法处理被调用了autorelease方法的对象。  
在main.m中，主线程已经包含了一个自动释放池的块。  
使用GCD, Operation这些多线程技术时，线程里面也会包含自动释放池的块。  
自动释放池可以有多个，形成一个栈的结构。  
可以自己创建自动释放池的块，在块开始的时候，会PUSH一个自动释放池对象，在块结束的时候，会从栈中pop。
自动释放池被pop的时候，会对它所包含的对象发送release消息。
每次runloop迭代结束的时候，主线程会向自动释放池中的对象发送release消息。


### 用户感觉卡顿后, 如何系统分析卡顿的原因？

### .Method, SEL, IMP都是什么含义？
```
struct objc_method {
    SEL method_name OBJC2_UNAVAILABLE;
    char *method_types  OBJC2_UNAVAILABLE;
    IMP method_imp OBJC2_UNAVAILABLE;
}

typedef struct objc_method *Method;

typedef struct method_list_t {
    uint32_t entsize_NEVER_USE;  // low 2 bits used for fixup markers
    uint32_t count;
    struct method_t first;
} method_list_t;

typedef struct objc_selector *SEL; //可以认为是一个C字符串

//定义了一个函数指针类型
typedef id _Nullable (*IMP)(id _Nonnull, SEL _Nonnull, ...);

```
### .实例，类，元类的关系图   
![实例，类，元类关系图](https://github.com/buptwsgprivate/iOSInterview/blob/master/Images/instance_class_metaclass.png)

### OC的消息转发过程
如果向一个对象发送一个不支持的消息，那么默认的实现是会调用NSObject类中的doesNotRecognizeSelector:方法，此方法会抛出异常，导致应用崩溃。
但是runtime在此之前，会给3次机会。  
**第1次，+(BOOL)resolveInstanceMethod:(SEL)name 或+ (BOOL)resolveClassMethod:(SEL)name**

```
@interface Person : NSObject

@property (nonatomic, copy) NSString* name;
@property (nonatomic, assign) NSUInteger age;

@end

@implementation Person
//如果需要传参直接在参数列表后面添加就好了
void dynamicAdditionMethodIMP(id self, SEL _cmd) {
    NSLog(@"dynamicAdditionMethodIMP");
}

+ (BOOL)resolveInstanceMethod:(SEL)name {
    NSLog(@"resolveInstanceMethod: %@", NSStringFromSelector(name));
    if (name == @selector(appendString:)) {
        class_addMethod([self class], name, (IMP)dynamicAdditionMethodIMP, "v@:");
        return YES;
    }
    return [super resolveInstanceMethod:name];
}

+ (BOOL)resolveClassMethod:(SEL)name {
    NSLog(@"resolveClassMethod %@", NSStringFromSelector(name));
    return [super resolveClassMethod:name];
}

@end
```

**第2次 - (id)forwardingTargetForSelector:(SEL)aSelector;**
可以override这个函数，返回其它的能处理这个消息的对象。  

```
@implementation TestClass
- (void)logString:(NSString *)str {
    NSLog(@"log string: %@", str);
}
@end

self.tc = [TestClass new];
//self并没有logString:这个方法，如果不处理转发，则会崩溃。
[self performSelector: @selector(logString:) withObject: @"Hello"];

- (id)forwardingTargetForSelector:(SEL)aSelector {
    if (aSelector == @selector(logString:)) {
        return self.tc;
    }
    
    return [super forwardingTargetForSelector: aSelector];
}   
```

**第3次 完整转发流程**

```
- (NSMethodSignature*)methodSignatureForSelector:(SEL)aSelector {
    if (aSelector == @selector(logString:)) {
        return [self.tc methodSignatureForSelector: aSelector];
    }
    return [super methodSignatureForSelector: aSelector];
}

- (void)forwardInvocation:(NSInvocation *)anInvocation {
    SEL aSelector = [anInvocation selector];
    if ([self.tc respondsToSelector: aSelector]) {
        [anInvocation invokeWithTarget: self.tc];
    }
    else {
        [super forwardInvocation: anInvocation];
    }
 }
```
完整的转发流程，代价比较高。

用图来总结：  
![消息转发](https://github.com/buptwsgprivate/iOSInterview/blob/master/Images/message_forwarding.png)

### weak如何实现
runtime对注册的类会进行布局，对于weak修饰的对象会放入一个hash表中。用weak指向的对象内存地址作为key，当此对象的引用计数为0的时候会dealloc，假如weak指向的对象内存地址是a，那么就会以a为键在这个weak表中搜索，找到所有以a为键的weak对象，从而设置为nil。

### 用于修饰属性的atomic关键字
atomic只能保证单步操作的原子性，因此，对于简单的赋值或者读取这类操作，还是可以保证该操作的完整性。
一旦涉及到多步骤的操作，还是需要lock等其它的同步机制来确保线程安全。

### 分类(Category)中定义了和原有方法重名的方法，会是什么效果？
将分类中的方法加入类中这一操作，是在运行期系统加载分类时完成的。运行期系统会把分类中所实现的每个方法都加入类的方法列表中。如果类中本来就有此方法，而分类又实现了一次，那么分类中的方法会覆盖原来那一份实现代码。实际上可能会发生很多次覆盖，比如某个分类中的方法覆盖了“主实现”中的相关方法，而另外一个分类中的方法又覆盖了这个分类中的方法。多次覆盖的结果以最后一个分类为准。

所以，最好的做法是为方法加前缀，避免发生覆盖。

### +initialize和+load都是什么？有什么区别？
load:     
      当一个类或是分类被加载进runtime的时候，会被调用load方法。  
      父类的load方法先被调用，子类的load方法之后被调用。   
      分类的load方法在类的load方法之后被调用。    
      
initialize: 当一个类，或是它的子类在接收到第一个消息之前，这个类都会收到initialize消息。  
            父类先收到消息，然后是子类。
            每个类只会收到一次，分类不会收到。    
            
顺序：先是load，然后是initialize。
         
### 讲一下NSObject协议中的isEqual:和hash
* 若想检测对象的等同性，需要同时提供isEqual:和hash两个方法。
* 相同的对象必须具有相同的哈希码，但是两个哈希码相同的对象却未必相同，这种情况发生时叫碰撞。
* hash值用于确定对象在哈希表中的位置。
* 不要盲目地爱个检测每条属性，而是应该依照具体需求来制定检测方案。
* 编写hash方法时，应该使用性能好而且哈希码碰撞几率低的算法。一个例子是：

```
- (NSUInteger)hash {
    NSUInteger firstNameHash = [_firstName hash];
    NSUInteger lastNameHash = [_lastName hash];
    NSUInteger ageHash = _age;
    return firstNameHash ^ lastNameHash ^ ageHash;
}
```

### Method Swizzle的实质是什么？
该特性的实质是，修改selector和IMP的对应关系。


