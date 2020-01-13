# 理解vue数据驱动

vue是双向数据绑定的框架，数据驱动是他的灵魂，他的实现原理众所周知是Object.defineProperty方法实现的get、set重写，但是这样说太牵强外门了。本文将宏观介绍他的实现

+ [使用vue](#使用vue)
+ [分析Object.defineProperty](#分析ObjectdefineProperty)
+ [简单的源码解析](#简单的源码解析)
    + [一切从头开始](#一切从头开始)
    + [数据驱动部分-观察者](#数据驱动部分-观察者)
    + [vue挂载到dom](#vue挂载到dom)
    + [简述Watcher](#简述watcher)
    + [从宏观角度看问题](#从宏观角度看问题)
+ [通过案例进行分析](#通过案例进行分析)
    + [vue数据驱动的前提](#vue数据驱动的前提)
    + [看到的未必真实的](#看到的未必真实的)
    + [看到的未必真实2](#看到的未必真实2)
+ [注意事项](#注意事项)
+ [附加讨论](#附加讨论)

## 使用vue
举个非常简单的栗子
```bash
# html
<div id="#app">
  {{msg}}
</div>

# script
<script>
new Vue({
  el: '#app',
  data: {
    msg: 'hello'
  },
  mounted() {
    setTimeout(() => {
      this.msg = 'hi'
    }, 1000);
  }
})
</script>
```

上面代码， new Vue进行创建vue对象， el属性是挂载的dom选择器，这里选择id为app的dom，data对象保存这所有数据响应的属性，当其中的某一属性值改变，就触发view渲染，从而实现了“数据->视图”的动态响应；

示例中msg初始值为hello，因此页面渲染时为hello，一秒之后，msg变为了hi，触发了view渲染，我们看到hello变为了li。那么接下来就从这简单的栗子来讲解vue的数据驱动把。

## 分析Object.defineProperty

我们说vue是怎么实现双向数据绑定的？是Object.defineProperty实现了，那么我们就直接聚焦`Object.defineProperty`

以下是代码
```javascript
function defineReactive (
  obj,
  key,
  val,
  customSetter,
  shallow
) {
  // 创建派发器
  var dep = new Dep();

  var property = Object.getOwnPropertyDescriptor(obj, key);
  if (property && property.configurable === false) {
    return
  }

  // cater for pre-defined getter/setters
  var getter = property && property.get;
  var setter = property && property.set;
  if ((!getter || setter) && arguments.length === 2) {
    val = obj[key];
  }

  var childOb = !shallow && observe(val);
  Object.defineProperty(obj, key, {
    enumerable: true,
    configurable: true,
    get: function reactiveGetter () {
      var value = getter ? getter.call(obj) : val;
      // 收集依赖对象
      if (Dep.target) {
        dep.depend();
        if (childOb) {
          childOb.dep.depend();
          if (Array.isArray(value)) {
            dependArray(value);
          }
        }
      }
      return value
    },
    set: function reactiveSetter (newVal) {
      var value = getter ? getter.call(obj) : val;
      /* eslint-disable no-self-compare */
      if (newVal === value || (newVal !== newVal && value !== value)) {
        return
      }
      /* eslint-enable no-self-compare */
      if ("development" !== 'production' && customSetter) {
        customSetter();
      }
      if (setter) {
        setter.call(obj, newVal);
      } else {
        val = newVal;
      }
      childOb = !shallow && observe(newVal);
      dep.notify();
    }
  });
}

```
vue在给每一个data的属性执行defineReactive函数，来达到数据绑定的目的。从代码中可以看到几点：

1. 每一个数据绑定，都会new一个Dep（暂且叫他派发器），派发器的功能是什么？依赖收集以事件分发；
2. 在属性get中，除了获取当前属性的值，还做了`dep.depend()`操作；
3. dep.depend的目的是什么？看Dep部分代码，很简单，其实就是依赖收集，将Dep.target需要收集的依赖进行添加到自己的派发器里
4. 在属性set时，就是给属性改变值时，除了改变值意外，还执行了`dep.notify()`操作；
5. dep.notify的目的又是什么？看代码，依旧很简单，将自己派发器的所有依赖触发update函数；

这一部分很容易了解，在data的属性get时，触发了派发器的依赖收集(dep.depend)，在data的属性set时，触发了派发器的事件通知(dep.notify)；

结合已知知识，Vue的数据绑定是上面这个函数带来的副作用，因此可以得出结论：

1.  当我们改变某个属性值时，派发器Dep通知了view层去更新
2.  Dep.target是派发器Dep收集的依赖，并在属性值改变时触发了update函数，view层的更新与Dep.target有必然的联系。换句话说：数据->视图的数据驱动就等于Dep.target.update()

## 简单的源码解析
上一节已经确定，当更改属性值时，是Dep.target.update更新了view，因此带着这个目的，此小节做一个简单的源码解析

### 一切从头开始

```javascript
function Vue (options) {
  this._init(options);
}

Vue.prototype._init = function (options) {
  var vm = this;
  callHook(vm, 'beforeCreate');
  initState(vm);
  callHook(vm, 'created');

  if (vm.$options.el) {
    vm.$mount(vm.$options.el);
  }
};

function initState (vm) {
  vm._watchers = [];
  var opts = vm.$options;
  if (opts.data) {
    initData(vm);
  } else {
    observe(vm._data = {}, true /* asRootData */);
  }
}

function initData (vm) {
  var data = vm.$options.data;
  observe(data, true /* asRootData */);
}

function observe (value, asRootData) {
  if (!isObject(value) || value instanceof VNode) {
    return
  }
  var ob = new Observer(value);;
  return ob
}

```

从头开始，一步一步进入，发现最终我们new Vue传进来的data进入了new Observer中；

### 数据驱动部分-观察者

```javascript
var Observer = function Observer (value) {
  this.value = value;
  this.dep = new Dep();
  this.vmCount = 0;
  def(value, '__ob__', this);
  if (Array.isArray(value)) {
    var augment = hasProto
      ? protoAugment
      : copyAugment;
    augment(value, arrayMethods, arrayKeys);
    this.observeArray(value);
  } else {
    this.walk(value);
  }
};
Observer.prototype.walk = function walk (obj) {
  var keys = Object.keys(obj);
  for (var i = 0; i < keys.length; i++) {
    defineReactive(obj, keys[i]);
  }
};

```
Observer构造函数中，最终执行了defineReactive为每一个属性进行定义，并且是递归调用，以树型遍历我们传入的data对象的所有节点属性，每一个节点都会被包装为一个观察者，当数据get时，进行依赖收集，当数据set时，事件分发。

**看到这里，感觉好像少了点什么，好像data到这里就结束了，但是并没有看懂为什么数据改变更新视图的，那么继续往下看**

### vue挂载到dom

回看[一切从头开始](#一切从头开始)的_init方法，在这个方法中，最后调用了` vm.$mount(vm.$options.el)`，这是把vm挂载到真实dom，并渲染view的地方，因此接着看下去。

```javascript
Vue.prototype.$mount = function (
  el,
  hydrating
) {
  return mountComponent(this, el, hydrating)
};
// 渲染dom的真实函数
function mountComponent (
  vm,
  el,
  hydrating
) {
  vm.$el = el;
  callHook(vm, 'beforeMount');

  var updateComponent;
  updateComponent = function () {
    vm._update(vm._render(), hydrating);
  };
  // new 一个Watcher，开启了数据驱动之旅
  new Watcher(vm, updateComponent, noop, {
    before: function before () {
      if (vm._isMounted) {
        callHook(vm, 'beforeUpdate');
      }
    }
  }, true /* isRenderWatcher */);
  hydrating = false;
  if (vm.$vnode == null) {
    vm._isMounted = true;
    callHook(vm, 'mounted');
  }
  return vm
}

```
上面部分看到的是，vue将vue对象挂载到真实dom的经历，最终执行了new Watcher，并且回调为`vm._update(vm._render(), hydrating)`。顾名思义，这里是执行了vue的更新view的操作（本文暂且不讲更新view，在其他文章已经讲过。本文专注数据驱动部分）。

问：为什么说new Watcher开启了数据驱动之旅呢？Watcher又是什么功能？

### 简述Watcher

如果说Object.defineProperty是vue数据驱动的灵魂，那么Watcher则是他的骨骼。

```javascript
// 超级简单的Watcher
var Watcher = function Watcher (
  vm,
  expOrFn,
  cb,
  options,
  isRenderWatcher
) {
  this.cb = cb;
  this.deps = [];
  this.newDeps = [];
  // 计算属性走if
  if (this.computed) {
    this.value = undefined;
    this.dep = new Dep();
  } else {
    this.value = this.get();
  }
};

Watcher.prototype.get = function get () {
  pushTarget(this);
  var value;
  var vm = this.vm;
  try {
    value = this.getter.call(vm, vm);
  } catch (e) {
    if (this.user) {
      handleError(e, vm, ("getter for watcher \"" + (this.expression) + "\""));
    } else {
      throw e
    }
  } finally {
    popTarget();
    this.cleanupDeps();
  }
  return value
};
```
简化后Watcher在new时，最终会调用自己的get方法，get方法中第一个语句`pushTarget(this)`是开启数据驱动的第一把钥匙，看下文
```javascript
function pushTarget (_target) {
  if (Dep.target) { targetStack.push(Dep.target); }
  Dep.target = _target;
}
```
pushTarget将传入的Watcher对象赋值给了Dep.target，还记得在讲Object.defineProperty时提到了，Dep.target.update是更新view的触发点，在这里终于找到了！

下面看Dep.targe.update
```javascript
Watcher.prototype.update = function update () {
  var this$1 = this;

  /* istanbul ignore else */
  if (this.computed) {
    if (this.dep.subs.length === 0) {
      this.dirty = true;
    } else {
      this.getAndInvoke(function () {
        this$1.dep.notify();
      });
    }
  } else if (this.sync) {
    this.run();
  } else {
  // update执行了这里
    queueWatcher(this);
  }
};
```
我们看到update方法最后执行了queueWatcher，继续看下去发现，这其实是一个更新队列，vue对同一个微任务的所有update进行了收集更新，最终执行了watcher.run，run方法又执行了`getAndInvoke`方法，getAndInvoke又执行了`this.get`方法。

到来一大圈，终于找到：在改变属性值时，触发了Dep.target所对应的Watcher的  `this.get`方法，this.get方法其实就是传入进来的回调函数。回想前面介绍的，vue在挂载到真实dom时，new Watcher传入的回调是updateComponent。串联起来得到了结论：

1. 我们在get属性时，Dep派发器收集到了Watcher当作依赖
2. 当我们set属性时，Dep派发器事件分发，使所有收集到的依赖执行`this.get`，这时候view会更新。

到这里，有没有明白为什么所有属性的派发器都会收集updateComponent的Watcher，从而在自己set时通知更新？如果没明白，那就看下一节分析

### 从宏观角度看问题

1. 当我们new Vue时，首先会将传入的data将被vue包装为观察者，所有get和set行为都会捕捉到并执行响应的操作
2. 接下来vue会将vue对象挂载到真实dom（其实指的虚拟的渲染），这个时候new 一个Watcher， 在new Watcher时，会执行一次`this.get`初始化一次值，对标updateComponent函数，这个时候会触发vue渲染过程
3. 在vue渲染过程中，所有的数据都需要进行get行为才能得到值并给真实dom赋值，因此这时触发了所有data属性的get，并且此时Dep.target是updateComponent的Watcher，因此所有的data属性派发器都收集到了此Watcher，在set时，派发器notify进行事件分发，收集到的依赖Watcher都得到了通知进行update，所有又会执行updateCompoent进行更新view



## 通过案例进行分析

### vue数据驱动的前提

vue数据驱动是有前提条件的，不是怎么用都可以的，前提条件就是必须在data中声明的属性才会参与数据驱动，数据->视图。看下面栗子

有如下html：
```html
<div id="app">
    <div>{{prev}}{{next}}</div>
</div>
```
如下js：
```javascript
new Vue({
  el: "#app",
  data: {
    prev: 'hello',
  },
  created() {
  },
  mounted() {
    this.next = 'world';
  }
})
```
页面渲染的结果是什呢？

答：hello；

为什么this.next明明赋值，没有渲染到view中去？因为他并没有参与数据驱动的观察者，还记得前面讲到vue会把传入的data对象深度遍历包装为观察者来吧，这里next属性并没有被成为观察者，因此并不会引发view更新。

### 看到的未必真实的

为什么看到的未必真实的，上面的栗子我们发现，view中看到的只有hello，但是数据真的是有hello么？未必，看下面栗子。

```javascript
new Vue({
  el: "#app",
  data: {
    prev: 'hello',
  },
  created() {
  },
  mounted() {
    this.next = 'world';
    setTimeout(() => {
      this.prev = 'hi';
      
    }, 1000);
  }
})
```
这个代码比上面栗子就多了3行代码，再页面渲染1秒后，改变prev的值为hi，那么页面会展现什么效果呢？

答：hi world

从这里可以看到，虽然next赋值并没有引起view更新，但是data确实成功变更了，当prev改变时，触发了update，从而将view更新，此时next有值，因此就显示在了view中。这就是很多初学者会遇到为什么明明赋值没有显示，但是点了一下其他的东西，却显示了的问题。

### 看到的未必真实2

还是根据第一个栗子引申一个案例如下：

```javascript
new Vue({
  el: "#app",
  data: {
    prev: 'hello',
  },
  created() {
    this.next = 'world';
  },
  mounted() {
    setTimeout(() => {
      this.next = 'memory'
    }, 1000)
  }
})
```
我们在created生命周期中赋值next，在mounted生命周期延迟一秒改变next的值，那结果会这样？

答：永远显示`helloworld`

如果已经掌握了vue实例化过程的同学可能已经猜到了为什么

当created生命周期执行时，此时还没有做vnode转化为真实dom的操作，此时data属性已经代理到this下，因此修改this.next就修改了data对象的值，data就变为了`{prev: 'hello', next: 'world'}`,因此在render时就将next也渲染到了页面上

另外此时已经完成了数据驱动的灵魂步骤（将data遍历包装为观察者），因此在延迟1s后改变next值，仍然跟栗子2一样不会引起view更新的。

**因此，写vue出现以上改变data时view未更新，首先要检查自己的代码，而不是怀疑vue框架的问题。。**

## 注意事项

* 本文参考vue版本v2.5.17-beta.0
* 本文专注数据驱动主线data，computed等支线并没有介绍，因此贴的代码都做了大量删减
* data属性收集到的依赖Dep.target并不止updateComponent的Watcher，还可能多个，比如computed属性

## 附加讨论

有些面试官会问在异步获取数据并改变data值时，放在created还是mounted？

我感觉没什么可答的，2个都没问题，当然对于代码优化，放在created更早的发出请求，因此放在created里更合适。