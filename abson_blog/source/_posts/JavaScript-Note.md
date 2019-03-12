---
title: JavaScript学习笔记
date: 2018-11-07 21:03:45
tags:
---

JavaScript 是一门基于原型的语言，不存在类与实例的区别，因为它只有对象。基于原型的语言具有所谓原型对象的概念。原型对象可以作为一个模板，新对象可以从中获得原始的属性。任何对象都可以指定其自身的属性，既可以是创建时也可以在运行时创建。而且，任何对象都可以作为另一个对象的原型，从而允许后者共享前者的属性。

JavaScript 的属性查找机制首先在对象自身的属性中查找，如果指定的属性名称没有找到，将在对象的特殊属性 __proto__ 中查找。这个过程是递归的；被称为“在原型链中查找”。

特殊的 __proto__ 属性是在构建对象时设置的；设置为构造器的 prototype 属性的值。所以表达式 new Foo() 将创建一个对象，其 __proto__ == Foo.prototype。因而，修改 Foo.prototype 的属性，将改变所有通过 new Foo() 创建的对象的属性的查找。

### 开发方法笔记

#### 观察者模式 Proxy 和 Reflect 应用
```javascript
const queuedObservers = new Set();
const observe = fn => queuedObservers.add(fn);
const observable = obj => new Proxy(obj, {set});
function set (target, key, value, receiver) {
	const result = Reflect.set(target, key, value, receiver);
	queuedObservers.forEach(observe => observe());
}

class Person {
	cons
}
const person = observerable()

```

### setTimeout
```javascript
setTimeout(function () {
  console.log('three');
}, 0);
console.log('one');

// one
// three
```
上面代码中，setTimeout(fn, 0)在下一轮“事件循环”开始时执行.

### yield
yield 感觉像协程。每次遇到 yield，函数暂停执行，下一次再从该位置继续向后执行。
```javascript
function* gen() {
  yield  123 + 456;
}
```

### obj != null && obj != undefined
快速判断方法 
```typescript
function isNullOrUndefined(obj) {
	return !!obj;
}
```

#### Object.defineProperty(exports, "__esModule", { value: true });
在一些第三方库中经常看到这样的代码
``` typescript
Object.defineProperty(exports, "__esModule", {
  value: true
});
```
给模块的输出对象增加 __esModule 是为了将不符合 Babel 要求的 CommonJS 模块转换成符合要求的模块，这一点在 require 的时候体现出来。如果加载模块之后，发现加载的模块带着一个 __esModule 属性，Babel 就知道这个模块肯定是它转换过的，这样 Babel 就可以放心地从加载的模块中调用 exports.default 这个导出的对象，也就是 ES6 规定的默认导出对象，所以这个模块既符合 CommonJS 标准，又符合 Babel 对 ES6 模块化的需求。然而如果 __esModule 不存在，也没关系，Babel 在加载了一个检测不到 __esModule 的模块时，它就知道这个模块虽然符合 CommonJS 标准，但可能是一个第三方的模块，Babel 没有转换过它，如果以后直接调用 exports.default 是会出错的，所以现在就给它补上一个 default 属性，就干脆让 default 属性指向它自己就好了，这样以后就不会出错了。
什么是 [CommonJS](https://github.com/seajs/seajs/issues/269)

>CommonJS 规范是为了解决 JavaScript 的作用域问题而定义的模块形式，可以使每个模块它自身的命名空间中执行。该规范的主要内容是，模块必须通过 module.exports 导出对外的变量或接口，通过 require() 来导入其他模块的输出到当前模块作用域中。

### InteractionManager 设置超时时间
```
// CustomInteractionManager.js
import { InteractionManager } from "react-native";

function timeout(ms: number) {
  return new Promise((resolve: any) => setTimeout(resolve, ms));
}

export default {
  ...InteractionManager,
  runAfterInteractions: async (task?: (() => any)) => {
    let called = false;
    const time = timeout(500).then(() => {
      if (called) return;
      called = true;
      task && task();
    });
    const promise = InteractionManager.runAfterInteractions().then(() => {
      if (called) return;
      called = true;
      task && task();
    });
    await Promise.race([promise, time]);
  }
};

// index.js
await YZInteractionManager.runAfterInteractions();
// other code

or

YZInteractionManager.runAfterInteractions().then(()=>{
 	// other code
})

```


