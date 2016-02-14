---
title: "JavaScript中__proto__与prototype"
layout: post
date: 2014-09-14 19:51
tags: [前端开发,JavaScript,Prototype]
#star: true
---
 `版权声明：本文为博主原创文章，未经博主允许不得转载。`

测试环境：Google Chrome 36.0.1985.125

如有不足，欢迎指正。


闲话少说，先上图。

<!--![](./20140914/img1.jpeg)-->
{% include image.html
            img="assets/images/20140914/img1.jpeg"
            title="title for image"
            caption="caption for image" %}


\_\_proto\_\_:对象的原型，JavaScript中每个对象都有一个\_\_proto\_\_属性，用于基于原型链的继承机制，当访问一个对象的属性时，会现在自身属性中查找，如果未找到，则会通过\_\_proto\_\_到原型链中去查找，如果自身属性中找到，则不会继续在原型链中查找该属性。



prototype:函数的原型，利用new foo()的方式创建对象时，会将函数的prototype赋值给所创建对象的\_\_proto\_\_。

`简言之，JavaScript中所有对象都有__proto__属性，prototype是function对象所独有。`

如上图所示，对照上图来说明一下。

1、如果不作额外修改，默认情况下，所有被创建者的\_\_proto\_\_都指向创建者的prototype：



（1） 如果不作额外修改，默认情况下，所有function的\_\_proto\_\_（因为function是函数对象，所有它有\_\_proto\_\_属性）都指向Function的prototype，包括Function自身。

（2）如果不作额外修改，默认情况下，所有对象的\_\_proto\_\_都指向用来创建它的function对象的prototype。



2、一个对象上的属性包括自身定义的属性和原型链上的属性，这对于该对象外部使用者来说是透明的，平坦的，都可以直接访问，还可以通过hasOwnProperty来区分自身属性和原型链属性。



当然，JavaScript很灵活，我们可以通过代码随时修改对象的\_\_proto\_\_和函数的prototype,但是有以下几个特殊情况：

####（1）Function.prototype不允许做任何修改，对其赋值将不会产生任何作用。

####（2）普通function.prototype可以赋任何值，都会赋值成功，但影响却不一样。看如下代码：

<pre>
<code>
Object.prototype.a = 100;  
function foo(){};  
</code>
</pre>

情况一：不修改foo.prototype

<pre>
<code>
var o = new foo();  
o.__proto__ === > {a:100};  
//这是因为：  
foo.prototype.__proto__ == Object.prototype; ===> foo.prototype : {a:100}  //对于外部使用者原型链属性平坦化  
  
o.__proto__ === foo.prototype；===> o.__proto__: {a:100}   
</code>
</pre>


情况二：修改foo.prototype为自定义对象

<pre>
<code>
var p1 =  {b:200}  
p1.__proto__ ===> {a:100}    //通过字面量创建的对象，__proto__指向Object.prototype  
       foo.prototype = p1;  
foo.prototype ===> {b: 200, a: 100}  //p1对于外部使用者原型链属性平坦化  
  
var o1 = new foo();  
o1.__proto__ === > {b: 200, a: 100} //因为o1.__proto__ === foo.prototype；  
</code>
</pre>

情况三：修改foo.prototype为自定义对象
<pre>
<code>
foo.prototype = 300;  
foo.prototype ===> 300  //修改成功  
  
var o2 = new foo();  
o2.__proto__ === > { a: 100}   
//按理来说应该是o2.__proto__ === foo.prototype，o2.__proto__应该为300，为什么确实{a:100}呢？这不正是Object.prototype么？  
o2.__proto__ === Object.prototype ===> true //测试一下,果然如此。  
  
//这说明，在var o2 = new foo();  时，引擎发现foo.prototype不是一个对象，而强行将Object.prototype赋值给了o2.__proto__。引擎为什么要这么做呢？我想这是因为JavaScript中对象的属性访问机制（如果对象自身找不到该属性，就去原型链中找），所以要保证o2.__proto__必须是个对象
</code>
</pre>

####(3)修改对象的\_\_proto\_\_只能是对象类型，否则修改不成功，\_\_proto\_\_保持原值

<pre>
<code>
var foo = function(){}  
foo.prototype = {a:100}  
  
var x = new foo;  
x ===>  {a: 100}  
  
x.__proto__ = true  
x.__proto__  ===>  {a: 100}                //failed  
  
x.__proto__ = 1  
x.__proto__  ===> {a: 100}                 //failed  
  
x.__proto__ = {b:200}  
x.__proto__  ===> Object {b: 200}     //succeed  
</code>
</pre>

####(4)Function.prototype是唯一一个类型为“function”的prototype，其他函数的prototype类型都是object。
<pre>
<code>
typeof Function.prototype ===> “function”  
typeof Object.prototype ===> “object”  
</code>
</pre>

###补充：

1、为什么要有prototype和\_\_proto\_\_？

主要是为了原型链继承的数据的共有和私有。函数对象将自身属性放到构造函数中，将共有属性放到prototype中，这样由自身创建的对象将具有所有属性，而继承下来的函数对象所创建的对象只具有prototype中的属性。

这个问题至于是否还有其他原因，还有待学习。

<pre>
<code>
var Base(){  
  this.a = 100;  
}  
Base.prototype = {b:200};  
  
var Derive(){  
  this.c = 300;  
}  
Derive.prototype = Base.prototype;  
  
var o1 = new Base();  
var o2 = new Derive();  
  
o1.a ===> 100  
o1.b ===> 200  
o2.a ===> undefined  
o2.b ===> 200  
o2.c ===> 300  
  
Base.prototype.b = 400;  
o1.b ===> 400  
o2.b ===> 400  
  
//这里也印证了之前的猜想：要保证o.__proto__必须是个对象。  
</code>
</pre>

2、new的背后做了什么？
<pre>
<code>
function Foo(){}  
var o=new Foo();  
  
//类似于：  
function Foo(){}  
var o = {}; //{}也是个对象，它的__proto__ === Object.prototype  
o.__proto__= Foo.prototype;    //真正的实现，此处还会判断Foo.prototype是否是对象类型  
Foo.call(o); 
</code>
</pre>