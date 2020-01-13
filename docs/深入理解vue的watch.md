# 深入理解vue的watch

vue中的wactch可以监听到data的变化，执行定义的回调，在某些场景是很有用的，本文将深入源码揭开watch额面纱

+ [前言](#前言)
+ [watch的使用](#watch的使用)
+ [watch的多种使用方式](#watch的多种使用方式)
    + [传值函数](#传值函数)
    + [传值数组](#传值数组)
    + [传值字符串](#传值字符串)
    + [传值对象](#传值对象)
    + [传值对象的其他作用](#传值对象的其他作用)
+ [源码分析watch](#源码分析watch)
    + [初始watch](#初始watch)
    + [创建Watcher](#创建watcher)
    + [watchWatcher](#watchwatcher)
    + [立即执行的watch](#立即执行的watch)
+ [与computed比较](#与computed比较)

## 前言

* version: v2.5.17-beta.0
* 阅读本文需读懂vue数据驱动部分

## watch的使用

当改变data值，同时会引发副作用时，可以用watch。比如：有一个页面有三个用户行为会触发this.count的改变，当this.count改变，需要重新获取list值，这时候就可以用watch轻松完成

```
new Vue({
  el: '#app',
  data: {
    count: 1,
    list: []
  },
  watch: {
    // 不管多少地方改变count，都会执行到这里去改变list的值
    count(val) {
      ajax(val).then(list => {
        this.list = list;
      })
    }
  },
  methods: {
    // 点击+1，count + 1，刷新列表
    handleClick() {
      this.count += 1;
    },
    // 点击重置，count = 1，刷新列表
    handleReset() {
      this.count = 1;
    },
    // 点击随机， count随机数，刷新列表
    handleRamdon() {
      this.count = Math.ceil(Math.random() * 10);
    }
  }
})
```

这样的好处就是把所有源头聚集到了watch中，不需要在多个count改变的地方手动去调用方法，减少代码冗余。

## watch的多种使用方式

watch的写法有多种，以上案例是最常见的一种方法，接下来介绍所有写法。

### 传值函数

```javascript
new Vue({
    data: {
        count: 1
    },
    watch: {
        count() {
            console.log('count改变')
        }
    }
})
```
最常见的写法，count改变时将会触发传值的回调函数

### 传值数组
```javascript
new Vue({
    data: {
        count: 1
    },
    watch: {
        count: [
            () => {
                console.log('count改变')
            },
            () => {
                console.log('count watch2')
            }
        ]
    }
})
```

传数组，count改变后会依次执行数组内每一个回调函数

### 传值字符串

```javascript
new Vue({
    data: {
        count: 1
    },
    watch: {
        count: 'handleChange'
    },
    methods: {
        handleChange(val) {
            console.log('count改变了')
        }
    }
})
```
我们也可以传值字符串handleChange，然后在methods写handleChange函数的逻辑，同样可以做到count改变执行`handleChange`

### 传值对象

```javascript
new Vue({
    data: {
        count: 1
    },
    watch: {
        count: {
            handler() {
                console.log('count改变')
            }
        }
    }
})
```

可以传值对象，该对象包含一个handler函数，当count改变时，会执行此handler函数，为什么多此一举需要包装一层对象呢？存在即合理，是有其特殊作用的。

### 传值对象的其他作用

watch为监听属性的变化，调用回调函数，因此，在初始化时，并不会触发，在初始化后属性改变才触发，如果想要初始时也要触发watch，那就需要传值对象，如下：

```javascript
new Vue({
    data: {
        count: 1
    },
    watch: {
        count: {
            immediate: true, // 加此属性
            handler() {
                console.log('count改变')
            }
        }
    }
})
```
传的对象有immediate属性为true，则watch会立刻触发。

## 源码分析watch

本节进行源码分析，探索watch的真面貌


### 初始watch
```javascript
// 初始化
function initState (vm) {
  vm._watchers = [];
  var opts = vm.$options;
  if (opts.data) {
    initData(vm);
  }
  if (opts.watch && opts.watch !== nativeWatch) {
    initWatch(vm, opts.watch);
  }
}
// watch初始化
function initWatch (vm, watch) {
  for (var key in watch) {
    var handler = watch[key];
    if (Array.isArray(handler)) {
      for (var i = 0; i < handler.length; i++) {
        createWatcher(vm, key, handler[i]);
      }
    } else {
      createWatcher(vm, key, handler);
    }
  }
}
```

从vue的执行流程，读到了initWatch函数，此函数的用法很清晰，将传入的每一个watch属性执行`createWatcher`处理。如果传值是数组，则遍历去调用。

下面看一下createWatcher函数
```javascript
function createWatcher (
  vm,
  expOrFn,
  handler,
  options
) {
  // 如果是对象，则处理
  if (isPlainObject(handler)) {
    // 将对象缓存，给$watch函数
    options = handler;
    handler = handler.handler;
  }
  if (typeof handler === 'string') {
    handler = vm[handler];
  }
  return vm.$watch(expOrFn, handler, options)
}
```
createWatcher中做了兼容处理：
1. 如果handler是个对象，则进行一步转换；
2. 如果handler是字符串，则取vue实例的方法（methods里声明）
3. 最后调用实例的$watch方法

### 创建Watcher

```javascript

Vue.prototype.$watch = function (
    expOrFn,
    cb,
    options
  ) {
    var vm = this;
    if (isPlainObject(cb)) {
      return createWatcher(vm, expOrFn, cb, options)
    }
    options = options || {};
    options.user = true;
    var watcher = new Watcher(vm, expOrFn, cb, options);
    if (options.immediate) {
      cb.call(vm, watcher.value);
    }
    return function unwatchFn () {
      watcher.teardown();
    }
  };
```

vm.$watch里是最终实现watch的部分，在这里仍然做了兼容判断，如果是对象，回调createWatcher；接下来就最重要的new Watcher。

$watch的功能其实就是new了一个Watcher，那么，我们在代码里实现的一切响应，都来自于Watcher，下面看一下watch里的Watcher

### watchWatcher

Watcher是vue数据驱动核心部分的一员，他承载着依赖收集与事件的触发。下面重点解读一下watch的Watcher实现。

```javascript
if (typeof expOrFn === 'function') {
    this.getter = expOrFn;
  } else {
    // parsePath去解析expOrFn并返回getter函数
    this.getter = parsePath(expOrFn);
    if (!this.getter) {
      this.getter = function () {};
    }
  }
```
watch Watcher会执行上面部分，`parsePath`源码可自行查看，他会将`obj.a`这种写法兼容， 最终是返回需要监听的属性的getter函数

```javascript
if (this.computed) {
    this.value = undefined;
    this.dep = new Dep();
  } else {
    // 执行get方法
    this.value = this.get();
  }
```
拿到getter后，会执行this.get方法:

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

以上为get方法，内容简单，但是做的事情举足轻重，他不仅做了值的获取，还做了依赖收集。

**pushTarget会将当前watchWatcher赋值到Dep.target中，然后执行this.getter函数，要监听的属性如count会触发他的get钩子，与此同时会进行收集依赖，收集到的依赖就是前面Dep.target也就是当前的watchWatcher**

正因为有上面的依赖收集，使count属性有了此watchWatcher的依赖，当this.count改变时，会触发set钩子，进行事件分发，从而执行回调函数

```javascript
Watcher.prototype.getAndInvoke = function getAndInvoke (cb) {
  var value = this.get();
  if (
    value !== this.value ||
    isObject(value) ||
    this.deep
  ) {
    // set new value
    var oldValue = this.value;
    this.value = value;
    this.dirty = false;
    if (this.user) {
      try {
        cb.call(this.vm, value, oldValue);
      } catch (e) {
        handleError(e, this.vm, ("callback for watcher \"" + (this.expression) + "\""));
      }
    } else {
      cb.call(this.vm, value, oldValue);
    }
  }
};

```
上面就是this.count改变时，最终调用的方法，在这里会执行this.cb，也就是定义的watch的回调函数，会把value/oldValue传递过去

### 立即执行的watch

前面说到，watch只会在监听的属性改变值后才会触发回调，在初始化时不会执行回调，如果想要一开始初始化就执行回调，需要传参对象，并immediate为true，实现原理已经在[创建Watcher](#创建Watcher)贴出来了

```javascript
if (options.immediate) {
      cb.call(vm, watcher.value);
}
```
创建watcher时，如果immediate为真值，会直接执行回调函数

## 与computed比较

computed是计算属性，watch是监听属性变化，有些场景计算属性做的事情，watch也可以做，当然要尽量用computed去做，为什么？

```javascript
new Vue({
    data: {
        num: 1,
        sum: 2
    },
    watch: {
        num(val) {
            this.sum = val +  1;
        }
    }
})
```
watch实现需要声明2个data属性num 和 sum，2个都会加入数据驱动，当num改变后，num和sum都触发了set钩子。
而computed不会，computed只会触发num的set钩子，因为sum根本没有声明，num改变后是动态计算出来的。
 