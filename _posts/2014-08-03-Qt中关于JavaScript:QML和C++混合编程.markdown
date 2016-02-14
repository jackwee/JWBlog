---
title: "Qt中关于JavaScript/QML和C++混合编程"
layout: post
date: 2014-08-03 14:26
tag:[qt JavaScript C++ 混合编程]
#star: true
---
 `版权声明：本文为博主原创文章，未经博主允许不得转载。`

Qt中关于JavaScript/QML和C++混合编程


声明：本文很多内容都是个人学习和尝试而得出的结果，难免会有疏漏和粗枝大叶，如有不正确或不严谨之处，还请指正。



感谢soloist的悉心教导和耐心讲解，以及在工作和学习中对我鼓励与帮助。



我使用的环境是： Qt 5.2.1 (Clang 5.0 (Apple), 64 bit)，Qt Creator 3.1.1 (opensource)。



[QML(Qt Markup Language)](http://qt-project.org/doc/qt-5/qmlapplications.html)

[The Meta-Object System](http://qt-project.org/doc/qt-5/metaobjects.html)


[ABI(Application binary interface)](http://en.wikipedia.org/wiki/Application_binary_interface)

[FFI(Foreign function interface)](http://en.wikipedia.org/wiki/Foreign_function_call)

[vc如何返回函数结果及压栈参数](http://blog.csdn.net/soloist/article/details/1267147)


1、前传

```
 “不同语言之间进行混合编程是计算机领域一个火热的话题，一些软件技术和工具，如SWIG (Simplified Wrapper and Interface Generator)、COM(Component Object Model) 的目的就是为了解决不同语言之间的互通性。”


  “对于不同语言之间的互操作，主要解决三大问题：

（1）数据类型的识别（包括数据的存储格式，访问方式）。

（2）数据类型的转换（类型系统如何互通）。

（3）对象生命周期管理。”


“说到不同编程语言的混合调用，就涉及到两种语言之间数据类型的互通，方法的调用，本质上就是要求两种语言能够互相通信，打个比方，这就好比两个国家之间的语言进行沟通，例如阳阳会讲日语，龙龙会讲法语，那么他们怎么才能进行沟通呢？一种方式是他们采用一种双方都能听懂的英语来沟通，另一种方式就是阳阳学会了说法语，龙龙学会了说日语。”


“回到混合编程的话题，与此类似，一种方式就是两种语言采用一种都支持的中间语言（如xml）进行通信，这种方式的优点是语言间的耦合性较低，不需要对另一门语言的实现细节有了解，缺点是通过中间语言进行通信所付出的代价；另一种方式就是充分了解另一门语言，包括数据类型，数据结构的组织，方法的调用形式等，这种方式的优点是互操作的性能高，缺点是需要底层实现繁琐。”


“说到为什么要进行js与c++的混合开发，原因是他们各有自己的优缺点。js的对象模型比较灵活，支持对象属性的动态创建和删除，支持闭包，但是性能不如c++高，而且局限于浏览器，否则需要有js引擎的支持。c++的优点就是性能比较高，但是是一门静态类型语言，灵活性不足。结合二者的优点，我们可以将逻辑代码部分采用js来实现，大量计算部分采用c++来实现。”

```




2、Qt的对象模型的介绍

 C++的静态特性不够灵活，为了满足GUI编程对运行时高效性和灵活性的要求，Qt对标准C++对象模型进行了扩展（http://qt-project.org/doc/qt-5/object.html），这些扩展都基于了Qt中两个十分重要的两个概念：

（1）QObject

（2）Meta-Object

QObject是所有Qt对象的基类，也是Qt对象模型的核心，扩展的大部分能力都是基于QObject实现。Qt 元对象系统(Meta-Object System)负责Qt中基于signals和slots的对象通信机制，运行期类型信息以及Qt的属性(Q_PROPERTY)系统，对于QObject以及它的子类所创建的对象，都会为其创建一个QMetaObject实例，该实例中存储了所有的关于该对象的元信息(属性的类型，方法的签名信息等)，这些都由Qt所实现的Meta-Object Compiler(moc)来提供。


moc会为每一个包含Q_OBJECT宏的类创建一个新的C++源文件，以“moc_”打头，该文件中包含类的元信息，该文件一般会被单独编译并与原类文件链接使用，也可以被#include到原类文件中使用。





关于moc，Qt官方文档中的介绍如下：

The meta-object system is based on three things:

(1)The QObject class provides a base class for objects that can take advantage of the meta-object system.

(2)The Q_OBJECT macro inside the private section of the class declaration is used to enable meta-object features, such as dynamic properties, signals, and slots.

(3)The Meta-Object Compiler (moc) supplies each QObject subclass with the necessary code to implement meta-object features.



了解了这些，我们也就知道了Qt中Javascript和C++的混合调用采用的是前面所提到的第二种方式进行通信，而不是通过中间语言。



3、JS调用C++

我想这就是FFI中所说的，“In most cases, a FFI is defined by a "higher-level" language, so that it may employ services defined and implemented in a lower level language, typically a systems language like C or C++. ”



Qt中，JS访问C++有两种方式：

(1)一种是将自定义的C++类型注册到QML的类型系统中。


```
class MyClass : public QObject//注意：必须要求是QObject派生的类  
{  
    Q_OBJECT //为了使用Meta-Object System，这个宏是必须的  
    Q_PROPERTY(int name READ name WRITE setName)//定义可以在js中被查询和修改的属性  
public:  
    Q_INVOKABLE void foo();//定义可以在js中访问的方法  
    int name();  
    void setName(int n);  
private:  
    int m_name;  
};  


```

```
qmlRegisterType<MyClass>(“mytype”, 1, 0, “MyClass”);//将MyClass注册到QML的类型系统中，以便在qml中使用该类型。  
```


```
//a.直接在qml文件中创建元素  
import QtQuick 2.2  
import mytype 1.0  
Window{  
    id:mywindow  
    MyClass{  
            id:myObj  
            name:“longlong”  
            Component.onComplete:{  
        console.log(myObj.name);  
            }  
    }；  
  
}  
  
  
//b.JS中动态创建对象  
var myObj = Qt.createQmlObject('import QtQuick 2.2;  import my 1.0;MyClass{name:“yangyang”}’,mywindow, “dynamicMyClassCreate");  
  
//c.将MyClass写入一个单独的qml文件，动态创建对象  
  
//myclass.qml  
import QtQuick 2.2  
import mytype 1.0  
  
MyClass{  
    name:”longlong-yangyang”  
}  
var myObj = Qt.createComponent("myclass.qml");  
```




(2)另一种是将c++中定义的对象放到js的全局变量中。


```
QQmlApplicationEngine engine;  
MyClass object;//在C++代码中定义对象object  
engine.rootContext()->setContextProperty("myObj", &object);//将object挂到QML engine的上下文中，这样在任何QML文件中，都可以通过myObj.property和myObj.method()的方式访问myObj了。不过这种方式的性能是比较低。  
```

“Note: Since all expressions evaluated in QML are evaluated in a particular context, if the context is modified, all bindings in that context will be re-evaluated. Thus, context properties should be used with care outside of application initialization, as this may lead to decreased application performance.”——http://qt-project.org/doc/qt-5/qtqml-cppintegration-contextproperties.html




Javascript访问c++属于脚本语言对静态语言的调用范畴，那么涉及到属性的访问，方法的调用，要想做到这些事情js需要考虑哪些问题呢？


首先，js要想要访问属性，需要了解c++的类型，数据的存储结构，已经如何得到数据并转换数据类型；另外，js想要调用c++中的方法，需要考虑如何找到方法的实现，如何像方法中传递参数，如何跳转到方法中去执行c++的方法，如何从c++方法中返回到js中继续执行。这里我们来了解一下lua中调用c过程的一个例子：

a.c文件中定义了add方法，如下:

```
//a.c  
int add(int a, int b){  
    return a+b;  
}  
```

为了能让lua中调用到c函数，lua要求提供一个wrap文件，对add函数进行包装，这里定义到wrap_a.c文件中，如下：

```
//wrap_a.c，这里还是c文件  
    int wrap_add(LuaState* L){  
        int tmp1 = Lua_checkValueInt(L,1);  
        int tmp2 = Lua_checkValueInt(L,2);  
        int result = add(tmp1, tmp2);  
        Lua_pushValue(L,result);  
        return 1;  
    }  
```

Lua规定，wrap函数的参数是LuaState*这样一个参数，这里就是lua中实现的c的调用过程，在调用wrap函数时，会分配一个LuaState（可以理解为一个栈），将参数放到LuaState中(压栈过程)，在wrap_add函数中，从LuaState取到参数并检查类型，最后将返回值压栈并返回返回值个数（Lua中支持多个返回值）。


这样我们在Lua中就可以以myModule.add(1,2);的方式调用c的接口（在这之前还需要建立一张表，如table = {“add”,wrap_add}，来告诉lua解释器函数名的映射）。当然，内部需要lua的运行时支持对c函数的调用，才能调用到wrap_add接口（关于这个过程涉及到更底层的abi层面的东西）。


了解Lua对c的调用，我们再回过头来思考Qt中Javascript对C++调用这个问题，我们考虑一下前面所提到的两个问题：

（1）数据类型的识别（包括数据的存储格式，访问方式）。

（2）数据类型的转换（类型系统如何互通）。

对象是采用C++中的类创建的，也是按照C++的数据存储格式存储的，JavaScript要想去访问，就必须要了解这一切，才能够认识C++中的数据，进而才能做数据类型转换的事情。那么JavaScript如何才能知道这些呢？答案只有一个：Meta-Object System。想想为了让JS能够认识C++的类型，我们做了什么？类必须由QObject继承而来，类中必须使用Q_OBJECT宏，如果想要在QML中使用该类型还必须调用qmlRegisterType进行类型注册，这一切都是为了让JS引擎更懂我们定义的类型，也就是让JS引擎知道我们定义类型的元信息，有了元信息，Qt就可以帮我们做很多像Lua要求c做的事情，而这些都不需要用户来操心了。

4、C++ 调用 JS

我想这又是FFI中所说的，“Many FFIs also provide the means for the called language to invoke services in the host language as well.”

Qt中，C++创建JS的对象的方式如下:   

```
// Using QQmlComponent  
   QQmlEngine engine;  
   QQmlComponent component(&engine,  
   QUrl::fromLocalFile(“MyItem.qml”));//若要动态创建，可以使用 QUrl::setData接口  
   QObject *object = component.create();  
   ...  
   delete object;  

```

```
//读写属性  
rx2 = object->property(“qmlObjProperty");  
object->setProperty("qmlObjProperty", wx2);  
```

//调用方法

```
QMetaObject::invokeMethod(object, "myMethodInt",  
                       Q_RETURN_ARG(QVariant, returnedValue),  
                       Q_ARG(QVariant,100));  
```

C++去操作JavaScript，也需要关心类似的事情：类型的识别，类型的转换。

QML中的对象的类型分为两种：

（1）一种是C++中定义的类型，注册到QML中。这种对象的数据实际是按照C++的格式布局，这种对象从JavaScript中传到C++中，C++只要做一下指针类型转换，就可以像操作C++对象一样使用了。

（2）另一种是JavaScript类型的对象。这种类型的对象，不能直接转换为C++对象使用，Qt中提供一种类型QJSValue，用于包装JavaScript类型的对象。



如果我们并不知道从QML中创建的对象是上面的哪一种，或者我们就不关心它是哪一种，我们就可以按照上面代码的方式去统一操作，这里的细节都被封装在了Component中，而不需要用户去操心这些。但是，这样我们也必须遵照Component创建对象的操作规则，属性访问只能通过getter和setter接口，方法调用只能通过invokeMethod接口。这里需要说明的是，这里需要JS提供一些支持，例如根据属性名找到对应的属性和方法，从外部去invoke一个方法等（Qt从Qt 5.2版本后采用Qt自己实现的V4JS引擎，之前使用v8引擎）。






##补充：关于JavaScript调用C++定义函数的参数传递

关于Qt中JavaScript和C++混合调用参数的内部的自动类型转换，可以参考:http://qt-project.org/doc/qt-5/qtqml-cppintegration-data.html，这里我们只探讨js中定义的对象作为参数传递给c++时的情况。



调用方式：

```
cOjb.foo(param);//其中cObj为C++类型的对象（如上文描述，该对象可能是在c代码中创建然后挂在js的全局上下文中供js访问，也可能是在js中创建）  
```


C++中函数定义方式：

```
void MyClass::foo(<Type> cParam){  
//do something  
}  
```

分为以下几种情况：

js中定义的对象作为函数参数传递给c++函数

|C++函数形参Type | C++函数形参Type:QObject * | C++函数形参Type:QJSValue | C++函数形参Type:QVariantMap|
|:--:|:--:|:--:|:--:|
|js中实参param类型:QML Base Type(我只试了Qt.rect)|不支持，QObject * = 0；|支持，通过propert方法获取属性|不支持，QVariantMap为空|
|js中实参param类型param:JS字面量对象|不支持，QObject * = 0；|支持，通过propert方法获取属性|支持，属于值传递|
|js中实参param类型param:c++注册进来的类型定义的对象或者QML元素|支持，可以将QObject*强转成指定类型的指针使用|支持，通过propert方法获取属性|支持，属于值传递|

