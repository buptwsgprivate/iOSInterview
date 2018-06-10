## Swift加强

### 在Swift中如何使用runtime
对于基类是NSObject的类，可以直接使用。  
对于Swift中的类，在属性和方法之前加上@objc关键字, 则一般情况下可以在runtime中使用了. 但有一些情况下, Swift会做静态优化而无法使用runtime.  
要想完全使得属性和方法被动态调用, 必须使用dynamic关键字. 而dynamic关键字会隐式地加上@objc来修饰.

### 什么是required initializer

### Any, AnyClass, AnyObject

### 如何获取一个对象的类型信息

### Swift与OC的混合编程

