# iOS面试题汇总

## 开篇
### 做个自我介绍吧。
教育经历：本科在重庆邮电大学，硕士在北京邮电大学，都是计算机专业。  

工作经历：早期一直使用C/C++语言，先后工作于威盛电子，索尼爱立信，诺基亚公司，从事移动端APP开发。在诺基亚裁员后，从2012年开始转型iOS平台，作为合伙人创办北京龙景科技，开发iOS平台上的应用和游戏项目。2015年末公司关闭后，于2016年初来到花椒直播，作为高级iOS，参与APP的研发。  

项目经历：在花椒，独立开发了花椒监控APP，参与很多业务功能的开发，直播间内外都有。另外还开发了防崩溃，卡顿分析，防止NSTimer的循环引用，以及一些基础组件。  

目前具备的技能：截止目前，掌握了开发iOS APP所需的从账号申请，开发，内测到上线的各个环节，包括三方库管理，持续集成，内存泄露检测，崩溃收集，防崩溃，性能优化，日志收集等。  

### 接下来几年，你的职业规划是什么？
找一家发展的好的公司  
大公司可以走技术路线  
小公司尝试带团队的职位  

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
如果只是简单的设置了shadowColor, shadowOffset, shadowOpacity等属性，那么Core Animation就得自己去计算阴影的形状。这会导致在渲染的时候，发生离屏渲染，降低性能。
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

* 找名字为`set<Key>:`或`_set<Key>`的方法，如果找到，调用之。  
* 如果第1步没有找到，并且类方法accessInstanceVariablesDirectly返回YES(默认)，那么会按`_<Key>, _is<Key>, <Key>, is<Key>`的顺序去寻找实例变量。如果找到了，就将值直接设置到实例变量上。  
* 如果最后没有找到，会去调用`setValue:forUndefinedKey:`。默认的实现是抛出异常，但是子类可以提供不同的行为。

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
实现思路：NSRunLoop调用方法主要就是在kCFRunLoopBeforeSources和kCFRunLoopBeforeWaiting之间,还有kCFRunLoopAfterWaiting之后,也就是如果我们发现这两个时间内耗时太长,那么就可以判定出此时主线程卡顿. 要监控NSRunLoop的状态，需要添加观察者。  
当检测到了卡顿，下一步需要做的就是记录卡顿的现场，即此时程序的堆栈调用，可以借助开源库 PLCrashReporter 来实现。   

在xcode里运行APP时，可以通过运行script，调用xcrun dsymutil工具，产生符号信息文件。有了符号文件，再加上崩溃日志，就可以解析出完整的调用栈。    
（2）FPS监控。要保持流畅的UI交互，APP刷新率应当努力保持在60FPS。监控实现原理比较简单，通过记录两次刷新时间间隔，就可以计算出当前的FPS。  
微信读书团队在实际应用过程中，发现上面两种方案，抖动都比较大。因此提出了一套综合的判断方法，结合了主线程监控，FPS监控，以及CPU使用率等指标，作为判断卡顿的标准。  

![卡顿分析](https://github.com/buptwsgprivate/iOSInterview/blob/master/Images/wechat-stuck.jpeg)  

[iOS实时卡顿监控](http://www.tanhao.me/code/151113.html/)  
[微信iOS卡顿监控系统](https://mp.weixin.qq.com/s?__biz=MzAwNDY1ODY2OQ==&mid=207890859&idx=1&sn=e98dd604cdb854e7a5808d2072c29162&scene=4#wechat_redirect)  
[调研和整理](https://github.com/aozhimin/iOS-Monitor-Platform)   
[简单监测iOS卡顿的demo](http://blog.csdn.net/game3108/article/details/51147946)  

如果想在线上产品中加入监控系统，有些问题是需要考虑的：  
对客户手机的性能影响(运行速度，流量)，流量影响，磁盘占用影响。  
对服务器的压力  

一般公司可以使用大厂的APM产品，大厂一般自己研发。  
### 如何检测后台线程中更新UI？
从Xcode9开始，诊断选项里有个叫"Main Thread checker"的，默认是打开的，在程序运行期间，如果检测到了主线程之外的线程中更新UI，那么会在控制台中打出警告。但问题是，很多开发者选择无视，需要依赖于开发者的自觉，才能避免之类的问题。  

也可以自己去实现一套机制，原理是通过hook UIView的-setNeedsLayout, -setNeedsDisplay, -setNeedsDisplayInRect三个方法，确保它们都是在主线程中执行。如果不是，那么让程序发生崩溃，可以强制开发者去修改。  

### 有没有什么办法能够防止crash?
可以看看这两篇文章：  
[网易iOS App运行时Crash自动防护实践](https://mp.weixin.qq.com/s?__biz=MzA3ODg4MDk0Ng==&mid=2651113088&idx=1&sn=10b28d7fbcdf0def1a47113e5505728d&chksm=844c6f5db33be64b57fc9b4013cdbb7122d7791f3bbca9b4b29e80cce295eb341b03a9fc0f2f&mpshare=1&scene=23&srcid=0207njmGS2HWHI0BZmpjNKhv%23rd)  
[XXShield实现防止iOS APP Crash和捕获异常状态下的崩溃信息](http://java.ctolib.com/ValiantCat-XXShield.html)  

unrecognized selector: 
可以利用运行时的消息转发机制，重写forwardingTargetForSelector方法，做以下几步的处理：  

* 为桩类添加相应的方法，方法的实现是一个具有可变数量参数的C函数
* 该C函数只是简单的返回0，类似于返回nil   
* 将消息直接转发到这个桩类对象上。  

KVO：  
容易出现崩溃的地方：忘记了移除观察者；没有添加就去移除；重复添加后导致添加/移除次数不匹配；

定时器：  
由于定时器对target进行了强引用，容易造成循环引用，一是造成内存不释放，二是不释放的对象在定时器被触发时执行代码，很有可能导致崩溃。  
使用weak proxy解决。  

其实我们自己的safe cast宏也是可以防止一些崩溃的。    

### 多线程下载文件实现方案？要能够支持暂停，重新开始。

### GPU和CPU是如何协同工作的？
关于两者的协同工作，[iOS核心动画高级技巧](https://zsisme.gitbooks.io/ios-/content/index.html)这本书里有讲，在第12章的第1节。里面讲述了渲染所涉及的6个阶段，只有最后一个阶段是由GPU执行的，并且只有前两个阶段是开发者可控的。但是就是在这前两个阶段，我们可以决定哪些由CPU执行，哪些交给GPU去执行。 

这本书其实可以系统的一读，并加深学习。  

### 如何创建CocoaPods私有仓库？
可以看下面两篇文章：  
[如何创建私有 CocoaPods 仓库](http://www.jianshu.com/p/ddc2490bff9f)  
[CocoaPods创建私有Pods](http://www.liuchungui.com/blog/2015/10/19/cocoapodschuang-jian-si-you-pods/)  

创建步骤总结：  
1. 创建一个spec仓库，用来存放私有仓库的spec文件  
2. 将这个私有的spec仓库，添加到CocoaPods
   ```pod repo add REPO_NAME SOURCE_URL```   
3. 生成代码库的spec文件，打tag，并push到私有spec仓库
   ```pod repo push REPO_NAME SPEC_NAME.podspec```   
4. 使用的时候，要在Podfile文件中同时添加本地私有源和官方源。如果没有添加本地私有源，它默认是用官方的repo，这样找不到本地的Pod；如果只是设置了本地私有源，就不会再去官方源中查找。
   
### 什么是组件化？如何实施呢？

[iOS 组件化方案探索](https://wereadteam.github.io/2016/03/19/iOS-Component/)  
[CTMediator](https://github.com/casatwy/CTMediator)  
[在现有工程中实施基于CTMediator的组件化方案](https://casatwy.com/modulization_in_action.html)  

然后，[iOS组件化方案调研](http://www.jianshu.com/p/34f23b694412)里有收集很多参考资料，有空时要系统的阅读。  

读了[在现有工程中实施基于CTMediator的组件化方案](https://casatwy.com/modulization_in_action.html)一文后，有下述问题：  
1) 可否将A_Category和B_Category合并为一个Pod?  
答：不能，除非这两个pod彼此不能拆解。但是如果彼此不能拆解的话，就不应该出现两个category，应该只有一个才合理。  
2）B_Category提供的服务和Target_B提供的服务是一样的，能不能把B_Category直接放到B的模块里替代Target_B？   
答：不要这么做。category是业务无关的，target是业务相关的。外部只会通过category来调用功能，这样能够达到两个效果：1.某组件即使缺少依赖，也能编译通过。2.在Podfile中添加被依赖的组件后，不用修改任何代码，就能够正常调用被依赖的组件的功能。  
这样才能做到完全解耦。  
3) 组件划分的标准是什么？  
答：考虑两点，1.这一部分是否能够自治，且相对独立。2.这一部分是否会被多个调用者调度。只要这二者有一个条件满足，那就会把这部分组件化出来，即使它只有一个对象不到50行代码。  

组件化更多的是针对横向依赖做依赖下沉。业务相对于服务之间的依赖属于纵向依赖，把服务作为普通Pod引入即可。业务和业务之间是横向依赖，必须组件化。  

4) 工程中的公用图片资源在组件化时应该怎么处理呢？
答：我们是单独一个子工程，里面只有图片。然后在其它使用图片的子工程里，只把这个图片子工程写入Podfile，而不写入podspec的dependency里面，这样能确保调试的时候有图片，组件发版的时候不带图片。然后在主工程的Podfile里面同样也写入图片子工程，这样就好了。

5) 组件的负责团队一般就只工作在组件工程里，里面除了组件自身，还有单元测试代码，一般保证自己工作正常就可以了。需要联调时，可以在Podfile里引入依赖的组件。

### 基于CTMediator的组件化方案，有哪些核心组成？
假如主APP调用某业务A，那么需要以下组成部分：  

* CTMediator类，该类提供了函数 ```- (id)performTarget:(NSString *)targetName action:(NSString *)actionName params:(NSDictionary *)params shouldCacheTarget:(BOOL)shouldCacheTarget;```  
这个函数可以根据targetName生成对象，根据actionName构造selector，然后可以利用performSelector:withObject:方法，在目标上执行动作。  

* 业务A的实现代码，另外要加一个专门的类，用于执行Target Action  
  类的名字的格式：`Target_%@`，这里就是Target_A。  
  这个类里面的方法，名字都以`Action_`开头，需要传参数时，都统一以NSDictionary*的形式传入。  
  CTMediator类会创建Target类的对象，并在对象上执行方法。  

* 业务A的CTMediator扩展  
  扩展里声明了所有A业务的对外接口，参数明确，这样外部调用者可以很容易理解如何调用接口。  
  在扩展的实现里，对Target, Action需要通过硬编码进行指定。由于扩展的负责方和业务的负责方是相同的，所以这个不是问题。      
  
### iOS11带来了哪些新特性？有哪些你觉得以后可能成为热点？  

### 什么情况下使用H5做页面比较合适？  
Native的缺点：  
App的发版周期偏长，有时无法跟上产品的更新节奏  
灵活性差，如果有圈套的方案变更，需要发版才能解决  
如果存在bug，无法在当前版本进行修复  
需要根据不同的平台写不同的代码  

H5的优点：  
页面可以实时在服务端进行修改，灵活性很强，避免了Native发版周期带来的时间成本  

H5的缺点：  
弱网情况下体验较差  
流量消耗较大  
访问原生系统的能力受到限制  

通常的经验是：对于一些比较稳定的业务，对用户体验要求较高的，可以选择Native开发。而对于一些业务变更比较快，处在不断试水的过程，而且不涉及调用文件系统和硬件调用的业务我们可以选择H5开发。    

### SDWebImage源码阅读笔记

### AFNetworking源码阅读笔记
  
### 性能优化的一些通用思路
个人总结的一些通用的优化思路：   
合并： 将多个操作进行合并，例如draw call, network request   
压缩： 例如纹理的压缩，网络请求响应里使用压缩格式   
延迟： 延迟创建，按需创建。   
对象池：反复使用，不要反复的创建和销毁。   

### 离屏渲染的准确解释？  
图像渲染工作原理

由CPU计算好显示内容，GPU渲染完成后将渲染结果放入帧缓冲区，随后视频控制器会按照 HSync 信号逐行读取帧缓冲区的数据，经过可能的数模转换传递给显示器显示。如下图：  
![图像渲染工作原理](https://github.com/buptwsgprivate/iOSInterview/blob/master/Images/图像渲染工作原理.png)  

屏幕渲染有以下两种方式：  
On-Screen Rendering  
当前屏幕渲染，指的是在当前用于显示的屏幕缓冲区中进行渲染操作。  

Off-Screen Rendering
离屏渲染，指的是GPU或CPU在当前屏幕缓冲区以外新开辟一个缓冲区进行渲染操作。过程中需要切换contexts，先从当前屏幕切换到离屏的contexts，渲染结束后，又要将contexts切换回来，而切换过程十分耗费性能。  

### 降低崩溃率的系统级方案  
不知道业界有没有这样的系统级方案存在，但我觉得可以尝试这样向面试官回答：  
消除内存泄露  
消除后台线程更新UI      
使用安全转换宏  
判断一个对象的类是不是期望的类型   
使用断言，在开发期间尽量多的暴露出问题  
通过Xcode的静态分析工具，查找出代码中的隐患  
通过XCode的诊断工具，检测出不容易暴露的问题  
最佳实践：在析构或其它合适的时机，将delegate或datasource置为nil，会比较安全   
在最后，可以使用防崩溃大招: unrecognized selector, KVO  

在事后，通过崩溃收集系统，收集崩溃日志，修复崩溃。

### 如何实现一个线程安全的NSMutableArray
可以从NSMutableArray派生，但要根据苹果的文档，实现那些所谓的Primitive Methods。
在这些方法的内部，加线程同步机制，比如锁。  

### NSTimer, CADisplayLink, `dispatch_source_t`，高精度定时器
下述内容摘自博客文章：[更可靠和高精度的 iOS 定时器](http://blog.lessfun.com/blog/2016/08/05/reliable-timer-in-ios/)  
* NSTimer  
  可靠性：  
  NSTimer不可靠，其所在的RunLoop 会定时检测是否可以触发 NSTimer 的事件，但由于 iOS 有多个 RunLoop 的运行模式，如果被切到另一个 run loop，NSTimer 就不会被触发。每个 RunLoop 的循环间隔也无法保证，当某个任务耗时比较久，RunLoop 的下一个消息处理就只能顺延，导致 NSTimer 的时间已经到达，但 Runloop 却无法及时触发 NSTimer，导致该时间点的回调被错过。

  最小精度：  
  理论上最小精度为0.1毫秒，不过由于受RunLoop的影响，会有50~100毫秒的误差，所以，实际精度可以认为是0.1秒。  
  
* CADisplayLink  
  也可以用作定时器，其调用间隔与屏幕刷新频率一致，也就是每秒60帧，间隔16.67ms。与NSTimer类似，如果在两次屏幕刷新之间执行了一个比较耗时的任务，其中的某一帧就会被跳过，造成UI卡顿。  
  
   可靠性：  
   如果执行的任务很耗时，也会导致回调被错过，所以并不十分可靠。但是，假如调用者能够确保任务能够在最小时间间隔内执行完成，CADisplayLink就比较可靠，因为屏幕的刷新频率是固定的。  
   
   最小精度：  
   受限于每秒60帧的屏幕刷新频率，注定CADisplayLink的最小精度为16.67毫秒。误差在1毫秒左右。  
   
* `dispatch_source_t`  
  精度很高，因为API接口中设置的都是纳秒的单位。但是当遇到比较耗时的运算任务的时候，就无法保证了。

* 更高精度的定时器   
  上述的各种定时器，都受限于苹果为了保护电池和提高性能采用的策略，导致无法实时地执行回调。并且前述的定时器的核心代码实际上是一样的。如果你的确需要使用更高精度的定时器，官方也是提供了方法。  
  而有别于普通定时器的高精度定时器，则是基于高优先级的线程调度类创建的定时器，在没有多线程冲突的情况下，这类定时器的请求会被优先处理。  
  
  实现方法：  
  把定时器所在的线程，移到高优先级的线程调度类。  
  使用更精确的计时器API：`mach_wait_until`。换言之，你想要10秒后执行，就绝对在10秒后执行，绝不提前，也不延迟。  
  
  最小精度： 小于0.5毫秒。  
  
###  关于`dispatch_barrier_async`函数需要搞清楚的一点
>The queue you specify should be a concurrent queue that you create yourself using the dispatch_queue_create function. If the queue you pass to this function is a serial queue or one of the global concurrent queues, this function behaves like the dispatch_async function.   

根据苹果的官方文档，barrier功能只能用在自定义的并发队列中，不能用在默认提供的并发队列中。如果用了，那么barrier功能是不起作用的，退化为普通的dispatch调用。  

### 电量优化方案都有哪些？
官方文档在这里：[Energy Efficiency Guide for iOS Apps](https://developer.apple.com/library/content/documentation/Performance/Conceptual/EnergyGuide-iOS/index.html#//apple_ref/doc/uid/TP40015243)  
一些要点：  
##### 指导原则
让CPU不间歇的做一些零碎的工作，不如集中的做完，这样CPU可以得到休息的机会。这里涉及到dynamic cost和fixed cost的概念，集中的做完的情况下，因为持续时间短，fixed cost会比较低。

##### 定位  
* 在需要定位时调用一次CLLocationManager类的requestLocation方法，这个方法在获取到定位信息后就会关闭定位服务。  
* 不使用时要及时的关闭定位服务。  
* 使用尽可能低的定位精度，只要能满足需要即可。  
* 设置location manager的pausesLocationUpdatesAutomatically和activityType两个属性，可以让location manager做适当的优化。  
* 在后台运行时，允许延期的位置更新。  
* 将定位更新限制在特定的区域或位置。  
* 以上都不适合时，考虑注册Significant-Change Location Updates.  

##### 传感器
* 停止设备方向变化的通知  
  如果当前APP或是界面只会停留在一个方向，可以临时关闭硬件。  
  
  ```
  // Turn on the accelerometer
  [[UIDevice currentDevice] beginGeneratingDeviceOrientationNotifications];
  
   // Turn off the accelerometer
  [[UIDevice currentDevice] endGeneratingDeviceOrientationNotifications];
  ```   
  
* 通过设置更新间隔，降低更新的次数  

##### 蓝牙设备
使用时要注意优化。

##### 高效的使用网络
##### 尽量减少定时器的使用
##### 尽量减少I/O调用

### UITableView有哪些优化的方案？  

### 写一下hitTest函数的实现代码  
```
- (UIView *)hitTest:(CGPoint)point withEvent:(UIEvent *)event {
    if (!self.isUserInteractionEnabled || self.isHidden || self.alpha <= 0.01) {
        return nil;
    }
    if ([self pointInside:point withEvent:event]) {
        for (UIView *subview in [self.subviews reverseObjectEnumerator]) {
            CGPoint convertedPoint = [subview convertPoint:point fromView:self];
            UIView *hitTestView = [subview hitTest:convertedPoint withEvent:event];
            if (hitTestView) {
                return hitTestView;
            }
        }
        return self;
    }
    return nil;
}
```

### dispatch_sync死锁问题（`dispatch_get_current_queue()`为什么被废弃？） 
之前都会说下面的代码会死锁：  

```
    dispatch_sync(dispatch_get_main_queue(), ^{
        NSLog(@"I will dead lock");
    });
```
其实现在在最新的Xcode9下，下面的代码直接就崩溃了，应该是系统改进了，便于发现问题。  

关于`dispatch_get_current_queue`被废弃的问题，是因为之前写下面的代码会导致死锁：  

```
void func(dispatch_queue_t queue, dispatch_block_t block)  
{  
    if (dispatch_get_current_queue() == queue) {  
        block();  
    }else{  
        dispatch_sync(queue, block);  
    }  
}  

- (void)deadLockFunc  
{  
    dispatch_queue_t queueA = dispatch_queue_create("com.yiyaaixuexi.queueA", NULL);  
    dispatch_queue_t queueB = dispatch_queue_create("com.yiyaaixuexi.queueB", NULL);  
    dispatch_sync(queueA, ^{  
        dispatch_sync(queueB, ^{  
            dispatch_block_t block = ^{  
                //do something  
            };  
            func(queueA, block);  
        });  
    });  
}  
```

上面的代码会导致连续两次针对queueA调用dispatch_sync函数，问题在于GCD队列本身是不可重入的。  

那么替代的方案是:  

```
void dispatch_queue_set_specific(dispatch_queue_t queue, const void *key, void *context, dispatch_function_t destructor);
void dispatch_set_target_queue(dispatch_object_t object, dispatch_queue_t queue);

例如如下代码：  
   dispatch_queue_t queueA = dispatch_queue_create("com.yiyaaixuexi.queueA", NULL);  
   dispatch_queue_t queueB = dispatch_queue_create("com.yiyaaixuexi.queueB", NULL);  
   //这行代码是必须的
   dispatch_set_target_queue(queueB, queueA);  
    
   static int specificKey;  
   CFStringRef specificValue = CFSTR("queueA");  
   dispatch_queue_set_specific(queueA,  
                               &specificKey,  
                               (void*)specificValue,  
                               (dispatch_function_t)CFRelease);  
    
   dispatch_sync(queueB, ^{  
       dispatch_block_t block = ^{  
               //do something  
       };  
       
       //当前线程为queueB对应的线程里，但是却能够获取到specific值，就是由于前面设置了queueA为queueB的target queue.
       CFStringRef retrievedValue = dispatch_get_specific(&specificKey);  
       if (retrievedValue) {  
           block();  
       } else {  
           dispatch_sync(queueA, block);  
       }  
   });  
```

### Core Data大量数据多线程同步
1. 搭建多线程环境  
   另外创建NSManagedObjectContext时，指定并发模式为NSPrivateQueueConcurrencyType，这样context会创建并管理一个private queue.  
   应用启动时创建的context，使用的是NSMainQueueConcurrencyType，被关联到了主线程。  
2. 在private queue context中进行操作时，应该使用performBlock:或是performBlockAndWait:方法，这两个方法能够保证操作会在正确的queue中执行。  

3. 多个context同步最简单的方案如下：  

```
NSNotificationCenter.defaultCenter().addObserver(self, 
                                             selector: "backgroundContextDidSave:", 
                                                 name: NSManagedObjectContextDidSaveNotification, 
                                               object: backgroundContext)
 
func backgroundContextDidSave(notification: NSNotification){
    mainContext.performBlock(){
        mainContext.mergeChangesFromContextDidSaveNotification(notification)
    }
}
```
这里要注意的是，注册通知时一定要设置object参数，不要传nil。原因是一些系统框架内部也会使用Core Data，如果不指定产生通知的对象，那么有可能会收到意料之外的通知。  

4. 大量数据的处理  
  大量数据意味着需要我们关注内存占用和性能，写代码时需要刻如下规则：  
  1）尽可能缓存需要的数据，不相关的数据保持faults状态。  
  2）fetch时尽可能精准，少引入不相关的数据。  
  3）构建多context时尽量将同类managed object集中，最大限度减少合并需求。  
  4）提升操作效率，对Asynchronous Fetch, Batch Update, Batch Delete等新特性尽可能利用。

### 常见的加密算法？对称加密和非对称加密的区别。  
对称加密：  
这类算法在加密和解决时使用相同的密钥  
常见的对称加密算法有：DES, 3DES, AES, Blowfish, IDEA, RC5, RC6  
特点：加密解密效率高，速度快，适合进行大数据量的加解密。  

非对称加密：  
非对称的加密算法，需要用到两个密钥：公钥和私钥。公钥对外公开，但并不会危害到另外一个的秘密性质。加密用公钥，解密用私钥。  
常见的非对称加密算法有：RSA, DSA, ECC  
特点：算法复杂，加解密速度慢，但安全性高，一般与对称加密结合使用（对称加密对内容加密，非对称加密对所使用的密钥加密）。

### C++ STL中的迭代器在什么情况下会失效？如何应对失效的情况？
最常见的，在erase(iter)的时候，iter会失效。但是好在这种情况下，erase函数会返回一个新的迭代器。  

## 设计模式
### iOS中有哪些设计模式？

### iOS移动APP架构

## 网络专场
### 网络优化方案都有哪些？

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

网络劫持一般有两种情况，一种是DNS劫持，另一种是HTTP劫持。

从表现上区分这两种劫持非常简单。

如果是DNS劫持，你输入的网址是google.com，然后出来的页面是百度。

如果是HTTP劫持，你打开了google.com，可是右下角弹出了百度推广的不孕不育广告。

URL域名解析成ip地址的过程被称作 DNS 解析。在这个过程中，由于 DNS 请求报文是明文状态，可能会在请求过程中被监测，然后攻击者伪装DNS服务器向主机发送带有假ip地址的响应报文，从而使得主机访问到假的服务器。这个就是DNS劫持的根本原理。

而另一种就是HTTP劫持。在运营商的路由器节点上，设置协议检测，一旦发现是HTTP请求，而且是html类型请求，则拦截处理。后续做法往往分为2种，1种是类似DNS劫持返回302让用户浏览器跳转到另外的地址，还有1种是在服务器返回的 HTML 数据中插入 js 或 dom 节点，从而使网页中出现自己的广告等等垃圾信息。

一般来说，针对各种网络劫持，大部分工作都是由前端来完成，针对这一方面的研究，也大多都是前端开发方向。但是其实客户端也可以通过一些方法来防劫持。

* DNS劫持   
  一般情况下，考虑DNS劫持大多发生在使用webview的时候。相较于使用网页，正常的网络请求，即便被劫持了无非是返回错误的数据，或者干脆404。   
  可以基于NSURLProtocol实现LocalDNS防劫持方案。 简单来说，在网页发起请求的时候获取请求域名，然后在本地进行解析得到ip，返回一个直接访问网页ip地址的请求。[DNS防劫持](http://sindrilin.com/apm/2017/03/31/DNS劫持/)这篇文章里有示例代码。     

### 什么是长连接？有没有优化方案？
TCP连接在长时间没有数据传输的时候，会断开连接。为了实现长连接，就要定期的发送心跳数据。所谓的长连接并没有什么高深的地方，就是想办法让一个TCP连接长时间的保持。  

心跳包，通常是客户端每隔一小段时间向服务器发送的一个数据包，通知服务器自己仍然在线，并传输一些可能有必要的数据。因按照一定的时间间隔发送，类似于心跳，所以叫做心跳包。事实上为了保持长连接，至于包的内容，是没有特别规定的，不过一般都是很小的包，或者只是包含包头的一个空包。  

在TCP协议的机制里面，本身是存在有心跳包机制的，也就是TCP协议中的SO_KEEPALIVE，系统默认是设置2小时的心跳频率。要用setsockopt将SOL_SOCKET.SO_KEEPALIVE设置为1才是打开，并且可以设置三个参数tcp_keepalive_time/tcp_keepalive_probes/tcp_keepalive_intvl，分别表示连接闲置多久开始发keepalive的ACK包、发几个ACK包不回复才当对方死了、两个ACK包之间间隔多长。  

至于优化方案，这个通过Google搜索不到。

### 什么是TCP通讯过程中出现的粘包现象？如何封包和拆包？
首先明确2点：  
1. 这里的包是指应用层面的消息包   
2. 使用UDP协议进行通信时不会有此问题，因为存在保护消息边界，就是指传输协议把数据当作一条独立的消息在网上传输，接收端只能接收独立的消息。  

TCP协议发送的数据，是流式的，没有保护消息边界。所谓的粘包现象，是指消息包在客户端粘在一起了，需要拆包。  
粘包产生的原因：  
1. 发送方引起。这个是由TCP协议本身造成的，TCP为提高传输效率，发送方往往要收集到足够多的数据后才发送一包数据。若连续几次发送的数据都很少，通常TCP会根据优化算法把这些数据合成一包后一次发送出去，这样接收方就收到了粘包数据。  
2. 接收方引起的粘包是由于接收方用户进程不及时接收数据，从而导致粘包现象。

其实这个问题也没有啥含量，封包的时候肯定得有定长的头部字段，里面可以读出来消息体的长度。而在客户端，读socket的时候可以指定读取多少字节。那么客户端可以一直重复一个循环：读头部，解析出消息体的长度，然后再读相应长度的字节。  

### IM


## 算法相关
### 反转二叉树，非递归
已经实现了，按层次遍历二叉树，对于每个结点，交换其左右子树。

### M个红球和N个黑球排序，有多少种排法？

### 给定一棵二叉树和两个节点，求这两个节点的最近的公共父节点。

## 针对项目的问题  
### 礼物相关
*  普通礼物是采用的什么方案？如果是序列帧，那么帧率是多少？每张图片多大？ 
   采用的是序列帧的方案，FPS设定为8，图片大小是700x450。
*  一个礼物压缩包是多大？  
   这个根据礼物的动画帧数的多少而不同。
*  如果收到礼物时，资源还没有下载完毕，你们项目中是如何处理的？  
   在这种情况下，会去下载这个礼物的资源，优先级设为高。等下载完成后再播放。

*  你知道哪些动图格式？  
   Gif: 致命的缺点是，它通常只支持256色索引颜色，这导致它只能通过抖动，差值等方式模拟较多丰富的颜色；它的alpha通常只有1bit，这意味着一个像素只能是完全透明或者完全不透明。  
   BPG: 一个移动端比较冷门的格式，压缩比很高，但是效率是硬伤。在所有的格式比较中，解码速度基本只有别的格式的1/2到1/10，基本不能用到礼物这种需要及时显示的需求上。  
   APNG: PNG的位图动画扩展，但未获得PNG组织官方认可。虽然有三方库可以解决播放的问题，但是它的压缩效率不高，文件体积较大。  
   WebP: 这个可以用在直播的礼物方案中。    
   可以查看文章[视频直播之webp礼物解决方案](https://www.jianshu.com/p/8dc745523e03)，了解更多。  

### 项目中在网络通信环节用了哪些安全技术？  
https  + AES加密。  
虽然SSL通信被认为是相当安全了，但是中间人攻击(man-in-the-middle attack)还是会带来威胁。为了万无一失，就需要用到SSL Pinning技术。使用这个技术，可以保证一个APP是在和期望的服务器在进行通信。在实现的时候，要求将服务器的SSL证书，打包进APP的bundle中。  

下面给出使用原生的NSURLSession进行网络通信时的方案。著名的网络库AFNetworking也支持SSL Pinning，使用起来更是简单。了解详情可以看下[这篇文章](https://infinum.co/the-capsized-eight/how-to-make-your-ios-apps-more-secure-with-ssl-pinning)  
##### NSURLSession  

```
-(void)URLSession:(NSURLSession *)session didReceiveChallenge:(NSURLAuthenticationChallenge *)challenge completionHandler:(void (^)(NSURLSessionAuthChallengeDisposition, NSURLCredential * _Nullable))completionHandler {

    // Get remote certificate
    SecTrustRef serverTrust = challenge.protectionSpace.serverTrust;
    SecCertificateRef certificate = SecTrustGetCertificateAtIndex(serverTrust, 0);

    // Set SSL policies for domain name check
    NSMutableArray *policies = [NSMutableArray array];
    [policies addObject:(__bridge_transfer id)SecPolicyCreateSSL(true, (__bridge CFStringRef)challenge.protectionSpace.host)];
    SecTrustSetPolicies(serverTrust, (__bridge CFArrayRef)policies);

    // Evaluate server certificate
    SecTrustResultType result;
    SecTrustEvaluate(serverTrust, &result);
    BOOL certificateIsValid = (result == kSecTrustResultUnspecified || result == kSecTrustResultProceed);

    // Get local and remote cert data
    NSData *remoteCertificateData = CFBridgingRelease(SecCertificateCopyData(certificate));
    NSString *pathToCert = [[NSBundle mainBundle]pathForResource:@"github.com" ofType:@"cer"];
    NSData *localCertificate = [NSData dataWithContentsOfFile:pathToCert];

    // The pinnning check
    if ([remoteCertificateData isEqualToData:localCertificate] && certificateIsValid) {
        NSURLCredential *credential = [NSURLCredential credentialForTrust:serverTrust];
        completionHandler(NSURLSessionAuthChallengeUseCredential, credential);
    } else {
        completionHandler(NSURLSessionAuthChallengeCancelAuthenticationChallenge, NULL);
    }
}
```

## 直播技术
### 直播涉及的步骤
1. 采集视频和音频数据  
   得到CMSampleBufferRef类型的数据,代表视频的CMSampleBufferRef中保存的数据是yuv420格式的视频帧，代表音频的CMSampleBufferRef中保存的数据是PCM格式的音频帧。    
2. 前处理  
   美颜算法，视频的模糊效果，水印等都是在这个环节做。一般现在都使用GPUImage。  
3. 编码  
   对视频进行硬编码得到h264数据，对音频进行硬编码得到AAC数据。重难点在于要在分辨率，帧率，码率等参数设计上找到最佳平衡点。iOS8之后可以使用VideoToolbox.frarmework进行硬编码。  
4. 对编码后的音视频数据进行FLV封包  
5. 建立RTMP连接并上推到服务器  
6. 服务端处理：需要在服务器做一些流处理工作，让推送上来的流适配各个平台多种不同的协议，例如RTMP, HLS。  
7. 拉流（看播）端解码和渲染，也就是音视频的播放。解码也必须要硬解码，这个很早就支持。这块儿的难点在于音画同步。    

### 直播协议
RTMP：  
Real Time Messaging Protocol是Macromedia开发的一套视频直播协议，现在属于Adobe公司。实时性好，一般使用这种协议来上传视频流，也就是视频流推送到服务器。延迟在1-3秒。  
相对其它协议而言，RTMP协议初次建立连接的时候握手过程过于复杂，视不同的网络状况会给首开带来100ms以上的延迟。      

HTTP-FLV:  
即使用HTTP协议流式的传输媒体内容。相对于RTMP，HTTP更简单和广为人知，而且不担心被Adobe的专利绑架。内容延迟同样可以做到1~3秒，打开速度更快，因为HTTP本身没有复杂的状态交互。所以从延迟角度来看，HTTP-FLV要优于RTMP。

HLS:  
即Http Live Streaming, 是由苹果提出的，基于HTTP的流媒体传输协议。它有一个非常大的优点：HTML5可以直接打开播放。这就意味着可以把一个直播链接通过微信等转发分享，不需要安装任何独立的APP，有浏览器即可，所以流行度很高。  

其实是一个“文本协议”，并不是一个“流媒体协议”。直播客户端获取到的，并不是一个完整的数据流。HLS协议在服务器端将直播数据流存储为连续的，很短时长的媒体文件（MPEG-TS格式），而客户端则不断的下载并播放这些小文件，因为服务器端总是会将最新的直播数据生成新的小文件，这样客户端只要不停的按顺序播放从服务器获取到的文件，就实现了直播。  

由此可见，基本上可以认为，HLS是以点播的技术方式来实现直播。由于数据通过HTTP协议传输，所以不用考虑防火墙或者代理的问题，而且分段文件的时长很短，客户端可以很快的选择和切换码率，以适应不同带宽条件下的播放。不过HLS的这种技术特点决定了延迟一般总是会高于普通的流媒体直播协议。

每一个.m3u8 文件，分别对应若干个 ts 文件，这些 ts 文件才是真正存放视频的数据，m3u8 文件只是存放了一些 ts 文件的配置信息和相关路径，当视频播放时，.m3u8 是动态改变的，video 标签会解析这个文件，并找到对应的 ts 文件来播放，所以一般为了加快速度，.m3u8 放在 Web 服务器上，ts 文件放在 CDN 上。

支持的视频流编码为H.264，音频流编码为AAC。

RTP：  
Real-time Transport Protocol，用于Internet上针对多媒体数据流的一种传输层协议。 和前面的协议的一个重要的区别就是默认是使用UDP协议来传输数据。  

### 如何观看直播的视频
观看直播视频有以下方式：HLS, RTMP, HTTP-FLV，RTP。  
好看的指标参数：  
码率：每秒传送的比特数，单位为bps。码率越高，视频越清晰。  
帧率：即fps，每秒钟多少帧。  
分辨率：影响图像大小，与图像大小成正比。

### 用到的格式
YUV是一种图片储存格式，跟RGB格式类似。YUV中，y表示亮度，单独只有Y就可以形成一张图片，只不过这张图片是灰色的。u和v表示色差(u和v也被称为Cb-蓝色差，Cr-红色差)。  

一张yuv格式的图像，占用字节数为 width * height * 3 / 2，而一张RGB格式的图像，占用字节数为width * height * 3，相比较RGB，YUV只是一半的占用字节数。  

PCM格式，就是录制声音时，保存的最原始的声音数据格式。  

h264，一种压缩格式，将视频帧分为关键帧和非关键帧。关键帧的数据是完整的。包含了所有的颜色数据。这样的视频帧称为I帧。

非关键帧数据不完整，但是它能够根据前面或者后面的帧数据，甚至自己的部分帧数据，将自身数据补充完整。这种视频帧被称为 B/P 帧。

总体来说，h264跟yuv相比，最大的不同就是它是压缩的（通常此过程称为编码，不只是简单的压缩）

aac: 性质同h264一样，是PCM的压缩格式。  

flv: 是音频和视频的封包格式，RTMP传输的时候要求是flv格式。  

### 视频编码压缩   
为了便于视频内容的存储和传输，通常需要减少视频内容的体积，也就是需要将原始的内容元素(图像和音频)经过压缩，压缩算法也简称编码格式。例如视频里边的原始图像数据会采用 H.264 编码格式进行压缩，音频采样数据会采用 AAC 编码格式进行压缩。

视频内容经过编码压缩后，确实有利于存储和传输; 不过当要观看播放时，相应地也需要解码过程。因此编码和解码之间，显然需要约定一种编码器和解码器都可以理解的约定。就视频图像编码和解码而言，这种约定很简单：
编码器将多张图像进行编码后生产成一段一段的 GOP ( Group of Pictures ) ， 解码器在播放时则是读取一段一段的GOP 进行解码后读取画面再渲染显示。   
GOP ( Group of Pictures ) 是一组连续的画面，由一张 I 帧和数张 B / P 帧组成，是视频图像编码器和解码器存取的基本单位，它的排列顺序将会一直重复到影像结束。

### sps&pps, AudioSpecificConfig
sps&pps是h264中的概念，它包含了一些编码信息，如profile，图像尺寸等信息。硬编码以后，sps&pps数据能够通过关键帧获取。    
AudioSpecificConfig是aac中的概念，它包含了音频信息，如采样率，声道数等信息。  
RTMP连接成功后，一定要先发送sps&pps，AudioSpecificConfig这两个数据对应的tag，否则视频是播放不出来的。  

### H264中的时间戳
H264里有两种时间戳：DTS(Decoding Time Stamp)和PTS（Presentation Time Stamp)。其实际意义是，PTS是真正录制和播放的时间戳，而DTS是解码的时间戳。  

对于普通的无B帧视频，PTS/DTS应该是相等的，因为解码的时候不需要先解出后面的帧。  

对于有B帧的视频，I帧的PTS依然等于DTS，P帧的PTS > DTS，B帧的PTS < DTS。  

也可以换个方式来理解：  
若视频没有B帧，则I和P都是解码后即刻显示。  
若视频含有B帧，则I是解码后即刻显示，P是先解码后显示，B是后解码先显示。  

### 怎么优化打开速度，达到传说中的“秒开”？
1. 改写播放器逻辑让播放器拿到第一个关键帧后就给予显示。  
2. 在APP业务逻辑层面方面优化，比如提前做好DNS解析（省几十毫秒），和提前做好测速选线（择取最优线路）
3. 除了移动端可以做体验优化之外，直播流媒体服务端架构也可以降低延迟。例如收流服务器主动推送GOP至边缘节点，边缘节点缓存GOP，播放端则可以快速加载，减少回源延迟。  
4. 在切换直播间的场景下，可以做预加载操作。  

### IJKPlayer在播放RTMP视频流时，出现视频音频不同步现象，解决方案是？
这个现象的原因是CPU在处理视频帧的时候处理得太慢，默认的音视频同步方案是视频同步到音频，导致了音频播放过快，视频跟不上。  
可以通过修改framedrop的数值来解决不同步的问题，framedrop是在视频帧处理不过来的时候丢弃一些帧达到同步的效果。  
代码如下：  

```
IJKFFOptions *options = [IJKFFOptions optionsByDefault];       
[options setPlayerOptionIntValue: 5 forKey:@"framedrop"];
```




