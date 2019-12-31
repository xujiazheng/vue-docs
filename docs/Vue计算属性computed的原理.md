# Vue的computed计算属性

+ [为什么computed](#为什么computed)
+ [使用computed](#使用computed)
+ [从入口开始](#从入口开始)
+ [计算属性的数据驱动](#计算属性的数据驱动)
    + [defineComputed](#defineComputed)
    + [createComputedGetter](#createComputedGetter)
+ [计算属性Watcher简述](#计算属性Watcher简述)
    + [Watcher构造函数](#Watcher构造函数)
    + [watcher.depend](#watcher.depend)
    + [watcher.evaluate](#watcher.evaluate)
+ [总结](#总结)
+ [结尾](#结尾)

## 为什么computed

vue的computed会自动处理一些计算赋值给某个属性，在某些场景是很有用的。举个例子：

有一个业务场景：有一个表单填写重量、单价、件数，重量<= 9，单价>= 0，件数为偶数才能提交，显示按钮，否则不展示按钮

如果没有computed
```html
<div id="app">
  <div><label for="">重量</label><input v-model="num1" type="text"></div>
  <div><label for="">单价</label><input v-model="num2" type="text"></div>
  <div><label for="">件数</label><input v-model="num3" type="text"></div>
  <button v-show="num1 <= 9 && num2 >= 0 && num3 % 2 === 0 ">提交</button>
</div>
```
html中使用了很长的逻辑判断去控制按钮的显示，这样违背了开发原则：js中控制逻辑；

如果用computed
```html
<div id="app">
  <div><label for="">重量</label><input v-model="num1" type="text"></div>
  <div><label for="">单价</label><input v-model="num2" type="text"></div>
  <div><label for="">件数</label><input v-model="num3" type="text"></div>
  <button v-show="isValid ">提交</button>
</div>
```
利用computed计算一个isValid属性，来控制按钮的展示与隐藏，这样代码更优雅。

## 使用computed
上面的栗子中，我们定义的computed如下：
```javascript
new Vue({
  el: "#app",
  data: {
    num1: 0,
    num2: 0,
    num3: 0
  },
  computed: {
    isValid() {
      const {num1, num2, num3} = this;
      return num1 <= 9 &&  num2 >= 0 && num3 % 2 === 0;
    }
  }
})
```
通过computed，进行逻辑判断得出isValid的值，并且当3个值任意一个改变后，会自动计算出。那vue是怎样实现的这样的功能的呢？下一节将进行分析。

## 从入口开始

new Vue时，首先vue执行了_init方法，在这里做了vue的初始化工作，其中执行了一个initState函数，最终执行了initComputed函数，如下：

```javascript
var computedWatcherOptions = { computed: true };
function initComputed (vm, computed) {
  // 创建watchers对象缓存watcher
  var watchers = vm._computedWatchers = Object.create(null);
  for (var key in computed) {
    // 计算属性的执行函数/get、set描述对象
    var userDef = computed[key];
    var getter = typeof userDef === 'function' ? userDef : userDef.get;
    watchers[key] = new Watcher(
      vm,
      getter || noop,
      noop,
      computedWatcherOptions
    );
    if (!(key in vm)) {
      defineComputed(vm, key, userDef);
    }
  }
}
function initState (vm) {
  var opts = vm.$options;
  // data数据绑定，数据驱动核心
  if (opts.data) {
    initData(vm);
  } else {
    observe(vm._data = {}, true /* asRootData */);
  }
  // 计算属性绑定
  if (opts.computed) { initComputed(vm, opts.computed); }
}
```
initComputed函数中做了2件重要的事：
1. 计算属性的Watcher-对每一个computed属性都new Watcher并保存了起来
2. 计算属性的数据驱动-对每一个没有进行data数据绑定的computed属性进行行为劫持，并作出对应的反馈。（因此data声明过的属性，再声明computed是不生效的）

接下来就重点看一下这2部分内容。

## 计算属性的数据驱动

defineComputed方法做的事情就是将计算属性加入到数据驱动去

### defineComputed

```javascript
function defineComputed (
  target,
  key,
  userDef // 我们定义的computed执行函数
) {
  var shouldCache = !isServerRendering();
  // 如果我们定义了函数，将他作为属性的get方法，set设置为空函数
  if (typeof userDef === 'function') {
    sharedPropertyDefinition.get = shouldCache
      ? createComputedGetter(key) // 创建计算属性的get方法
      : userDef;
    sharedPropertyDefinition.set = noop;
  } else {
    // 如果我们传入的不是函数，那么就应该是一个对象，包含着get和set，将我们传入的函数进行赋值
    sharedPropertyDefinition.get = userDef.get
      ? shouldCache && userDef.cache !== false
        ? createComputedGetter(key)
        : userDef.get
      : noop;
    sharedPropertyDefinition.set = userDef.set
      ? userDef.set
      : noop;
  }
  // 对计算属性进行行为劫持
  Object.defineProperty(target, key, sharedPropertyDefinition);
}
```
通过以上代码，可以看出，defineComputed对computed属性进行了get和set的方法重写，createComputedGetter方法就是核心，是computed实现的原理。

### createComputedGetter

createComputedGetter方法代码很简单，他返回一个方法，这个返回的方法就是作为computed属性的get方法。
```javascript
function createComputedGetter (key) {
  return function computedGetter () {
    var watcher = this._computedWatchers && this._computedWatchers[key];
    if (watcher) {
      watcher.depend();
      return watcher.evaluate()
    }
  }
}
```
从以上代码看出，当computed属性get触发时，他获取到当前computed的watcher，并且触发了watcher的depend方法，最终返回了watcher.evaluate方法的执行结果。从这里可以得出结果：

1. 计算属性的值，是根据自身watcher去获取的


2. 计算属性的实现与Watcher有必然的联系


## 计算属性Watcher简述

在前面提到过，每一个计算属性都会new Watcher并保存下来，代码如下：
```javascript
watchers[key] = new Watcher(
    vm,
    getter || noop, // 定义的计算属性执行方法
    noop,
    computedWatcherOptions // {computed: true}
);
```
这里new Watcher传入了4个参数给Watcher

* vm 当前vue实例
* getter 定义的计算属性执行方法
* noop 空函数
* 一个对象{computd: true}

new Watcher时，根据传入的参数，生成一个watcher实例，接下来分析一下计算属性的Watcher

### Watcher构造函数
下面是经过删减，只保留computd相关的Watcher构造函数
```javascript
var Watcher = function Watcher (
  vm,
  expOrFn,
  cb,
  options
) {
  this.vm = vm;
  vm._watchers.push(this);
  // options
  if (options) {
    this.computed = !!options.computed;
  }
  this.cb = cb;
  this.active = true;
  this.dirty = this.computed; // for computed watchers
  this.deps = [];
  this.newDeps = [];
  this.depIds = new _Set();
  this.newDepIds = new _Set();
  this.expression = expOrFn.toString();
  // parse expression for getter
  if (typeof expOrFn === 'function') {
    this.getter = expOrFn;
  }
  if (this.computed) {
    this.value = undefined;
    this.dep = new Dep();
  }
};
```
不难看出，计算属性的Watcher，使得`this.computed = true`、 `this.dirty = true`、`this.getter = computed执行函数`、`this.dep = 派发器`

好了，了解到了计算属性Watcher实例的一些特性，接下来将看一下重点内容.

### watcher.depend

在前面章节[createComputedGetter](#createComputedGetter)里介绍到，computed属性get时执行了自己watcher的depend方法，方法内容如下：
```javascript
Watcher.prototype.depend = function depend () {
  if (this.dep && Dep.target) {
    this.dep.depend();
  }
};
```
这里执行了this.dep.depend方法，通过源码得出，最终执行了Dep.target.addDep方法，作用是把Dep.target这个Watcher收集到自己的dep中，也就是计算属性的dep派发器收集Dep.target作为自己的依赖。这里Dep.target其实是view渲染的Watcher，这里不做讲解。

### watcher.evaluate

```javascript

Watcher.prototype.evaluate = function evaluate () {
  if (this.dirty) {
    this.value = this.get();
    this.dirty = false;
  }
  return this.value
};

```
这个函数可以看出，this.dirty为真时，会触发this.get，this.get就是我们定义的computed执行函数，此后，直接返回this.value

```javascript
Watcher.prototype.get = function get () {
  // 加入Dep.target
  pushTarget(this);
  var value;
  var vm = this.vm;
  try {
    value = this.getter.call(vm, vm); // 执行
  } catch (e) {
  } finally {
    popTarget();
    this.cleanupDeps();
  }
  return value
};
```
执行watcher.get首先会执行`pushTarget(this)`，通过代码可以看出他的目的是将当前Watcher赋值给Dep.target，为什么这样做，后面会介绍。

接下来this.getter执行，执行了我们声明的computed执行函数如下：
```javascript
isValid() {
  const {num1, num2, num3} = this;
  return num1 <= 9 &&  num2 >= 0 && num3 % 2 === 0;
}
```
这里，isValid对应的执行函数执行时，会触发num1、num2、num3的get，从vue的数据驱动知识，我们知道，在此时data属性会将Dep.target作为依赖收集到自己的dep派发器，在这里Dep.target就是isValid这个计算属性的Watcher。

## 总结

* 计算属性会new Watcher来监听行为，并会Object.defineProperty进行行为劫持
* 计算属性在第一次发生get行为时，会主动执行执行函数，一方面获取到初始值，另一方面将自己所依赖的data（num1、num2、num3）收集到自己的Watcher作为依赖，这样在data改变时，会分发事件通知到自身Watcher进行值的变更
* 计算属性在第二次get行为时，就不需要计算了，直接获取值，因为在num1或其他data改变时，会触发computed的Watcher去自动计算出值

## 结尾

* vue版本v2.5.17-beta.0
* 本文只针对computed做分析，数据驱动部分在其他章节有详情
* 本文不考虑其他的场景，因此贴出代码有一定删减