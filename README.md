# 大标题
## 问答题
### 1.说说ARC和MRC的区别
ARC: 自动引用计数， MRC：手动引用计数


### 2.线程同步工具都有哪些？Lock与Condition的区别是什么？
主要有：Atomic Operations, Lock和Condition。
* Atomic Operations
系统提供了一些原子性的数学运算和逻辑运算函数，声明在/usr/include/libkern/OSAtomic.h中。这些函数分为以下类别：
Add: Adds two integer values together and stores the result in one of the specified variables.
Increment: Increments the specified integer value by 1.
Decrement: Decrements the specified integer value by 1.
