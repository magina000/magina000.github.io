---
title: mvvm理解并简易实现数据双向绑定
date: 2020-05-28 15:01:20
categories: 学习
tags: [vue]
---

> 近段时间打算花点时间研究一些vue底层的代码，也顺便想为面试准备一下
## MVVM（Model-View-ViewModel)
+ <font size="3">__ViewModel__：内部集成了Binder(Data-binding Engine，数据绑定引擎)，在MVP中派发器View或Model的更新都需要通过Presenter手动设置，而Binder则会实现View和Model的双向绑定，从而实现View或Model的自动更新。</font>
+ <font size="3">__View__：可组件化，例如目前各种流行的UI组件框架，View的变化会通过Binder自动更新相应的Model。</font>
+ <font size="3">__Model__：Model的变化会被Binder监听(仍然是通过观察者模式)，一旦监听到变化，Binder就会自动实现视图的更新。</font>

## 发布订阅模式
**发布---订阅模式又叫观察者模式，它定义了对象间的一种一对多的关系，让多个观察者对象同时监听某一个主题对象，当一个对象发生改变时，所有依赖于它的对象都将得到通知。**
具体可以看另一篇<font size="3">**实现event**</font>就是典型的该模式的实现。

## 简易数据双向绑定实现（重点）
> tips: vue2.0数据劫持还是通过Object.defineProperty实现，不过vue3.0中已经采用Object.proxy来实现，具体区别以后分析

大致内容摘自 [<font color=#6971FE>github/DMQ</font>](https://github.com/DMQ/mvvm#_2) 
### 思路整理
<font size="2">已经了解到vue是通过数据劫持的方式来做数据绑定的，其中最核心的方法便是通过Object.defineProperty()来实现对属性的劫持，达到监听数据变动的目的，要实现mvvm的双向绑定，就必须要实现以下几点： 1、实现一个数据监听器Observer，能够对数据对象的所有属性进行监听，如有变动可拿到最新值并通知订阅者 2、实现一个指令解析器Compile，对每个元素节点的指令进行扫描和解析，根据指令模板替换数据，以及绑定相应的更新函数 3、实现一个Watcher，作为连接Observer和Compile的桥梁，能够订阅并收到每个属性变动的通知，执行指令绑定的相应回调函数，从而更新视图 4、mvvm入口函数，整合以上三者。</font>

### 个人理解
<!-- 个人理解补充（加深印象) -->
<font size="2">1. Observer数据监听时要对对象的所有属性进行遍历监听，每一个属性都自定义一个订阅器（dep）,将所有订阅者（watcher）都保存在订阅器中，当监听到数据变化时，通知订阅器中的所有订阅者。</font>
<font size="2">2. Complie扫描所有元素节点，在这次简易的demo中应该扫描出\{ \{ \} \}这样格式的文本节点和元素节点下的v-model的这类指令模板，所谓的数据双向绑定也就是"修改input的值->改变数据"，"数据改变->视图模板替换"，只实现其一也就是说只实现单向绑定。在扫描所有节点时，对指定属性查找出对应的上述节点时，自动创建订阅者（demo中也就是说有2个订阅者）</font>
<font size="2">3. Watcher我理解为这就是订阅者，当Observer发出通知数据已变化后，遍历dep中所有的订阅者执行对应的回调函数。</font>

### 代码展示
> 有个小技巧，在数据劫持时，利用订阅器Dep的某个属性（targets）临时赋值Watcher，添加到dep中，添加完后再删除这个属性,因为在new Watcher阶段不太方便在Watcher的原型中取出this并添加到dep中,所以采用new阶段时 调用监听数据的getter方法时添加（具体在代码中会详细备注）

#### observer.js
```
//数据劫持构造函数
function Observe(data) {
  this.data = data;
  this.init(data);
}

Observe.prototype = {
  init: function (data) {
    var _this = this;
    Object.keys(data).forEach((key) => {
      _this.convert(key, data[key]);
    })
  },
  convert(key, val) {
    this.defineReactive(this.data, key, val);
  },
  defineReactive(data, key, val) {
    var dep = new Dep(); //定义一个订阅器
    observe(val);  // 递归继续向下找，实现深度的数据劫持
    Object.defineProperty(data, key, {
      configurable: true,
      enumerable: true,
      get: function () {
        Dep.target && dep.addWatcher(Dep.target);  //这一行在new Watcher阶段触发，将订阅者添加到订阅器中
        return val;
      },
      set: function (newVal) {
        if (newVal == val) return;
        val = newVal;
        observe(newVal); // 当设置为新值后，也需要把新值再去定义成属性
        dep.notify(); // 数据变化，通知所有订阅者
      }
    })
  }
}

function observe(value) {
  if (!value || typeof value !== 'object') {
    return;
  }
  return new Observe(value);
};

//订阅器构造函数
function Dep() {
  this.watchers = [];
}

Dep.prototype = {
  addWatcher: function (watcher) {
    this.watchers.push(watcher);
  },
  notify: function () {
    this.watchers.forEach(function (watcher) {
      //所有订阅者执行对应的更新函数
      watcher.update();
    });
  }
}
```

#### complie.js
```
function Compile(el, mvvm) {
  this.$vm = mvvm;
  this.$el = document.querySelector(el);
  if (this.$el) {
    //因为遍历解析的过程有多次操作dom节点，为提高性能和效率，
    //会先将vue实例根节点的el转换成文档碎片fragment进行解析编译操作，解析完成，
    //再将fragment添加回原来的真实dom节点中
    this.$fragment = this.node2Fragment(this.$el);
    this.init();
    this.$el.appendChild(this.$fragment);
  }
}

Compile.prototype = {
  node2Fragment: function (el) {
    var fragment = document.createDocumentFragment(),
      child;
    // 将原生节点拷贝到fragment
    while (child = el.firstChild) {
      fragment.appendChild(child);
    }
    return fragment;
  },
  init: function () {
    this.compileElement(this.$fragment);
  },
  compileElement: function (el) {
    var childNodes = el.childNodes,
      _this = this;
    [].slice.call(childNodes).forEach((node) => {
      var text = node.textContent;
      var reg = /\{\{(.*)\}\}/;

      if (_this.isElementNode(node)) { //扫描到元素节点
        let nodeAttrs = node.attributes;
        [].slice.call(nodeAttrs).forEach((attr) => {
          let name = attr.name;
          let exp = attr.value;
          if (name.includes('v-')) { //如果元素节点包含指令
            node.value = _this.$vm[exp]; 
          }
          // 监听变化
          new Watcher(_this.$vm, exp, function (newVal) {
            node.value = newVal;   // 当watcher触发时会自动将内容放进输入框中
          });

          node.addEventListener('input', e => {
            let newVal = e.target.value;
            // 相当于给this.c赋了一个新值
            // 而值的改变会调用set，set中又会调用notify，notify中调用watcher的update方法实现了更新
            _this.$vm[exp] = newVal;
          });
        })
      } else if (_this.isTextNode(node) && reg.test(text)) { //扫描到相同的文本节点{{exp}}
        //RegExp.$1表示正则校验第一个匹配值 比如文本节点{{obj.a.b}} 
        //RegExp.$1就是obj，RegExp.$2就是a
        let arr = RegExp.$1.split('.');
        let val = _this.$vm;
        arr.forEach(key => {
          val = val[key];     // 如this.a.b
        });
        // 用trim方法去除一下首尾空格
        node.textContent = text.replace(reg, val).trim(); //初始化替换数据
        // new订阅者，节点的回调执行为文本替换
        new Watcher(_this.$vm, RegExp.$1, newVal => {
          node.textContent = text.replace(reg, newVal).trim();
        });
      }
      // 如果还有子节点，继续递归扫描
      if (node.childNodes && node.childNodes.length) {
        _this.compileElement(node);
      }
    })
  },
  isElementNode: function (node) {
    return node.nodeType == 1;
  },
  isTextNode: function (node) {
    return node.nodeType == 3;
  }
}
```

#### watcher.js
```
function Watcher(vm, exp, cb) {
  this.$vm = vm;
  this.$exp = exp;
  this.$cb = cb;
  this.value = this.get();
}
Watcher.prototype = {
  get: function () {
    Dep.target = this;
    let value = this.$vm[this.$exp];	// 这里会触发属性的getter，从而添加订阅者
    Dep.target = null;
    return value
  },
  update: function () {
    var oldVal = this.value;
    let arr = this.$exp.split('.');
    let value = this.$vm;
    arr.forEach(key => {
      value = value[key];   // 通过get获取到新的值
    });
    if (value !== oldVal) {
      this.value = value;
      this.$cb.call(this.$vm, value); // 执行Compile中绑定的回调，更新视图
    }
  }
}
```

#### mvvm.js
```
// 创建一个Mvvm构造函数
// 这里用es6方法将options赋一个初始值，防止没传，等同于options || {}
function Mvvm(options = {}) {
  // vm.$options 将所有属性挂载到上面
  this.$options = options;
  // this._data 这里也和Vue一样
  let data = this._data = this.$options.data;

  let _this = this;
  Object.keys(data).forEach(function (key) {
    _this._proxy(key);
  })

  // 数据劫持
  observe(data);
  // 解析指令
  new Compile(options.el || document.body, this)
}

//数据代理 把mvvm.a等同于mvvm._data.a
Mvvm.prototype = {
  _proxy: function (key) {
    var _this = this
    Object.defineProperty(_this, key, {
      configurable: true,
      enumerable: true,
      get: function() {
        return _this._data[key]
      },
      set: function(newVal) {
        _this._data[key] = newVal
      }
    })
  }
}
```

#### index.html
```
<!DOCTYPE html>
<html lang="en">

<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Document</title>
</head>

<body>
  <div id="mvvm-app">
    <input type="text" v-model="name">
    <p>{{name}}</p>
  </div>

  <script src="./observe.js"></script>
  <script src="./watcher.js"></script>
  <script src="./compile.js"></script>
  <script src="./mvvm.js"></script>
  <script>
    let mvvm = new Mvvm({
      el: '#mvvm-app',
      data: {
        name: 'caimj',
      }
    });
    // console.log(mvvm);
    // mvvm.name = 'chenym'
  </script>
  </script>
</body>

</html>
```

### 简单总结
1.通过Object.defineProperty的get和set进行数据劫持，实现Observer
2.通过遍历节点解析指令添加订阅者
3.订阅者构造并塞入订阅器中，收到通知执行对应回调

核心思想：数据劫持+发布订阅模式


