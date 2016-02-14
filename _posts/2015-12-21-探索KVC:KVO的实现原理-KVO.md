---
title: "探索KVC/KVO的实现原理-KVO"
layout: post
date: 2015-12-21 17:31
tags: [iOS,Objective-C,KVO]
#star: true
---

##KVC/KVO

KVO(Key-value observing)是观察者模式在Objective-C中的应用，提供对对象属性变化的监听机制，非正式协议NSKeyValueObserving对其接口进行了定义，NSObject中提供了接口的默认实现。本文主要介绍KVO的实现原理，并通过一些示例来验证，此外，对KVO性能做了简单测试，还对KVO小小的“吐槽”了一点。

##KVO的使用
查看NSKeyValueObserving.h文件，可以看到KVO的相关API:

* addObserver:forKeyPath:options:context:  添加属性监听
* removeObserver:forKeyPath:context:	删除属性监听（带有传入的上下文）
* removeObserver:forKeyPath: 删除属性监听
* observeValueForKeyPath:ofObject:change:context: 属性变化的事件通知回调

使用示例

类定义：

<!--![](./kvo1.png)
![](./kvo2.png)-->

{% include image.html
            img="assets/images/20151221/kvo1.png"
            title="title for image"
            caption="caption for image" %}
        
{% include image.html
            img="assets/images/20151221/kvo2.png"
            title="title for image"
            caption="caption for image" %}

代码很简单，我们在KVOTestClass中对成员变量的intVar和arrayVar两个属性的变化进行监听，包括变化前和变化后的值，并打印回调过来的数据。

使用如下：

<!--![](./kvo3.png)-->
{% include image.html
            img="assets/images/20151221/kvo3.png"
            title="title for image"
            caption="caption for image" %}

在KVOTest函数中，我们修改了intVar和arrayVar的值，还对arrayVar的数组添加元素和替换元素，程序输出结果如下：

<pre>
<code>
2015-12-20 12:34:39.716 KVC-KVOTest[58295:4236087] intVar is set to ===>200
2015-12-20 12:34:39.717 KVC-KVOTest[58295:4236087] {
    kind = 1;
    new = 200;
    old = 100;
}
2015-12-20 12:34:39.717 KVC-KVOTest[58295:4236087] {
    kind = 1;
    new =     (
        4,
        5,
        6
    );
    old =     (
        1,
        2,
        3
    );
}
2015-12-20 12:34:39.718 KVC-KVOTest[58295:4236087] {
    indexes = "<_NSCachedIndexSet: 0x100204170>[number of indexes: 1 (in 1 ranges), indexes: (3)]";
    kind = 2;
    new =     (
        4
    );
}
2015-12-20 12:34:39.718 KVC-KVOTest[58295:4236087] {
    indexes = "<_NSCachedIndexSet: 0x100204130>[number of indexes: 1 (in 1 ranges), indexes: (1)]";
    kind = 4;
    new =     (
        20
    );
    old =     (
        5
    );
}
</code>
</pre>

从上面的例子，我们看到，kvo可以做到对普通属性和集合属性变化的感知，并通知给监听者，我们可以获取到变化的前后值，还可以根据kind来区分是何种操作。这里稍微有点不同的是，想要能够感知到集合内部内容的变化，集合操作的方式稍有不同，例如我们这里是使用`mutableArrayValueForKey:@"arrayVar"]`而不是采用`.arrayVar`的方式。此外，KVO还可以表达属性变化的依赖关系，具体使用可以参考相关文档，在此不作介绍。


##KVO的实现

回头再看上面的例子，我们可以先YY一下KVO的实现，我们倒着来推理一下。如果observeValueForKeyPath:ofObject:change:context:能被回调给我们，那系统必定对ClassC做了什么事情，这件事情又一定是和属性的setter函数有关才行，比如上面我们实现的setIntVar方法，而我们的实现中并没有包含任何通知观察者的操作，可以猜测，系统一定做了这件事情。Object-C语言底层是C语言实现，在我们的setter函数（一个“C函数”）调用前后，系统一定做了插入了一些函数的工作，并且这个工作发生在运行期，这就需要Objective-C的runtime的支持，包括动态创建类，动态查找和替换类方法。其实想到这一步大概也知道实现流程是什么样子了，下面看一下KVO实现的庐山真面目：

看下苹果文档：

<pre>
<code>
Key-Value Observing Implementation Details
Automatic key-value observing is implemented using a technique called isa-swizzling.
The isa pointer, as the name suggests, points to the object's class which maintains a dispatch table. This dispatch table essentially contains pointers to the methods the class implements, among other data.
When an observer is registered for an attribute of an object the isa pointer of the observed object is modified, pointing to an intermediate class rather than at the true class. As a result the value of the isa pointer does not necessarily reflect the actual class of the instance.
You should never rely on the isa pointer to determine class membership. Instead, you should use the class method to determine the class of an object instance.
</code>
</pre>


意思是，当某个对象第一次被观察时，系统就会在运行期动态地创建该对象的类的一个派生类，这个对象的isa指针将被指向该派生类。这个派生类中会重写被观察属性的setter方法，也就是会加入一些用于通知观察者的代码，另外，这个派生类还会重写class方法，返回原来的类给调用者，这样做到了对系统的‘欺骗’。

那么通知观察者的接口长什么样子呢，在NSKeyValueObserving.h中我们找到两个方法：

<pre>
<code>
- (void)willChangeValueForKey:(NSString *)key;
- (void)didChangeValueForKey:(NSString *)key;
</code>
</pre>
这两个方法的作用是根据key去回调到-observeValueForKeyPath:ofObject:change:context: 来通知观察者属性的变化，而属性变化前后的值是通过KVC的-valueForKey:接口获取的，该接口的查找顺序可以参考[这里](http://ks.netease.com/blog?id=3270)，所以，KVO的实现实际对KVC的实现有依赖。

##我们来验证一下

我们在KVOTestClass中加入下面一个方法，其中PrintDescription用来打印相关信息：

<!--![](./kvo4.png)-->
{% include image.html
            img="assets/images/20151221/kvo4.png"
            title="title for image"
            caption="caption for image" %}

打印信息如下：

<pre>
<code>
2015-12-20 14:13:05.320 KVC-KVOTest[58477:4289189] 
	cObj: <ClassC: 0x100100650>
	[NSObject class]: ClassC
	object_getClass: NSKVONotifying_ClassC
	object_getClass implements methods : <setArrayVar:, setIntVar:, class, dealloc, _isKVOA>
2015-12-20 14:13:05.321 KVC-KVOTest[58477:4289189] 
	cObj2: <ClassC: 0x1001037e0>
	[NSObject class]: ClassC
	object_getClass: ClassC
	object_getClass implements methods : <setIntVar:, intVar, arrayVar, setArrayVar:, .cxx_destruct, init>

</code>
</pre>

我们看到cObj2是没有添加KVO的对象，通过class方法获取的类名和真正的类名都是ClassC，其方法列表中包括构造函数，析构函数，还有属性的getter和setter。而cObj由于添加了KVO,其通过class方法获取的类名仍然是ClassC，而真正的类名是NSKVONotifying_ClassC这个类中重写了被监听的属性的setter方法（没有被监听的属性，不会重写setter方法），class方法，dealloc方法（资源释放），还有私有方法_isKVOA（用于是否是KVO机制判断）。




##性能
我们来简单测试一下KVO的性能，我们的测试用例包括：

设需要进行监控测试的对象属性是NSInteger类型的intVar，kTestTimes是测试次数，另外，我们的observeValueForKeyPath:ofObject:change:context:的实现代码是空实现，

* (A)_normalObjs中有kTestTimes个对象，分别对他们的intVar属性进行赋值操作
* (B)_observedOnceObj是有一个监听者，对其intVar属性进行赋值操作kTestTimes次
* (C)_observedObjs中有kTestTimes个对象，对他们的intVar属性进行监听， 分别对_observedObjs中的对象赋值依次
* (D)_observers中有kTestTimes个监听者，他们对_observedObj的intVar属性进行监听，对_observedObj的intVar属性进行一次赋值操作

测试核心代码如下:
<!--![](./kvo6.png)
![](./kvo5.png)-->
{% include image.html
            img="assets/images/20151221/kvo6.png"
            title="title for image"
            caption="caption for image" %}
{% include image.html
            img="assets/images/20151221/kvo5.png"
            title="title for image"
            caption="caption for image" %}          

测试数据如下：


手机：iPhone5S

系统：iOS9.1

单位：毫秒

| kTestTimes |A|B|C|D|
|----|----|----|----|----|
|5,000|4|20|21|10|
|10,000|4|44|44|20|
|20,000|6|81|84|40|
|50,000|13|197|213|99|
|100,000|22|422|447|-[内存不足了]|


可以看到，KVO带来了一定的性能损耗，且随着KVO使用数量的增多，性能影响也越大，另外，多个对象被同一个对象监听比一个对象有多个监听者带来的性能开销要大。

##KVO的存在的问题

  
  KVO的接口设计不是很方便，对一个属性的监听，监听者需要实现observeValueForKeyPath:ofObject:change:context:方法，使得代码相分离，另外，添加监听时是通过传入context作为用户参数，这并不能想block那样方便的访问上下文信息。

另外，苹果的文档里我们还看到这样一段话：

<pre>
<code>
When the same observer is registered for the same key path multiple times, but with different context pointers each time, -removeObserver:forKeyPath: has to guess at the context pointer when deciding what exactly to remove, and it can guess wrong
</code>
</pre>

意思就是当我们对同一属性多次添加同一个监听者，而传入的context不同，在移除的时候，系统需要去猜测相应的context。

KVO的一些缺点可以参考[这里](https://mikeash.com/pyblog/key-value-observing-done-right.html)。

本文示例代码查看[这里](https://git.hz.netease.com/hzzhangjw/kvc-kvo-demo)。

