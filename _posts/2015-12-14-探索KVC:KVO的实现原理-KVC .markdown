---
title: "探索KVC/KVO的实现原理-KVC"
layout: post
date: 2015-12-14 15:59
tags: [iOS,Objective-C,KVC]
#star: true
---

##KVC/KVO
KVC(Key-value coding)是Object-C提供的一种利用字符串来间接访问对象属性的一种机制，它是通过访问器去访问对象属性的另一个可选方案，KVO(Key-value observing)是观察者模式在Object-C中的应用，提供对对象属性变化的监听机制，非正式协议NSKeyValueCoding/NSKeyValueObserving对其接口进行了定义，NSObject中提供了接口的默认实现。本文主要介绍KVC的实现原理，并通过一些示例来验证。


##KVC的使用
查看NSKeyValueCoding.h文件，可以看到KVC的相关API:

* valueForKey: 根据传入字符串名称获取属性值。
* valueForKeyPath:根据传入字符串路径获取属性值。
* valueForUndefinedKey: 当valueForKey无法找到属性值调用该方法，默认实现是抛出异常，可重写这个函数做错误处理。

* setValue:forKey:根据传入字符串名称设置属性。
* setValue:forKeyPath:根据传入字符串路径设置属性值。
* setValue:forUnderfinedKey:当setValue:forKey无法找到属性值调用该方法，默认实现是抛出异常，可重写这个函数做错误处理。
* setNilValueForKey: 对非类对象属性设置nil时调用，默认抛出异常。

另外，头文件中还有一些获取可变集合的API接口，顾名思义，是为了获得集合类型属性的可变集合。

使用示例：

<!--![](./img5.png)-->
{% include image.html
            img="assets/images/20151214/img5.png"
            title="title for image"
            caption="caption for image" %}

ClassA和ClassB的定义如上，那我们可以通过以下的方法来实现对实例对象integerVar属性的存取

<!--![](./img6.png)-->

{% include image.html
            img="assets/images/20151214/img6.png"
            title="title for image"
            caption="caption for image" %}

##KVC查找规则
查阅苹果源代码，可以在API注释中看到KVC键值查找的方式，如下

1、valueForKey:的查找规则（valueForKeyPath:类似）

* 首先，在对象的类中查找访问器，按-get< Key >, -< key >, or -is< Key >的顺序查找getter方法，如果找到则直接调用。如果返回值是对象指针类型，则直接返回，如果是BOOL、Integer等基本数据类型，则转换成NSNumber返回，否则转换成NSValue返回。
* 否则，查找-countOf< Key > ， -indexIn< Key >OfObject: ， -objectIn< Key >AtIndex:,或者-< key >AtIndexes:格式的方法。如果countOfKey,indexInKeyOfObject:和另外两个方法中的一个找到，那么就会返回一个可以响应NSOrderedSet所有方法的代理集合,任何发送给集合代理的方法都会被转换成以上几个方法的组合。如果类还实现了-get< Key >:range:方法，也将被调用来进行性能优化。
* 否则，查找-countOf< Key > ， -objectIn< Key >AtIndex:,或者-< key >AtIndexes:格式的方法。如果countOfKey和另外两个方法中的一个找到，那么就会返回一个可以响应NSArray所有方法的代理集合,任何发送给集合代理的方法都会被转换成以上几个方法的组合。如果类还实现了-get< Key >:range:方法，也将被调用来进行性能优化。
* 否则，查找-countOf< Key > ,-enumeratorOf< Key >, and -memberOf< Key >:格式的方法。如果这三个方法都找到，那么就会返回一个可以响应NSSet所有方法的代理集合,任何发送给集合代理的方法都会被转换成以上几个方法的组合。
* 否则，如果类方法+accessInstanceVariablesDirectly返回YES,则按照_< key >, _is< Key >, < key >, 或者 is< Key > 的顺序查找对象的成员变量，如果找到就返回该变量，对于非对象指针类型，同样会按照之前那样做NSNumber和NSValue的转换。
* 否则，调用-valueForUndefinedKey:方法，返回其调用结果。该方法的默认实现是抛出NSUndefinedKeyException异常，代码中可以覆写该方法。

2、setValue:forKey:的查找规则（setValue:forKeyPath:类似）

* 首先，在对象的类中查找-set< Key >格式的方法，找到则检查其参数类型，如果参数类型不是对象指针类型，而value是nil，则调用setNilValueForKey:方法，否则就直接调用该方法并传入value作为参数。如果参数类型是其他类型，函数调用之前会进行NSNumber/NSValue到基本类型的转换。
* 否则，如果类方法+accessInstanceVariablesDirectly返回YES,则按序搜索名称为_< key >, _is< Key >, < key >, 或者 is< Key >的实例变量，如果找到改变量，并且其类型是对象指针类型，则首先对原变量对象进行release操作，然后对value retain并进行赋值操作。如果是基本类型，同样需要进行NSNumber/NSValue到基本类型的转换。
* 否则，调用-setValue:forUndefinedKey:方法。




-------------------------

按照上面的规则，我们进行实验。

定义一个类，并实现其属性的访问器。
<!--![](./img1.png)-->
{% include image.html
            img="assets/images/20151214/img1.png"
            title="title for image"
            caption="caption for image" %}

调用如下：
<!--![](./img2.png)-->
{% include image.html
            img="assets/images/20151214/img2.png"
            title="title for image"
            caption="caption for image" %}
输出结果：

<pre>
<code>
2015-12-13 12:24:44.028 KVC-KVOTest[23971:1739788] setValue:forKey:strProperty
2015-12-13 12:24:44.029 KVC-KVOTest[23971:1739788] setter of strProperty called.
2015-12-13 12:24:44.029 KVC-KVOTest[23971:1739788] valueForKey:strProperty
2015-12-13 12:24:44.029 KVC-KVOTest[23971:1739788] getter of strProperty called.
2015-12-13 12:24:44.029 KVC-KVOTest[23971:1739788] setValue:forKey:intProperty
2015-12-13 12:24:44.029 KVC-KVOTest[23971:1739788] setter of intProperty called.
2015-12-13 12:24:44.029 KVC-KVOTest[23971:1739788] valueForKey:intProperty
2015-12-13 12:24:44.029 KVC-KVOTest[23971:1739788] getter of intProperty called.
</code>
</pre>

我们看到，当属性的访问器存在时，KVC会调用他们，实现对属性的存取，并且内部完成了NSNumber和基本类型之间的类型转换。

下面我们把属性的访问器方法全部注释掉，并执行下面的代码。
<!--![](./img3.png)-->
{% include image.html
            img="assets/images/20151214/img3.png"
            title="title for image"
            caption="caption for image" %}

输出结果：

<pre>
<code>
2015-12-13 12:30:18.231 KVC-KVOTest[23988:1742674] setValue:forKey:strProperty
2015-12-13 12:30:18.232 KVC-KVOTest[23988:1742674] valueForKey:strProperty
2015-12-13 12:30:18.233 KVC-KVOTest[23988:1742674] I am a string.
</code>
</pre>

我们看到，即使没有访问器，KVC依然可以正确的对属性进行存取，这就是上面提到的`按照_<key >, _is< Key >, < key >, 或者 is< Key > 的顺序查找对象的成员变量`查找规则，我们尝试把属性名_strProperty改成_strProperty2，重新运行程序，程序输出如下。

<pre>
<code>
2015-12-13 12:34:06.852 KVC-KVOTest[24003:1745390] *** Terminating app due to uncaught exception 'NSUnknownKeyException', reason: '[<KVCTestClass 0x100203cd0 > setValue:forUndefinedKey:]: this class is not key value coding-compliant for the key strProperty.'
</code>
</pre>

如我们所预料，程序在执行`[kvcObj setValue:@"I am a string." forKey:@"strProperty"];`这句代码时抛出了NSUnknownKeyException异常，找不到strProperty属性。我们在KVCTestClass类中添加如下方法，重新运行程序，程序运行正常。
<!--![](./img4.png)-->
{% include image.html
            img="assets/images/20151214/img4.png"
            title="title for image"
            caption="caption for image" %}


##小结
本文主要介绍和验证KVC的查找规则，而这些规则的应用离不开Objective-C强大运行时的支持，正是因为用了运行时的支持，KVC才可以方便的按照上面的去查找方法和属性，这里还不涉及利用运行时对类的方法进行修改，后面还会写一篇关于KVO实现的文章，将看到runtime的强大之处以及苹果工程师的聪明才智。

本文示例代码参见:[这里](https://git.hz.netease.com/hzzhangjw/kvc-kvo-demo)