# vue虚拟dom渲染

vue的view render是虚拟dom到真实dom的过程，虚拟dom的存在是为了渲染前的一系列算法，通过算法计算出最小dom更新机制，从而降低dom更新带来的消耗，文本将介绍vue如何用虚拟dom渲染成真实dom的。

+ [view更新原理](#view更新原理)
+ [vue更新监听](#vue更新监听)
+ [更新view](#更新view)
    + [patch函数](#patch函数)
    + [patchVnode函数](#patchvnode函数)
    + [updateChildren函数](#updatechildren函数)
+ [结尾](#结尾)

## view更新原理

vue在数据更新后会自动触发view的render工作，其依赖于数据驱动；在数据驱动的工作下，每一个vue的data属性都被监听，并且在set触发时，派发事件，通知收集到的依赖，从而触发对应的操作，render工作就是其中的一个依赖，并且被每一个data属性所收集，因此每一个data属性改变后，都会触发render。

## vue更新监听

看一段代码
```javascript
// 来自mountComponent函数
updateComponent = function () {
  vm._update(vm._render(), hydrating);
};

new Watcher(vm, updateComponent, noop, {
  before: function before () {
    if (vm._isMounted) {
      callHook(vm, 'beforeUpdate');
    }
  }
}, true /* isRenderWatcher */);

```
updateComponent是更新组件的函数，内部调用vm._update，并且传参vm._render()；

* vm._render()返回了什么？看源码则得知返回了虚拟dom(VNode)
* vm._update函数又做了什么事情？顾名思义，更新传入的vnode
* 什么时候触发updateComponent函数？在任何vue的data属性更改值都会触发

## 更新view

阅读_update函数得知，最终调用了`vm.__patch__`方法，最终找到为createPatchFunction方法的返回值
```javascript

var patch = createPatchFunction({ nodeOps: nodeOps, modules: modules });

Vue.prototype.__patch__ = inBrowser ? patch : noop;

```
接下来重点看createPatchFunction的返回函数patch.

### patch函数
patch函数中处理了vnode的对比，处理了各种情况，比如第一次进行渲染的情况，注销的情况、更新vnode的情况等。具体步骤如下：

1. 如果新的vnode不存在，则注销旧的vnode
```javascript
if (isUndef(vnode)) {
  if (isDef(oldVnode)) { invokeDestroyHook(oldVnode); }
  return
}
```
2. 如果旧的vnode不存在，则创建新的vnode
```javascript
if (isUndef(oldVnode)) {
  // empty mount (likely as component), create new root element
  isInitialPatch = true;
  createElm(vnode, insertedVnodeQueue);
}
```
3. 如果以上不成立，则新的vnode和oldVnode都存在.如果oldVnode不是真实的dom，则为虚拟dom节点，并且新老vnode相似，则进行diff算法

```javascript
var isRealElement = isDef(oldVnode.nodeType);
if (!isRealElement && sameVnode(oldVnode, vnode)) {
    // patch existing root node
    patchVnode(oldVnode, vnode, insertedVnodeQueue, removeOnly);
}

```

4. 如果新老vnode不同，则拿到oldVnode的父节点，辅助创建vnode的新节点

```javascript
var oldElm = oldVnode.elm;
var parentElm = nodeOps.parentNode(oldElm);

// create new node
createElm(
  vnode,
  insertedVnodeQueue,
  // extremely rare edge case: do not insert if old element is in a
  // leaving transition. Only happens when combining transition +
  // keep-alive + HOCs. (#4590)
  oldElm._leaveCb ? null : parentElm,
  nodeOps.nextSibling(oldElm)
);
```
以上的步骤发现，更新view时，重点进入到了patchVnode函数，因此下面进入patchVnode的函数阅读

### patchVnode函数
patchVnode函数主要工作就是将2个vnode，一新一旧做不同情况的处理。比如相同的情况、类型改变的情况、类型不变内容改变的情况等。详细步骤如下：

1. 如果新老node相等，则不做处理

```javascript
if (oldVnode === vnode) {
  return
}
```
2. 如果vnode不是文本节点则有几种情况

```javascript
if (isDef(oldCh) && isDef(ch)) {
  // 如果oldVnode和vnode的children都有值（组件层），并且不想等，则执行更新children流程
  if (oldCh !== ch) { updateChildren(elm, oldCh, ch, insertedVnodeQueue, removeOnly); }
} else if (isDef(ch)) {
  // 如果vnode的children有值，如果当前dom有文本则清空，
  // 并将oldVnode的dom作为父节点，创建新vnode的children节点
  if (isDef(oldVnode.text)) { nodeOps.setTextContent(elm, ''); }
  addVnodes(elm, null, ch, 0, ch.length - 1, insertedVnodeQueue);
} else if (isDef(oldCh)) {
  // 如果oldVnode存在children，但是新的没有，则删除oldVnode的children的vnode
  removeVnodes(elm, oldCh, 0, oldCh.length - 1);
} else if (isDef(oldVnode.text)) {
  // 如果oldVnode有文本信息，则将dom的文本清空
  nodeOps.setTextContent(elm, '');
}
```

3. 如果vnode是文本节点, 则当新老节点文本不同，将dom的文本更新成新vnode的文本

```javascript
else if (oldVnode.text !== vnode.text) {
  nodeOps.setTextContent(elm, vnode.text);
}
```

patchVnode函数处理的情况梳理一下则为：

1. 如果新老vnode相同，不作处理
2. 如果新vnode是文本节点，并且新老文本不同，将dom更新为vnode的文本
3. 如果新老vnode都有children，表示他们是组件层，则调用updateChildren去做组件层更新
4. 如果新vnode是组件层，oldVnode不是，则将当前dom添加新vnode组件的子元素
5. 如果oldVnode是组件层，新vnode不是，则操作dom，将oldVnode包含的子元素删除
6. 如果新vnode是组件层，oldVnode是文本节点，则将dom的文本清空

**在patchVnode部分又浮现了一个新的函数：updateChildren，是在新老vnode都不是文本节点时调用的，这里就是vue的渲染机制的核心**

### updateChildren函数
updateChildren函数其实算是最后的也是最难的步骤了，在这里进行了一系列的计算，对不同情况做特殊处理，是核心。代码复杂切多，就不贴代码了。

updateChildren中将新老vnode的children进行的循环处理，每一次循环去判断是否有相同的vnode。

如果没有则查找当前新vnode的子节点的key是否存在oldVnode的children中，如果不存在或存在但已经不相同则创建新的dom。

否则，如果是新老节点相同，回调patchVnode函数去处理2个节点。
这样进行了递归处理，组件层的更新就完成了。

## 结尾

* 本文只针对vue的渲染部分做解析，对语法树vnode并不做解析，可自行阅读。
* 了解vue的更新机制，需要详细了解数据驱动机制，针对vue数据驱动部分（data -> view）在其他文章有详细讲解。