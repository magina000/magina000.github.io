---
title: 详解js中的this
date: 2019-08-15 10:56:02
categories: 学习
tags: [javascript]
---

### 主要解释在JS里面this关键字的指向问题(在浏览器环境下)

#### 函数的几种调用方式
+ <font size=3 > 普通函数调用 </font>
+ <font size=3 > 作为方法来调用 </font>
+ <font size=3 > 作为构造函数来调用 </font>
+ <font size=3 > 使用apply/call方法来调用 </font>
+ <font size=3 > Function.prototype.bind方法 </font>
+ <font size=3 > es6箭头函数 </font>

<font size=3 >**重点：谁调用这个函数或方法,this关键字就指向谁。**</font>

###### <font size=4 > *普通函数调用* </font>
```
function person(){
  this.name="xl";
  console.log(this);
  console.log(this.name);
}

person();  //输出  window  xl   
```
在这段代码中`person`函数作为普通函数调用，实际上`person`是作为全局对象`window`的一个方法来进行调用的,即`window.person()`;
所以这个地方是`windo`w对象调用了`person`方法,那么`person`函数当中的`this`即指`window`,同时`window`还拥有了另外一个属性`name`,值为`xl`。

###### <font size=4 > *作为方法调用* </font>
```
var name="XL";
var person={
  name:"xl",
  showName:function(){
      console.log(this.name);
  }
}
person.showName();  //输出  xl
```
这里是`person`对象调用`showName`方法，很显然`this`关键字是指向`person`对象的，所以会输出`name`。
```
var showNameA=person.showName;
showNameA();    //输出  XL
```
这里将`person.showName`方法赋给`showNameA`变量，此时`showNameA`变量相当于`window`对象的一个属性，因此`showNameA()`执行的时候相当于`window.showNameA()`,即`window`对象调用`showNameA`这个方法，所以`this`关键字指向`window`。
<font size=2 > **换种写法:** </font>
```
var personA={
  name:"xl",
  showName:function(){
      console.log(this.name);
  }
}
var personB={
  name:"XL",
  sayName:personA.showName
}

personB.sayName();  //输出 XL
```
 虽然`showName`方法是在`personA`这个对象中定义，但是调用的时候却是在`personB`这个对象中调用，因此`this`对象指向`personB`。

###### <font size=4 > *作为构造函数调用* </font>
```
function  Person(name){
  this.name=name;
}
var personA=Person("xl");
console.log(personA.name); // 输出  undefined
console.log(window.name);//输出  xl
```
上面代码没有进行`new`操作，相当于`window`对象调用`Person("xl")`方法，那么`this`指向`window`对象，并进行赋值操作`window.name="xl"`。
<font size=2 > **new操作符：** </font>
```
//下面这段代码模拟了new操作符(实例化对象)的内部过程
function person(name){
  var o={};
  o.__proto__=Person.prototype;  //原型继承
  Person.call(o,name);
  return o;
}
var personB=person("xl");

console.log(personB.name);  // 输出  xl
```
+ <font size=2> 在`person`里面首先创建一个空对象`o`，将`o`的`proto`指向`Person.prototype`完成对原型的属性和方法的继承 </font>
+ <font size=2> `Person.call(o,name)`这里即函数`Person`作为`apply/call`调用(具体内容下方)，将`Person`对象里的`this`改为`o`，即完成了`o.name=name`操作 </font>
+ <font size=2> 返回对象`o`。 </font>

###### <font size=4 > *call/apply方法的调用* </font>
在JS里函数也是对象，因此函数也有方法。
从`Function.prototype`上继承到`Function.prototype.call/Function.prototype.apply`方法
call/apply方法最大的作用就是能改变this关键字的指向。

`Obj.method.apply(AnotherObj,arguments);`

```
var name="XL";
var Person={
  name:"xl",
  showName:function(){
      console.log(this.name);
  }
}
Person.showName.call(); //输出 "XL"
//这里call方法里面的第一个参数为空，默认指向window。
//虽然showName方法定义在Person对象里面，但是使用call方法后，将showName方法里面的this指向了window。因此最后会输出"XL";
```
<font size=2 > **举例：** </font>
```
funtion FruitA(n1,n2){
  this.n1=n1;
  this.n2=n2;
  this.change=function(x,y){
      this.n1=x;
      this.n2=y;
  }
}

var fruitA=new FruitA("cheery","banana");
var FruitB={
  n1:"apple",
  n2:"orange"
};
fruitA.change.call(FruitB,"pear","peach");

console.log(FruitB.n1); //输出 pear
console.log(FruitB.n2);// 输出 peach
//FruitB调用fruitA的change方法，将fruitA中的this绑定到对象FruitB上。
```

###### <font size=4 > *Function.prototype.bind()方法* </font>
```
var name="XL";
function Person(name){
  this.name=name;
  this.sayName=function(){
      setTimeout(function(){
          console.log("my name is "+this.name);
      },50)
  }
}
var person=new Person("xl");
person.sayName()  //输出  “my name is XL”;
```
这里的`setTimeout()`定时函数,相当于`window.setTimeout()`,由`window`这个全局对象对调用,因此`this`的指向为`window`, 则`this.name`则为`XL`。
那么如何才能输出`"my name is xl"`呢？
```
var name="XL";
function Person(name){
  this.name=name;
  this.sayName=function(){
      setTimeout(function(){
          console.log("my name is "+this.name);
      }.bind(this),50)  
      //注意这个地方使用的bind()方法，绑定setTimeout里面的匿名函数的this一直指向Person对象
  }
}
var person=new Person("xl");
person.sayName(); //输出 “my name is xl”;
```
这里`setTimeout(function(){console.log(this.name)}.bind(this),50);`,匿名函数使用`bind(this)`方法后创建了新的函数，这个新的函数不管在什么地方执行，`this`都指向`Person`,而非`window`,因此最后的输出为`"my name is xl"`而不是`"my name is XL"`。

<font size=2 > **另外几个需要注意的地方：** </font>
`setTimeout/setInterval/匿名函数`执行的时候，`this`默认指向`window`对象，除非手动改变`this`的指向。在《javascript高级程序设计》当中，写到：**“超时调用的代码(setTimeout)都是在全局作用域中执行的，因此函数中的this的值，在非严格模式下是指向window对象，在严格模式下是指向undefined”**。本文都是在非严格模式下的情况。
<font size=2 > **举例：** </font>
```
var name="XL";
function Person(){
  this.name="xl";
  this.showName=function(){
      console.log(this.name);
  }
  setTimeout(this.showName,50);
}
var person=new Person(); //输出 "XL"
//在setTimeout(this.showName,50)语句中，会延时执行this.showName方法
//this.showName方法即构造函数Person()里面定义的方法。50ms后，执行this.showName方法，this.showName里面的this此时便指向了window对象。则会输出"XL";
```
<font size=2 > **修改上面的代码：** </font>
```
var name="XL";
function Person(){
  this.name="xl";
  var that=this;
  this.showName=function(){
      console.log(that.name);
  }
  setTimeout(this.showName,50)
}
var person=new Person(); //输出 "xl"
//这里在Person函数当中将this赋值给that，即让that保存Person对象
//因此在setTimeout(this.showName,50)执行过程当中，console.log(that.name)即会输出Person对象的属性"xl"
```
<font size=2 > **匿名函数：** </font>
```
var name="XL";
var person={
  name:"xl",
  showName:function(){
      console.log(this.name);
  }
  sayName:function(){
      (function(callback){
          callback();
      })(this.showName)
  }
}
person.sayName();  //输出 XL
```
<font size=2 > **改变this：** </font>
```
var name="XL";
var person={
  name:"xl",
  showName:function(){
      console.log(this.name);
  }
  sayName:function(){
      var that=this;
      (function(callback){
          callback();
      })(that.showName)
  }
}
person.sayName() ;  //输出  "xl"
//匿名函数的执行同样在默认情况下this是指向window的，除非手动改变this的绑定对象
```

###### <font size=4 > *箭头函数* </font>
`es6`里面`this`指向固定化，始终指向外部对象，因为箭头函数没有`this`,因此它自身不能进行`new`实例化,同时也不能使用`call, apply, bind`等方法来改变`this`的指向
```
function Timer() {
  this.seconds = 0;
  setInterval( () => this.seconds ++, 1000);
} 
var timer = new Timer();
setTimeout( () => console.log(timer.seconds), 3100);// 3  
//在构造函数内部的setInterval()内的回调函数，this始终指向实例化的对象，并获取实例化对象的seconds的属性,每1s这个属性的值都会增加1。否则最后在3s后执行setTimeOut()函数执行后输出的是0
```