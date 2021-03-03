
# 深度解析v-model的原理

说到vue，最容易想到的就是“双向数据绑定”，但是它的双向数据绑定的原理是什么，本文将会深入解析。

从官方的教程就能得出结论，官方介绍如下：

> 你可以用 v-model 指令在表单 input、textarea 及 select 元素上创建双向数据绑定。它会根据控件类型自动选取正确的方法来更新元素。尽管有些神奇，但 v-model 本质上不过是语法糖。它负责监听用户的输入事件以更新数据，并对一些极端场景进行一些特殊处理。

可以明确的得出结论，v-model只是语法糖，其实现原理还是基于事件本身。

从应用层则可以认为v-model的实现等价于如下代码：

```
# 给input绑定msg属性，input事件中改变msg的值
<input :value="msg" @input="handleChangeMsg" />

```
其实事实也是这样，后续源码分析会以事实论证。

## 目录

+ [一切从案例开始](#一切从案例开始)
+ [Vue初始化](#Vue初始化)
+ [render函数的构建](#render函数的构建)
  + [compileToFunctions函数](#compileToFunctions函数)
  + [createCompilerCreator函数](#createCompilerCreator函数)
  + [ast树生成render函数体](#ast树生成render函数体)
  + [小结](#小结)
+ [VNode生成真实DOM](#VNode生成真实DOM)
  + [patch函数重心](#patch函数重心)
  + [createElm函数生成真实Dom](#createElm函数生成真实Dom)
  + [invokeCreateHooks函数](#invokeCreateHooks函数)
  + [cbs的真相](#cbs的真相)
  + [events事件注册](#events事件注册)
+ [总结v-model的实现原理](#总结v-model的实现原理)
+ [结尾](#结尾)



### 一切从案例开始

本文有些地方，会以以下示例来输出案例

```
# html
<div id="app">
  <input  v-model="msg" />
</div>

# js

new Vue({
  el: '#app',
  data: {
    msg: '11'
  }
})
```

### Vue初始化

`new Vue`首先会进入到`this._init`方法中去，这个方法中，初始化了一系列内容：

```
Vue.prototype._init = function (options) {
  // 省略了一系列其他代码
  var vm = this;
  /* istanbul ignore else */
  {
    initProxy(vm);
  }
  // expose real self
  vm._self = vm;
  initLifecycle(vm);
  initEvents(vm);
  // 初始化渲染函数
  initRender(vm);
  callHook(vm, 'beforeCreate');
  initInjections(vm); // resolve injections before data/props
  // 数据绑定data/computed/watch等
  initState(vm);
  initProvide(vm); // resolve provide after data/props
  callHook(vm, 'created');
  if (vm.$options.el) {
    // 挂载到dom
    vm.$mount(vm.$options.el);
  }
};

```

`initRender`函数单独讲述一下：
initRender中可以看到主要是给vm实例初始化了一些方法，其中`_c`方法将会在后续源码中扮演主角;


```
function initRender (vm) {
  vm._vnode = null; // the root of the child tree
  // 省略很多其他代码
  // createElement函数就是创建VNode的功能函数
  vm._c = function (a, b, c, d) { return createElement(vm, a, b, c, d, false); };
  vm.$createElement = function (a, b, c, d) { return createElement(vm, a, b, c, d, true); };

}
```


最后执行了`vm.$mount`函数，该函数内部做了render函数赋值，如果options没有render，并且有template，则根据template创建render函数，这一过程为`render`函数构建过程，这一过程我们后面详细说；

```
Vue.prototype.$mount = function (
  el,
  hydrating
) {
  el = el && query(el);
  var options = this.$options;
  // 如果没有render函数，则用传入模板构建render
  if (!options.render) {
    var template = options.template;
    if (template) {
      if (typeof template === 'string') {
        if (template.charAt(0) === '#') {
          template = idToTemplate(template);
        }
      } else if (template.nodeType) {
        template = template.innerHTML;
      } else {
        return this
      }
    } else if (el) {
      // 传入的是el，则获取el的html内容做为template
      template = getOuterHTML(el);
    }
    if (template) {
      // 调用compileToFunctions，拿到render函数
      var ref = compileToFunctions(template, {
        shouldDecodeNewlines: shouldDecodeNewlines,
        shouldDecodeNewlinesForHref: shouldDecodeNewlinesForHref,
        delimiters: options.delimiters,
        comments: options.comments
      }, this);
      var render = ref.render;
      var staticRenderFns = ref.staticRenderFns;
      options.render = render;
      options.staticRenderFns = staticRenderFns;
    }
  }
  // 调用mount函数完成mounted周期
  return mount.call(this, el, hydrating)
};

```


最终调用了mount，其实是`mountComponent`函数，此函数也是Vue初始化的最后一步，我们熟悉的mounted声明周期就在这里触发的；

```
function mountComponent (
  vm,
  el,
  hydrating
) {
  vm.$el = el;
  // 省略很多内容
  callHook(vm, 'beforeMount');
  var updateComponent = function () {
    vm._update(vm._render(), hydrating);
  };
  // 实例一个Watcher，此Watcher为render专用，每当vue有data改变，就会通知到此Watcher从而执行updateComponent
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

可以看到，这里只时定义了一个Watcher，并且回调函数为`updateComponent`,如果你已经看过vue数据驱动部分的源码分析，你就明白，这里数据驱动必要的一部分，我称它为renderWatcher，因为他负责的是渲染dom；不明白的可以去看vue数据驱动篇；

本文重点内容很清晰，就是renderWatcher的回调`vm._update(vm._render(), hydrating);`，后续内容将会一探究竟。

### render函数的构建

我们`new Vue`时，一般不会指定render函数，因此这里需要构建出render函数来，步骤有以下几步：
1. 获取模板字符串，如果传入id，则根据id查出dom获取到outerHTML内容作为模板字符串；
2. 根据模板字符串，通过`parse`方法解析出ast树
3. 根据ast树结构，调用`generate`函数获取到render函数体
4. 通过`compiler`再包装，返回最终对象，该对象包含render函数

通过以上4步，则构建出了`render`函数；

#### compileToFunctions函数

从前面源码可以看出，在`$mount`函数中，会执行`compileToFunctions`函数得到一个ref对象，`ref.render`就是我们构建出的render函数，来看一下compileToFunctions函数做了什么事；
```
var ref$1 = createCompiler(baseOptions);
var compileToFunctions = ref$1.compileToFunctions;
```
从源码可以看出, compileToFunctions其实是一个已经实例好的函数，是通过`createCompiler(baseOptions)`的执行得到的结果，因此接下来，看createCompiler：
```
var createCompiler = createCompilerCreator(function baseCompile (
  template,
  options
) {
  // 解析成ast
  var ast = parse(template.trim(), options);
  if (options.optimize !== false) {
    optimize(ast, options);
  }
  // 根据ast，生成render函数体code.render
  var code = generate(ast, options);
  return {
    ast: ast,
    render: code.render,
    staticRenderFns: code.staticRenderFns
  }
});
```
createCompiler是一个高阶函数，是通过`createCompilerCreator`函数传入`baseCompile`构建而来的，`baseCompile`函数是ast树的构建函数，这里暂时先不讲，后续再将，接下来看`createCompilerCreator `函数

#### createCompilerCreator函数

通过源码，我们看到了，前面用到的`compileToFunctions`方法就是 createCompilerCreator高阶函数返回的函数中的返回值，compileToFunctions 的值是`createCompileToFunctionFn(compile)`的结果，`createCompileToFunctionFn`中传入的`compile`,注意了，`compile`正是关键函数，而`createCompileToFunctionFn`函数也是高阶函数，它返回的函数中，执行了`compile`并得到ast树相关信息，并且在这个函数中，根据ast真正形成了我们需要的`render`函数。

```
function createCompilerCreator (baseCompile) {
  return function createCompiler (baseOptions) {
    function compile (
      template,
      options
    ) {
      // 省略代码
      // 调用传入的baseCompile构建render函数
	var compiled = baseCompile(template, finalOptions);
  	{
    	rrors.push.apply(errors, detectErrors(compiled.ast));
  	}
  	compiled.errors = errors;
  	compiled.tips = tips;
  	return compiled
    }

    return {
      compile: compile,
      compileToFunctions: createCompileToFunctionFn(compile)
    }
  }
}

```

compile函数内是执行了外部传入的baseCompile得到的结果，当然compile函数主要功能就是一系列的计算得到finalOptions传给baseCompile；


这一部分非常绕，需要耐心阅读源码去理解清楚。在这里大致理一下：

在Vue初始化时会调用`compileToFunctions`函数获取到一个对象`ref`，`ref.render`就是需要的render函数，`ref.render`是通过`baseCompile `函数生成ast对象，然后createCompileToFunctionFn再处理，最终得到了`render`函数；

接下来重点在`baseCompile`中,ast树的生成

### ast树生成render函数体

`baseCompile`函数就是生成ast树，通过ast树来生成render函数体，是个string，然后前面说的`createCompileToFunctionFn`再处理其实就是将函数体变成函数，利用`new Function(string)`的特性；

```
// 生成ast树
var ast = parse(template.trim(), options);
```
baseCompile第一行代码就是根据`parse`函数生成ast树，`parse`函数的功能就是将template转化为ast，此函数不在本文讨论中，接着往下看

```
// 通过ast树生成render函数体
var code = generate(ast, options);
```
上面一行代码，就是生成函数体的核心代码，`generate`方法生成了函数体，往下看`generate`内容：

```
function generate (
  ast,
  options
) {
  var state = new CodegenState(options);
  var code = ast ? genElement(ast, state) : '_c("div")';
  return {
    render: ("with(this){return " + code + "}"),
    staticRenderFns: state.staticRenderFns
  }
}

```

`generate`函数通过options创建了`CodegenState`实例，这个其实是一些“辅助性的解析工具”（纯属个人看法），其用途就是辅助ast树构建函数体的。这里核心还是`genElement`函数

```
function genElement (el, state) {
  if (el.staticRoot && !el.staticProcessed) {
    return genStatic(el, state)
  } else if (el.once && !el.onceProcessed) {
    return genOnce(el, state)
  } else if (el.for && !el.forProcessed) {
    return genFor(el, state)
  } else if (el.if && !el.ifProcessed) {
    return genIf(el, state)
  } else if (el.tag === 'template' && !el.slotTarget) {
    return genChildren(el, state) || 'void 0'
  } else if (el.tag === 'slot') {
    return genSlot(el, state)
  } else {
    // component or element
    var code;
    if (el.component) {
      code = genComponent(el.component, el, state);
    } else {
      // 通过genData$2获取到全部属性信息
      var data = el.plain ? undefined : genData$2(el, state);
      var children = el.inlineTemplate ? null : genChildren(el, state, true);
      code = "_c('" + (el.tag) + "'" + (data ? ("," + data) : '') + (children ? ("," + children) : '') + ")";
    }
    // module transforms
    for (var i = 0; i < state.transforms.length; i++) {
      code = state.transforms[i](el, code);
    }
    return code
  }
}
```

`genElement`函数主要功能就是根据不同的ast类型， 生成不同的函数体，比如`v-if`,`v-for`等ast，都有自己特定的处理函数，我们这里讨论的案例，最终会在else中生成函数体，就是看到的`code`,不难看出，`code`是一个字符串，但是字符串表示的意思是`_c`函数执行，`el.tag`，`data`,`children`依次是`_c`函数的参数，还记得前面说的主角`_c`函数吧，用处就在这里。

`genChildren`函数其实是递归处理ast树的方法，genChildren和genElement相互调用，最终递归遍历全部ast节点，最终生成了`code`,本文案例中，`code`生成打印出来如下：
```
_c('div',{attrs:{"id":"app"}},[_c('input',{directives:[{name:"model",rawName:"v-model",value:(msg),expression:"msg"}],domProps:{"value":(msg)},on:{"input":function($event){if($event.target.composing)return;msg=$event.target.value}}}),_v(" "),_c('div',[_v("11")])])
```

那问题来了，这串代码是怎么进行页面渲染的？看不出什么来，接下来看前面讲过的一部分代码

```
function generate (
  ast,
  options
) {
  var state = new CodegenState(options);
  var code = ast ? genElement(ast, state) : '_c("div")';
  return {
    render: ("with(this){return " + code + "}"),
    staticRenderFns: state.staticRenderFns
  }
}
```

`generate`函数拿到`code`后，包装了一层，`with(this){return xxx}`,这里可以看出`code`中的`_c`其实就是`this._c`，这里的`this`就是当前Vue实例`vm`，所以render函数最后返回了`this._c(...)`所生成的VNode

#### 小结

到这里，通过`this._c`生成了VNode，也就是虚拟Dom，其中v-model的input元素的VNode节点包含如下data：

```
{
  // 指令列表
  directives: [
    {
      name: "model",
      rawName: "v-model",
      value: (msg), expression: "msg"
    }],
  // dom属性和表达式
  domProps: { "value": (msg) },
  // 事件绑定和回调
  on: {
    "input": function ($event) {
      if ($event.target.composing) return; msg = $event.target.value
    }
  }
}

```

可以看出，其实v-model做的事情就是给元素增加了属性和事件,那么什么时候加的呢，其实就是在`genElement`函数中`genData$2`函数内部的`genDirectives`方法中执行的，有兴趣可以自己深入看一下源码，这里不再深入。

### VNode生成真实DOM

上面`render`函数其实是生成了虚拟Dom，真是Dom生成是在`vm._update`函数中处理的，接下来看此函数做了什么操作

```
Vue.prototype._update = function (vnode, hydrating) {
  var vm = this;
  var prevVnode = vm._vnode;
  activeInstance = vm;
  vm._vnode = vnode;
  if (!prevVnode) {
    // initial render
    vm.$el = vm.__patch__(vm.$el, vnode, hydrating, false /* removeOnly */);
  } else {
    // updates
    vm.$el = vm.__patch__(prevVnode, vnode);
  }
};

```

主要逻辑如上，_update时，判断是否有VNode，如果没有，则进行第一次渲染，如果有，则update，最终执行的都是`__patch__`函数，下面进入正题。

#### patch函数重心

patch函数是由`createPatchFunction`函数执行返回的函数，`createPatchFunction`函数本身包含了许多工具方法等内容，非常繁琐，这里不赘述，后面只讲述主线相关内容。

直接看return的`patch`方法.

```
function patch (oldVnode, vnode, hydrating, removeOnly) {
  // 如果传入新节vnode不存在，则注销vnode
  if (isUndef(vnode)) {
    if (isDef(oldVnode)) { invokeDestroyHook(oldVnode); }
    return
  }

  var isInitialPatch = false;
  var insertedVnodeQueue = [];
  // 案例中，总会执行else， if这里是在$mount不指定dom情况下才会执行；
  if (isUndef(oldVnode)) {
    isInitialPatch = true;
    createElm(vnode, insertedVnodeQueue);
  } else {
    // 判断oldVnode是真实Dom还是Vnode
    var isRealElement = isDef(oldVnode.nodeType);
    // 如果是Vnode并且新老Vnode相似，则执行patchVnode；
    // 如果新老Vnode不相似，走else，根据新Vnode创建dom
    // 如果isRealElement为true（oldVnode是真实dom），意味着初次执行patch，因此根据Vnode创建dom
    if (!isRealElement && sameVnode(oldVnode, vnode)) {
      patchVnode(oldVnode, vnode, insertedVnodeQueue, removeOnly);
    } else {
      // 如果oldVnode是真实dom， 将一个空的Vnode赋值给oldVnode，此为初始化
      if (isRealElement) {
        oldVnode = emptyNodeAt(oldVnode);
      }

      var oldElm = oldVnode.elm;
      var parentElm = nodeOps.parentNode(oldElm);
      
      // 根据Vnode创建dom
      createElm(
        vnode,
        insertedVnodeQueue,
        oldElm._leaveCb ? null : parentElm,
        nodeOps.nextSibling(oldElm)
      );
      // 省略后续父组件处理逻辑和注销老组件逻辑
    }
  }

  return vnode.elm
}

```

在patch方法中，经过一系列的判断，可以看出，当第一次渲染页面，执行了createElm，创建真实dom；在后续的渲染，都会走patchVnode方法进行更新。
其实patchVnode函数中做了对Vnode不同场景的不同更新方式.
[vue虚拟dom渲染](./vue虚拟dom渲染.md)中也简单介绍了其中的流程，这里再做介绍，直接看`createElm`方法


#### createElm函数生成真实Dom


简化后的函数如下：
```
function createElm (
  vnode,
  insertedVnodeQueue,
  parentElm,
  refElm
) {
  // 如果vnode是component，会执行createComponent并返回true，终止createElm函数；
  if (createComponent(vnode, insertedVnodeQueue, parentElm, refElm)) {
    return
  }
  var data = vnode.data;
  var children = vnode.children;
  var tag = vnode.tag;
  if (isDef(tag)) {
    // 创建真实dom
    vnode.elm = vnode.ns
      ? nodeOps.createElementNS(vnode.ns, tag)
      : nodeOps.createElement(tag, vnode);
    // style scope的处理
    setScope(vnode);
    {
      // 递归创建子节点，createChildren中也会调用createElm形成递归
      createChildren(vnode, children, insertedVnodeQueue);
      
      if (isDef(data)) {
        // create相关钩子注册， v-model指令vue默认行为的input事件就是在这里注册的
        invokeCreateHooks(vnode, insertedVnodeQueue);
      }
      insert(parentElm, vnode.elm, refElm);
    }
  } else if (isTrue(vnode.isComment)) {
    // 注释节点创建
    vnode.elm = nodeOps.createComment(vnode.text);
    insert(parentElm, vnode.elm, refElm);
  } else {
    // 文本节点创建
    vnode.elm = nodeOps.createTextNode(vnode.text);
    insert(parentElm, vnode.elm, refElm);
  }
}
```
可以看出，在`createElm`函数中，真正创建了dom并切缓存到了`vnode.elm`中去；
`createChildren`函数会将子节点递归遍历调用`createElm`从而达到生成所有层级dom的效果；`invokeCreateHooks`函数则是执行所有的create相关的钩子事件;

#### invokeCreateHooks函数

函数内容如下：

```
function invokeCreateHooks (vnode, insertedVnodeQueue) {
    for (var i$1 = 0; i$1 < cbs.create.length; ++i$1) {
      cbs.create[i$1](emptyNode, vnode);
    }
    i = vnode.data.hook; // Reuse variable
    if (isDef(i)) {
      if (isDef(i.create)) { i.create(emptyNode, vnode); }
      if (isDef(i.insert)) { insertedVnodeQueue.push(vnode); }
    }
  }

```

这个函数内容不多，因为它只是负责创建，至于创建的内容完全不在乎，只看这个函数看不到任何价值，因此首先要搞懂`cbs`是干什么用的；

#### cbs的真相

在`createPatchFunction`函数最开始，就有这一段代码：

```
var i, j;
var cbs = {};

var modules = backend.modules;
var nodeOps = backend.nodeOps;

for (i = 0; i < hooks.length; ++i) {
  cbs[hooks[i]] = [];
  for (j = 0; j < modules.length; ++j) {
    if (isDef(modules[j][hooks[i]])) {
      cbs[hooks[i]].push(modules[j][hooks[i]]);
    }
  }
}
```
可以看出在这里，生成了`cbs`，但是具体是干嘛的，还需要看`backend.modules`和`hooks`的内容，`hooks`很简单，是源码中定义的一个数组如下：

```
var hooks = ['create', 'activate', 'update', 'remove', 'destroy'];
```
`backend.modules`则是一组工具对象，包括如下：

```
var baseModules = [
  ref,
  directives
]

var platformModules = [
  attrs,
  klass,
  events,
  domProps,
  style,
  transition
]

var modules = platformModules.concat(baseModules);

var patch = createPatchFunction({ nodeOps: nodeOps, modules: modules });

```
可以去看一下每一个对象的内容，每个对象包含着`create`, `update`等包含于`hooks`数组的方法，类似于:

```
// 接口
interface BaseModule {
  create?: function;
  update?: function;
  activate?: function;
  remove?: function;
  destroy?: function;
}

// 声明ref
const ref: BaseModule = {
  create(_, vnode) {
    registerRef(vnode);
  }
  update(oldVnode, vnode) {
    updateRef(oldVnode, vnode);
  }
}
// 声明attrs
const attrs: BaseModule = {
  create(_, vnode) {
    createAttrs(vnode);
  }
}

```
以上是伪代码，只是代表一个意思，可以看出，所有的module只能包含`hooks`中的方法名，可以没有某一些方法，比如只包含`create`和`update`方法；
看到这种结构，回到本节刚开始的代码，仔细阅读可以发现，`cbs`是一个集合，是一个包含了`hooks`的几种钩子的集合，最终的结构如下：

```
cbs = {
  create: [ref.create, attrs.create...],
  update: [],
  activate: [],
  remove: [],
  destory: []
}

```
每一种钩子都包含了所有module的对应的方法；

所以，回到`invokeCreateHooks`方法中

```
cbs.create[i$1](emptyNode, vnode);
```
以上语句则是把所有module的create钩子执行一遍，将vnode传递进入，具体该如何执行，是module自己的事情了。

为了探究`v-model`指令，vue做了什么，我们去看一下`events`中的逻辑：

#### events事件注册

```
// 钩子函数执行了此函数
function updateDOMListeners (oldVnode, vnode) {
  // 如果vnode中没有on，也就是没有定义事件，则不处理
  if (isUndef(oldVnode.data.on) && isUndef(vnode.data.on)) {
    return
  }
  var on = vnode.data.on || {};
  var oldOn = oldVnode.data.on || {};
  // createElm章节说到，vnode.elm是创建的真实dom
  target$1 = vnode.elm;
  // 兼容性处理，如兼容ie的事件处理
  normalizeEvents(on);
  // 处理函数主体
  updateListeners(on, oldOn, add$1, remove$2, vnode.context);
  target$1 = undefined;
}
// events的钩子函数
var events = {
  create: updateDOMListeners,
  update: updateDOMListeners
}

```

根据上面我们看出，`updateListeners`函数就是处理函数的主体，来看一下

```
function updateListeners (
  on,
  oldOn,
  add,
  remove$$1,
  vm
) {
  var name, def, cur, old, event;
  for (name in on) {
    def = cur = on[name];
    old = oldOn[name];
    event = normalizeEvent(name);
    if (isUndef(old)) {
      if (isUndef(cur.fns)) {
        //高阶函数，返回的依旧是是事件处理函数
        cur = on[name] = createFnInvoker(cur);
      }
      // 给dom添加事件
      add(event.name, cur, event.once, event.capture, event.passive, event.params);
    } else if (cur !== old) {
      old.fns = cur;
      on[name] = old;
    }
  }
  // 清除旧的dom的事件处理函数
  for (name in oldOn) {
    if (isUndef(on[name])) {
      event = normalizeEvent(name);
      remove$$1(event.name, oldOn[name], event.capture);
    }
  }
}

```
`updateListeners`方法也很简单，就是给dom注册事件，给旧的dom删除事件，具体dom注册事件在add$1方法，具体可以自己去看；


### 总结v-model的实现原理

通过源码阅读，v-model并没有那么神奇了，根据源码大致讲述一下原理如下：

1. Vue在实例化时，会根据模板生成的ast树，递归遍历处理，在处理过程中，会将含有`v-model`指令的ast节点进行加工，根据不同的类型，执行不同的逻辑处理， `v-model`为例，最终在render函数体中增加value和input事件的描述，其中input事件会将最新输入内容赋值给绑定的属性，本案例中为`msg`；
2. Vue执行render函数获取到Vnode虚拟DOM，并在mounted阶段，执行patch方法，进入createElm函数创建真实dom；
3. 在创建真实dom过程中，会执行所有modules的create钩子处理函数，其中就有events模块的create钩子，该钩子函数中，会将vnode虚拟dom的事件描述进行真实dom绑定，也就是前面的input事件；
4. 最终，在我们改变输入框内容时，出发了input事件，input事件中，修改了我们绑定的`msg`属性，根据vue的数据驱动原理，修改了data属性，则会重新绘制页面，因此最终，`v-model`的功能完成了。


### 结尾

* vue版本v2.5.17-beta.0
* 本文在针对v-model的实现，进行了一整条链路的源码分析，帮助你阅读源码
* 本文涉及到的部分有数据驱动部分、虚拟dom渲染部分不进行过多描述，详情可以观看相关文章