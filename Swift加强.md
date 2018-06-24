## Swift加强

### Swift相比OC的优点
* 不需要写头文件
* 任意位置的缺省参数
* 泛型编程
* 扩展，不但可以对类类型进行扩展，还可以对值类型进行扩展，协议也可以被扩展以提供缺省行为
* 强大的枚举功能
* 集合类型提供对函数式编程的支持
* Optional类型

### 在Swift中如何使用runtime
对于基类是NSObject的类，可以直接使用。  
对于Swift中的类，在属性和方法之前加上@objc关键字, 则一般情况下可以在runtime中使用了. 但有一些情况下, Swift会做静态优化而无法使用runtime.  
要想完全使得属性和方法被动态调用, 必须使用dynamic关键字. 而dynamic关键字会隐式地加上@objc来修饰.

### 什么是required initializer

### Any, AnyClass, AnyObject

### 如何获取一个对象的类型信息

### Swift与OC的混合编程

