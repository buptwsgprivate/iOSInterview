### Runtime运行时
程序调用某个方法时, 会向消息接收者发送一条消息, 运行时会根据消息的接收者是否响应该消息而做出不同的反应.
方法调用会被编译器转化成objc_msgSend(receiver, selector, arg1, ...)

### 实例，类，元类的关系图   
![实例，类，元类关系图](https://github.com/buptwsgprivate/iOSInterview/blob/master/Images/instance_class_metaclass.png)  
上图中的实现代表`superclass`指针, 虚线是`isa`指针. 根元类的超类是NSObject, `isa`指针指向了自己, NSObject的超类为nil.  
`cache_t` 

```
struct cache_t {
	struct bucket_t *_buckets;
    mask_t _mask;
    mask_t _occupied;
}
```  

`_buckets`存储`IMP`, `_mask`和`_occupied`对应vtable.  

cache 为方法调用的性能进行优化, 每当实例对象接收到一个消息时, 为了避免效率, 它优先在 cache 中查找, 找不到的情况下在去isa指向的类的方法列表中遍历查找. Runtime 系统会把被调用的方法存到 cache 中(理论上讲一个方法如果被调用，那么它有可能今后还会被调用), 下次查找的时候效率更高.  

bucket_t 中存储了指针与 IMP 的键值对:  

```
struct bucket_t {
private:
    cache_key_t _key;
    IMP _imp;

public:
    inline cache_key_t key() const { return _key; }
    inline IMP imp() const { return (IMP)_imp; }
    inline void setKey(cache_key_t newKey) { _key = newKey; }
    inline void setImp(IMP newImp) { _imp = newImp; }

    void set(cache_key_t newKey, IMP newImp);
};
```
