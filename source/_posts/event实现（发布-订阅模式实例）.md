---
title: event实现（发布-订阅模式实例）
date: 2020-05-29 13:58:43
categories: 学习
tags: [框架]
---
> 通过event的实现，加深自己对发布-订阅模式的理解

其实js的addEventListener就是一种发布订阅模式的实现，对dom元素进行某个指令的监听。

### 基本构造

#### 构造函数
初始化Event的事件清单和监听者上限。
```
function EventEmeitter() {
  this._events = this._events || new Map(); // 储存事件/回调键值对
  this._maxListeners = this._maxListeners || 10; // 设立监听上限
}
```
#### 监听触发
触发监听函数我们可以用apply与call两种方法,在少数参数时call的性能更好,多个参数时apply性能更好。
```
// 触发名为type的事件
EventEmeitter.prototype.emit = function(type, ...args) {
  let handler;
  // 从储存事件键值对的this._events中获取对应事件回调函数
  handler = this._events.get(type);
  if (args.length > 0) {
    handler.apply(this, args);
  } else {
    handler.call(this);
  }
  return true;
};
// 监听名为type的事件
EventEmeitter.prototype.addListener = function(type, fn) {
  // 将type事件以及对应的fn函数放入this._events中储存
  if (!this._events.get(type)) {
    this._events.set(type, fn);
  }
};
```

#### 简单实践
```
// 实例化
const emitter = new EventEmeitter();

// 监听一个名为arson的事件对应一个回调函数
emitter.addListener('target', val => {
  console.log(`监听函数 ${val}`);
});

// 我们触发target事件,发现回调成功执行
emitter.emit('target', '123'); // 监听函数 123
```

### 问题->多个监听者
```
// 重复监听同一个事件名
emitter.addListener('target', man => {
  console.log(`监听函数1 ${man}`);
});
emitter.addListener('target', man => {
  console.log(`监听函数2 ${man}`);
});

emitter.emit('target', '123'); // 监听函数1 123
```
只会触发第一次监听，所以要在绑定时需要使用数组存储不同的监听函数。

#### 升级优化
addListener实现方法还不够健全,在绑定第一个监听者之后,就无法对后续监听者进行绑定了,因此需要将后续监听者与第一个监听者函数放到一个数组里.
```
// 监听名为type的事件
EventEmeitter.prototype.addListener = function(type, fn) {
  const handler = this._events.get(type);
  if (!handler) {
    this._events.set(type, fn);
  } else if (handler && typeof handler === 'function') {
    // 如果handler是函数说明只有一个监听者
    this._events.set(type, [handler, fn]); // 多个监听者我们需要用数组储存
  } else {
    if (handler.length === this._maxListeners) {
      return false;
    }
    handler.push(fn); // 已经有多个监听者,那么直接往数组里push函数即可
  }
};
```
在调用时进行监听者判断
```
EventEmeitter.prototype.emit = function(type, ...args) {
  let handler;
  handler = this._events.get(type);
  if (Array.isArray(handler)) {
    // 如果是一个数组说明有多个监听者,需要依次此触发里面的函数
    for (let i = 0; i < handler.length; i++) {
      if (args.length > 0) {
        handler[i].apply(this, args);
      } else {
        handler[i].call(this);
      }
    }
  } else { // 单个函数的情况我们直接触发即可
    if (args.length > 0) {
      handler.apply(this, args);
    } else {
      handler.call(this);
    }
  }
  return true;
};
```
问题解决！

#### 补充-移除监听
```
EventEmeitter.prototype.removeListener = function(type, fn) {
  const handler = this._events.get(type); // 获取对应事件名称的函数清单

  // 如果是函数,说明只被监听了一次
  if (handler && typeof handler === 'function') {
    this._events.delete(type, fn);
  } else {
    let position;
    // 如果handler是数组,说明被监听多次要找到对应的函数
    for (let i = 0; i < handler.length; i++) {
      if (handler[i] === fn) {
        position = i;
      } else {
        postion = -1;
      }
    }
    // 如果找到匹配的函数,从数组中清除
    if (position !== -1) {
      // 找到数组对应的位置,直接清除此回调
      handler.splice(position, 1);
      // 如果清除后只有一个函数,那么取消数组,以函数形式保存
      if (handler.length === 1) {
        this._events.set(type, handler[0]);
      }
    } else {
      return this;
    }
  }
};
```
小问题：匿名函数无法移除



