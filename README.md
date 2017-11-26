# iOS面试题汇总
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
该特性的实质是，为类添加一个新的方法，然后将目标方法和新的方法的IMP进行互换，结果就是修改selector和IMP的对应关系。

### Block深入
Block本身是对象，下图描述了Block对象的内存布局：  
![Block对象内存布局](https://github.com/buptwsgprivate/iOSInterview/blob/master/Images/Block_Layout.png)

在内存布局中，最重要的就是invoke变量，这是一个函数指针，指向Block的实现代码，函数原型至少要接受一个void*型的参数，此参数代表块。

descriptor变量是指向结构体的指针，每个块里都包含此结构体，其中声明了块对象的总体大小，还声明了copy与dispose这两个辅助函数所对应的函数指针。辅助函数在拷贝及丢弃块对象时运行，其中会执行一些操作，比方说，前者要保留捕获的对象，而后者则将之释放。  

块还会把它所捕获的所有变量都拷贝一份。这些拷贝放在descriptor变量后面，捕获了多少个变量，就要占据多少内存空间。请注意，拷贝的并不是对象本身，而是指向这些对象的指针变量。invoke函数为何需要把块对象作为参数传进来呢？原因就在于，执行块时，要从内存中把这些捕获到的变量读出来。

* 全局Block  

```
    void (^globalBlock)(void) = ^{
        NSLog(@"This is a block");
    };
    NSLog(@"global block is kind of class: %@", [globalBlock class]);
```
输出：global block is kind of class: __NSGlobalBlock__

上面定义的globalBlock，因为没有捕捉任何状态（比如外围的变量等），运行时也无须有状态来参与。块所使用的整个内存区域，在编译期已经完全确定了，因此，全局块可以声明在全局内存里，而不需要在每次用到的时候于栈中创建。这种块实际上相当于单例。

* 堆上的Block

```
    void (^block)(void) = ^{
            NSLog(@"self = %@", self);
        };
    block();
    NSLog(@"block is kind of class: %@", [block class]);
```
输出：block is kind of class: __NSMallocBlock__  
上面定义的block，因为在内部捕捉了self，就从全局Block变成了堆上的Block。

* 栈上的Block   
Block在定义的时候，其所占的内存区域是分配在栈中的。  

```
CGFloat f = 1.1;
NSLog(@"%@", ^{NSLog(@"%lf",f);});
NSLog(@"%@",[^{NSLog(@"%lf",f);} copy]);
void(^deliveryBlock)(void) = ^{NSLog(@"%lf",f);};
NSLog(@"%@", deliveryBlock);
```
经打印得知，只有第一次打印输出类型为__NSStackBlock__, 其它的两次都变成了__NSMallocBlock__。所以栈上的Block，在ARC环境下，一经赋值，copy，参数传递，就都变成了堆上的Block。  


### 阴影的渲染为什么慢？Instruments在View渲染优化中的使用。
如果只是简单的设置了shadowColor, shadowOffset, shadowOpacity等属性，那么Core Animation就得自己去计算阴影的形状。这会导致在渲染的时候，从GPU切换到CPU去计算阴影的形状，计算完成之后再切换回GPU。这种现象叫离屏渲染，降低性能。
可以利用Simulator或Instruments去测试Core Animation的性能。如：  
Color Blended Layers  
Color Copied Images  
Color Misaligned Images
Color Off-screen Rendered

### KVC的原理
Key-Value Coding:   
通过key来访问一个对象的属性或是实例变量的值，而不是通过具体的访问方法。  
直接或者间接继承自NSObject的类，都拥有这个能力，是因为NSObject类中提供了缺省的实现。  
实现的原理就是，将key映射到真正的访问方法。  
例如Setter方法的寻找：  

* 找名字为set<Key>:或_set<Key>的方法，如果找到，调用之。  
* 如果第1步没有找到，并且类方法accessInstanceVariablesDirectly返回YES(默认)，那么会按`_<Key>, _is<Key>, <Key>, is<Key>`的顺序去寻找实例变量。如果找到了，就将值直接设置到实例变量上。  
* 如果最后没有找到，会去调用setValue:forUndefinedKey:。默认的实现是抛出异常，但是子类可以提供不同的行为。

### KVO背后的原理
原理：  
假设self.man为Person类的对象，有如下的注册观察者的代码：  

```
[self.man addObserver: self forKeyPath: @"name" options: kNilOptions context: NULL];
```
如果我们在这一行打上断点，当断点停下来的时候，在控制台里运行```po self.man->isa```，那么会打印NSKVONotifying_Person，可以发现对象的isa指针被指向了一个运行期创建的类。但是运行```po [self.man class]```，却还是显示的Person。
   
使用运行时的class_copyMethodList方法，获取并打印临时类的方法列表，会打印出以下：  
setName:  
class  
dealloc  
_isKVOA  

背后的实现原理：   
当类Person的对象被观察时，KVO在运行时动态创建了一个Person类的子类：NSKVONotifying_Person，并为这个新的子类重写了被观察属性keyPath的setter方法。子类的setter方法会去负责通知观察者对象属性的改变。被观察的实例对象的isa指针被修改为指向新创建的子类。   
子类中的setter方法的工作原理：

```
-(void)setName:(NSString *)newName{ 
    [self willChangeValueForKey:@"name"];    //KVO 在调用存取方法之前总调用 
    [super setValue:newName forKey:@"name"]; //调用父类的存取方法 
    [self didChangeValueForKey:@"name"];     //KVO 在调用存取方法之后总调用
}
```

集合类型的监听：
假设有一个属性是可变数组类型，通过常规的方法，只能监听到数组被赋值的时候。如果想监听到里面的值被添加，删除，那么就得依赖于手动的触发KVO了。

### 如何手动的触发KVO？
为了减少不必要的通知，或是为了将几个变化归为一组一起发出通知，可以覆写下面的类方法来告诉KVO，哪些key使用自动的触发。


```
+ (BOOL)automaticallyNotifiesObserversForKey:(NSString *)theKey {
 
    BOOL automatic = NO;
    if ([theKey isEqualToString:@"balance"]) {
        automatic = NO;
    }
    else {
        automatic = [super automaticallyNotifiesObserversForKey:theKey];
    }
    return automatic;
}
```

非集合类型的属性，比较简单，只用指定变化的key。

```
///Testing the value for change before providing notification  
- (void)setBalance:(double)theBalance {
    if (theBalance != _balance) {
        [self willChangeValueForKey:@"balance"];
        _balance = theBalance;
        [self didChangeValueForKey:@"balance"];
    }
}
```

对于一个to-many类型的属性，不但要指定变化的key，还要指定变化的类型和所涉及到的对象的索引集。

```
//Implementation of manual observer notification in a to-many relationship  
- (void)removeTransactionsAtIndexes:(NSIndexSet *)indexes {
    [self willChange:NSKeyValueChangeRemoval
        valuesAtIndexes:indexes forKey:@"transactions"];
 
    // Remove the transaction objects at the specified indexes.
 
    [self didChange:NSKeyValueChangeRemoval
        valuesAtIndexes:indexes forKey:@"transactions"];
}
```

### 传值的几种方式？各自的使用场景是什么？
* 通知：  
使用Notification，可以在发送通知者和观察者之间解耦，观察者只需要知道通知的名字，以及通知中携带的参数即可。传值是通过userInfo来传一个字典。
* 代理：  
此种情况下，一般是A持有B，然后A遵从B制定的协议，实现代理方法。在代理方法中，可以自由的传值。  
* block:  
使用block，可以实现两种场景：回调；调用传入的block，来实现自身的逻辑。
   
* selector:  
例如UIGestureRecognizer, UIControl都有addTarget:action:之类的方法，支持添加多于一个的target, action，在需要时通过[target persormSelector]来回调。  

### 简述一个进程的内存分区情况  
1)代码区：存放函数二进制代码
2)数据区：系统运行时申请内存并初始化，系统退出时由系统释放。存放全局变量、静态变量、常量
3)堆区：通过malloc等函数或new等操作符动态申请得到，需程序员手动申请和释放
4)栈区：函数模块内申请，函数结束时由系统自动释放。存放局部变量、函数参数

### NSUserDefaults都可以存储哪些数据类型？
float, double, iteger, boolean, URL, NSData, NSString, NSNumber, NSDate, NSArray, NSDictionary.

### CALayer和UIView
* 为什么会有CALayer? 为了代码复用，在它之上，分别是NSView和UIView。  
* UIView持有一个CALayer的实例，并实现了CALayerDelegate协议，CALayer的内容需要重新加载时，通过-drawLayer:inContext:或-displayLayer:方法，要求UIView提供内容。CALayer之后去管理这个内容。当然CALayer也有自己的一些可视属性。  
* Core Animation是负责图形渲染和动画的框架。自身并不是一个绘制系统，它负责合成和操作一个应用的显示内容，并且利用GPU来实现硬件加速。  
* 很多属性，如frame, position，在UIView层面上修改，也会作用于CALayer上。从这一点儿来看，UIView象是一个CALayer的包装。  
* UIView还用来处理事件，参与职责链，一个APP并不能只用CALayer来去构建。  
* 普通的视图的layer都是CALayer的实例，但可以通过覆写+(Class)layerClass这个方法，改变layer的类型。    
* 树形结构：和UIView的树形结构类似，layer也有相应的树形结构。也可以在不添加视图的情况下，往layer上再添加子layer。  
* 动画：没有与UIView相关联的layer，修改属性值时，会带有隐式动画。UIView自带的那个layer，隐式动画被关闭了，需要通过UIView的动画block，或是Core Animation来做动画。
* 坐标系统：CALayer相比UIView多了一个anchorPoint属性，使用CGPoint结构表示，值域是[0 1]。这个点是各种图形变换的中心点，同时会决定layer的position的位置，缺省值是[0.5,0.5]。  

### CALayer的三个树状结构
不是指三个树状层级，而是指三个layer对象的集合。  

* model layer tree     
  这个集合里面的对象，存储了动画的目标值。平常我们都是和这里的对象打交道。  
* presentation tree   
  这个集合里面的对象，存储的是动画的执行过程中的即时值。  
* render tree   
  这个集合里面的对象，执行实际的动画，是核心动画的私有的。

### CoreGraphics和CoreAnimation的区别和内容
从架构上来说，Core Graphics位于Core Animation的下方，如图所示：  
![架构](https://github.com/buptwsgprivate/iOSInterview/blob/master/Images/ca_architecture_2x.png)  

Core Graphics也叫Quartz 2D, 是一个先进的，二维绘图引擎，可以工作于iOS, tvOS, macOS。
三个核心概念：  
上下文： 主要用于描述图形写入哪里  
路径：是在图层上绘制的内容  
状态：  用于保存配置变换的值，填充和alpha值等。  

个人的理解：UIView会去利用Core Graphics去绘制，绘制好的内容交给Core Animation去做渲染。  

### 说说UIScrollView的原理
scroll view的大小固定，但它的content view可以是任意的尺寸，contentSize属性表明了内容视图的大小。它上面添加了好几种手势，用来滚动内容视图。
感觉这个题用来考新手的吧。  

### UIWebView中注入JS时，获取JSContext的正确方法是什么？
有一个简单的办法，可以拿到一个webview的JSContext:   

```
JSContext *context = [self.webView valueForKeyPath:@"documentView.webView.mainFrame.javaScriptContext"];
```
即使我们在viewDidLoad和webViewDidFinishLoad方法中，利用上述方法获取JSContext，然后向其中注入Native Object，在某些情况下仍然会出问题：当在html进行页面跳转的时候，JS调用OC对象出现undefined.   
webViewDidFinishLoad这个方法是在web的window.onload以后才调用（也就是html所有的资源已经加载和渲染结束后）,这就明了了，在JS调用OC的对象时，还没有注入。   
从[这篇文章](http://www.codertian.com/2016/04/22/iOS-javascriptcore-call-native/)里，可以学习到正确的方法：  

* 为NSObject添加一个分类，并实现下面的方法  

```
@implementation NSObject (JSTest)
- (void)webView:(id)unuse didCreateJavaScriptContext:(JSContext *)ctx forFrame:(id)frame {
    [[NSNotificationCenter defaultCenter] postNotificationName:@"DidCreateContextNotification" object:ctx];
}
@end
```
调用这个方法的时候WebKit就已经创建了JSContext对象，在这个方法中我们发出一个通知，这个通知会把获取到的JSContext环境对象传递出去。

* 在WebViewController的viewDidLoad方法中观察通知  

```
- (void)viewDidLoad {
    [super viewDidLoad];
    [[NSNotificationCenter defaultCenter] addObserver:self selector:@selector(didCreateJSContext:) name:@"DidCreateContextNotification" object:nil];
    
    NSURL *url = [[NSBundle mainBundle] URLForResource:@"test" withExtension:@"html"];
    [self.webView loadRequest:[NSURLRequest requestWithURL:url]];
}
```

* 在通知回调函数中判断JSContext对象是不是当前WebView的  

```
- (void)didCreateJSContext:(NSNotification *)notification {
    NSString *indentifier = [NSString stringWithFormat:@"indentifier%lud", (unsigned long)self.webView.hash];
    NSString *indentifierJS = [NSString stringWithFormat:@"var %@ = '%@'", indentifier, indentifier];
    [self.webView stringByEvaluatingJavaScriptFromString:indentifierJS];
    JSContext *context = notification.object;
    //判断这个context是否属于当前这个webView
    if (![context[indentifier].toString isEqualToString:indentifier]) return;
    
    _context = context;		//如果属于这个webView
    MyJSObject *jsObject = [MyJSObject new];
    _context[@"ttf"] = jsObject;	//将对象注入这个context中
}
```

### 数据库的数据迁移场景及实现
* 在Core Data打开一个store的时候，如果老版本的格式，与新的模型不兼容，那么需要执行数据迁移，如果不执行，那么会发生崩溃。
* Core Data会在store的metadata中存储一些Hash值，利用这些值，Core Data可以迅速的判断当前store的格式，是否与用来打开store的模型相匹配。  
* Core Data也允许针对Entity或是Attribute设置一个hash modifier，来强制Hash值发生改变。一个例子：只是改变了一个属性的内部格式。
* .xcdatamodeld文件实际上是一个文件包，里面包含了不同版本的model，每个由.xcdatamodel和Info.plist文件组成。在Xcode里，我们可以指定当前要使用哪个版本。
* 在许多情况下，Core Data都可以执行自动的数据迁移，即lightweight migration。 Core Data基于源和目的managed object model的不同，会去推断出一个mapping model。  
* 如果对一个Entity或是attribute进行了重新的命名，应该在新版本的model中，设置对应的renaming identifier。这样Core Data就可以知道新版本中的名字，对应的是旧版本中的哪个Entity或是attribute。  
* 要想使用轻量级迁移，除了Core Data能够推断出mapping model之外，我们还应该请求Core Data去执行迁移。关键点就在于在调用addPersistentStoreWithType:configuration:URL:options:error: 方法的时候，传入两个选项：NSMigratePersistentStoresAutomaticallyOption和 NSInferMappingModelAutomaticallyOption, 值均设为YES。 代码如下：   

```
NSError *error = nil;
NSURL *storeURL = <#The URL of a persistent store#>;
NSPersistentStoreCoordinator *psc = <#The coordinator#>;
NSDictionary *options = [NSDictionary dictionaryWithObjectsAndKeys:
    [NSNumber numberWithBool:YES], NSMigratePersistentStoresAutomaticallyOption,
    [NSNumber numberWithBool:YES], NSInferMappingModelAutomaticallyOption, nil];
 
BOOL success = [psc addPersistentStoreWithType:<#Store type#>
                    configuration:<#Configuration or nil#> URL:storeURL
                    options:options error:&error];
if (!success) {
    // Handle the error.
}
```
* 要想使用轻量级迁移，Core Data必须在运行期能够找到源和目的managed object models。默认是在NSBundle的allBundles和allFrameworks方法返回的位置。如果模型被存储在别的位置，那么我们就得自己去写代码产生mapping model，然后使用migration manager去发起迁移。示例代码如下：  

```
- (BOOL)migrateStore:(NSURL *)storeURL toVersionTwoStore:(NSURL *)dstStoreURL error:(NSError **)outError {
 
    // Try to get an inferred mapping model.
    NSMappingModel *mappingModel =
        [NSMappingModel inferredMappingModelForSourceModel:[self sourceModel]
                        destinationModel:[self destinationModel] error:outError];
 
    // If Core Data cannot create an inferred mapping model, return NO.
    if (!mappingModel) {
        return NO;
    }
 
    // Create a migration manager to perform the migration.
    NSMigrationManager *manager = [[NSMigrationManager alloc]
        initWithSourceModel:[self sourceModel] destinationModel:[self destinationModel]];
 
    BOOL success = [manager migrateStoreFromURL:storeURL type:NSSQLiteStoreType
        options:nil withMappingModel:mappingModel toDestinationURL:dstStoreURL
        destinationType:NSSQLiteStoreType destinationOptions:nil error:outError];
 
    return success;
}
```

* 如果Core Data不能推断出Mapping model，那么就麻烦了，需要自己去定义。不过这个一般用不到，用到时再去看文档就可以。  

### 用户感觉卡顿后, 如何系统分析卡顿的原因？
卡顿监控的实现一般有两种方案：  
（1）主线程卡顿监控。通过子线程监测主线程的runLoop，判断两个状态区域之间的耗时是否达到一定阈值。具体原理和实现，[这篇文章](http://www.tanhao.me/code/151113.html/)介绍得比较详细。  
（2）FPS监控。要保持流畅的UI交互，APP刷新率应当努力保持在60FPS。监控实现原理比较简单，通过记录两次刷新时间间隔，就可以计算出当前的FPS。  
微信读书团队在实际应用过程中，发现上面两种方案，抖动都比较大。因此提出了一套综合的判断方法，结合了主线程监控，FPS监控，以及CPU使用率等指标，作为判断卡顿的标准。  

![卡顿分析](https://github.com/buptwsgprivate/iOSInterview/blob/master/Images/wechat-stuck.jpeg)  

iOS卡顿监测分析: http://blog.csdn.net/ycm1101743158/article/details/77508924  
简单监测iOS卡顿的demo: http://blog.csdn.net/game3108/article/details/51147946

### 什么是长连接？有没有优化方案？
TCP连接在长时间没有数据传输的时候，会断开连接。为了实现长连接，就要定期的发送心跳数据。所谓的长连接并没有什么高深的地方，就是想办法让一个TCP连接长时间的保持。  

心跳包，通常是客户端每隔一小段时间向服务器发送的一个数据包，通知服务器自己仍然在线，并传输一些可能有必要的数据。因按照一定的时间间隔发送，类似于心跳，所以叫做心跳包。事实上为了保持长连接，至于包的内容，是没有特别规定的，不过一般都是很小的包，或者只是包含包头的一个空包。  

在TCP协议的机制里面，本身是存在有心跳包机制的，也就是TCP协议中的SO_KEEPALIVE，系统默认是设置2小时的心跳频率。要用setsockopt将SOL_SOCKET.SO_KEEPALIVE设置为1才是打开，并且可以设置三个参数tcp_keepalive_time/tcp_keepalive_probes/tcp_keepalive_intvl，分别表示连接闲置多久开始发keepalive的ACK包、发几个ACK包不回复才当对方死了、两个ACK包之间间隔多长。  

至于优化方案，这个通过Google搜索不到。  

### 多线程下载文件实现方案？要能够支持暂停，重新开始。

### GPU和CPU是如何协同工作的？
关于两者的协同工作，[iOS核心动画高级技巧](https://zsisme.gitbooks.io/ios-/content/index.html)这本书里有讲，在第12章的第1节。里面讲述了渲染所涉及的6个阶段，只有最后一个阶段是由GPU执行的，并且只有前两个阶段是开发者可控的。但是就是在这前两个阶段，我们可以决定哪些由CPU执行，哪些交给GPU去执行。 

这本书其实可以系统的一读，并加深学习。  

### TCP的三次握手是什么？
这张图描述了三次握手的过程：  
![三次握手](https://github.com/buptwsgprivate/iOSInterview/blob/master/Images/TCPHandshake.png)

第一次握手：建立连接。客户端发送连接请求报文段，将SYN位置为1，Sequence Number为x；然后，客户端进入SYN_SEND状态，等待服务器的确认；  
第二次握手：服务器收到SYN报文段。服务器收到客户端的SYN报文段，需要对这个SYN报文段进行确认，设置Acknowledgment Number为x+1(Sequence Number+1)；同时，自己自己还要发送SYN请求信息，将SYN位置为1，Sequence Number为y；服务器端将上述所有信息放到一个报文段（即SYN+ACK报文段）中，一并发送给客户端，此时服务器进入SYN_RECV状态；  
第三次握手：客户端收到服务器的SYN+ACK报文段。然后将Acknowledgment Number设置为y+1，向服务器发送ACK报文段，这个报文段发送完毕以后，客户端和服务器端都进入ESTABLISHED状态，完成TCP三次握手。

SYN, ACK都是TCP报文头部中的标志位，为1表示被置。  
三次握手的本质：  
这个问题的本质是, 信道不可靠, 但是通信双发需要就某个问题达成一致. 而要解决这个问题, 无论你在消息中包含什么信息, 三次通信是理论上的最小值. 所以三次握手不是TCP本身的要求, 而是为了满足"在不可靠信道上可靠地传输信息"这一需求所导致的. 请注意这里的本质需求,信道不可靠, 数据传输要可靠. 三次达到了, 那后面你想接着握手也好, 发数据也好, 跟进行可靠信息传输的需求就没关系了. 因此,如果信道是可靠的, 即无论什么时候发出消息, 对方一定能收到, 或者你不关心是否要保证对方收到你的消息, 那就能像UDP那样直接发送消息就可以了.

“三次握手”的目的是“为了防止已失效的连接请求报文段突然又传送到了服务端，因而产生错误”。“已失效的连接请求报文段”的产生在这样一种情况下：client发出的第一个连接请求报文段并没有丢失，而是在某个网络结点长时间的滞留了，以致延误到连接释放以后的某个时间才到达server。本来这是一个早已失效的报文段。但server收到此失效的连接请求报文段后，就误认为是client再次发出的一个新的连接请求。于是就向client发出确认报文段，同意建立连接。假设不采用“三次握手”，那么只要server发出确认，新的连接就建立了。由于现在client并没有发出建立连接的请求，因此不会理睬server的确认，也不会向server发送数据。但server却以为新的运输连接已经建立，并一直等待client发来数据。这样，server的很多资源就白白浪费掉了。采用“三次握手”的办法可以防止上述现象发生。例如刚才那种情况，client不会向server的确认发出确认。server由于收不到确认，就知道client并没有要求建立连接。”

### 如何防止网络劫持的发生？
[iOS 客户端对于运营商劫持的一点点对抗方式](https://segmentfault.com/a/1190000009049544)  
[NSURLProtocol：DNS劫持和Web资源本地化](http://www.qingpingshan.com/rjbc/ios/167550.html)  
[可能是最全的iOS端HttpDns集成方案](http://dev.dafan.info/detail/378770?p=)  
[移动开发构架漫谈——反劫持实战篇](http://blog.csdn.net/shaobo8910/article/details/46953007)  
[iOS强制ATS后,DNS劫持问题如何解决?](http://www.itdadao.com/articles/c15a1171332p0.html)  

### 网络优化方案都有哪些？

### 电量优化方案都有哪些？

## 算法相关
### 反转二叉树，非递归

### M个红球和N个黑球排序，有多少种排法？

