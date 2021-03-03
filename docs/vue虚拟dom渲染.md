# vue虚拟dom渲染

vue的view render是虚拟dom到真实dom的过程，虚拟dom的存在是为了渲染前的一系列算法，通过算法计算出最小dom更新机制，从而降低dom更新带来的消耗，文本将介绍vue如何用虚拟dom渲染成真实dom的。

+ [view更新原理](#view更新原理)
+ [vue更新监听](#vue更新监听)
+ [更新view](#更新view)
    + [patch函数](#patch函数)
    + [patchVnode函数](#patchvnode函数)
    + [updateChildren函数](#updatechildren函数)
    + [diff算法详解](#diff算法详解)
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
updateChildren函数是最后的也是最难的步骤，在这里进行了一系列的比对，对不同情况做特殊处理，是diff算法的核心，代码复杂较大。

updateChildren中，采用双指针进行遍历，动态对比新老vnode指向的节点是否相似，如果匹配到相似的内容，则进行dom节点的移动，文案更改等操作，否则，则执行创建新的dom。

updateChildren函数采用了递归处理方法，vnode相似的情况下，继续执行patchVnode函数进行dom更改，组件层的更新就完成了。

接下来看详细的源码内容
### diff算法详解
先看代码解析，然后再进行具体分析
```javascript
function updateChildren (parentElm, oldCh, newCh, insertedVnodeQueue, removeOnly) {
  var oldStartIdx = 0; // oldCh的开始坐标
  var newStartIdx = 0; // newCh的开始坐标
  var oldEndIdx = oldCh.length - 1; // oldCh的结束坐标
  var oldStartVnode = oldCh[0];  // oldCh的开始节点
  var oldEndVnode = oldCh[oldEndIdx];  // oldCh的结束节点
  var newEndIdx = newCh.length - 1;// newCh的结束坐标
  var newStartVnode = newCh[0];   // newCh的开始节点
  var newEndVnode = newCh[newEndIdx];  // newCh的结束节点
  var oldKeyToIdx, idxInOld, vnodeToMove, refElm;
  // while循环的条件： 左指针<=右指针， 也就是oldCh和newCh的开始下标<=结束下标
  while (oldStartIdx <= oldEndIdx && newStartIdx <= newEndIdx) {
    // 匹配过的节点，oldCh中会删除，因此有没有值的情况
    if (isUndef(oldStartVnode)) {
      oldStartVnode = oldCh[++oldStartIdx];
    } else if (isUndef(oldEndVnode)) {
      oldEndVnode = oldCh[--oldEndIdx];
    } else if (sameVnode(oldStartVnode, newStartVnode)) {
      // 左侧的vnode相似，执行patchVnode更新dom，右指针同时右移
      patchVnode(oldStartVnode, newStartVnode, insertedVnodeQueue);
      oldStartVnode = oldCh[++oldStartIdx];
      newStartVnode = newCh[++newStartIdx];
    } else if (sameVnode(oldEndVnode, newEndVnode)) {
      // 右侧的vnode相似，执行patchVnode更新dom，右指针同时左移
      patchVnode(oldEndVnode, newEndVnode, insertedVnodeQueue);
      oldEndVnode = oldCh[--oldEndIdx];
      newEndVnode = newCh[--newEndIdx];
    } else if (sameVnode(oldStartVnode, newEndVnode)) {
      // oldCh的左节点和newCh的右节点相似，则证明新的children中往右移动，dom位置更换
      patchVnode(oldStartVnode, newEndVnode, insertedVnodeQueue);
      canMove && nodeOps.insertBefore(parentElm, oldStartVnode.elm, nodeOps.nextSibling(oldEndVnode.elm));
      oldStartVnode = oldCh[++oldStartIdx];
      newEndVnode = newCh[--newEndIdx];
    } else if (sameVnode(oldEndVnode, newStartVnode)) {
      // oldCh的右节点和newCh的左节点相似，证明新的children中，往左移动，dom位置更换
      patchVnode(oldEndVnode, newStartVnode, insertedVnodeQueue);
      canMove && nodeOps.insertBefore(parentElm, oldEndVnode.elm, oldStartVnode.elm);
      oldEndVnode = oldCh[--oldEndIdx];
      newStartVnode = newCh[++newStartIdx];
    } else {
      // 处理当前循环没有命中的情况， 如果newCh的左节点key值存在oldCh数组中，通过index找到oldCh的这个节点
      // 2者对比，如果节点相同，按节点移动并更新操作处理，如果不同则新建dom
      // 如果newCh的左节点key值不存在oldCh数组中，直接新建dom
      if (isUndef(oldKeyToIdx)) { oldKeyToIdx = createKeyToOldIdx(oldCh, oldStartIdx, oldEndIdx); }
      idxInOld = isDef(newStartVnode.key)
        ? oldKeyToIdx[newStartVnode.key]
        : findIdxInOld(newStartVnode, oldCh, oldStartIdx, oldEndIdx);
        if (isUndef(idxInOld)) { // New element
        createElm(newStartVnode, insertedVnodeQueue, parentElm, oldStartVnode.elm, false, newCh, newStartIdx);
      } else {
        vnodeToMove = oldCh[idxInOld];
        if (sameVnode(vnodeToMove, newStartVnode)) {
          patchVnode(vnodeToMove, newStartVnode, insertedVnodeQueue);
          oldCh[idxInOld] = undefined;
          canMove && nodeOps.insertBefore(parentElm, vnodeToMove.elm, oldStartVnode.elm);
        } else {
          // same key but different element. treat as new element
          createElm(newStartVnode, insertedVnodeQueue, parentElm, oldStartVnode.elm, false, newCh, newStartIdx);
        }
      }
      // 不管结果如何，左节点都会被新建或更新，因此左指针右移处理下一个新节点
      newStartVnode = newCh[++newStartIdx];
    }
  }
  /**
   * 处理剩余的情况：
   * 1.如果oldStartIdx > oldEndIdx，证明，oldCh中的节点全部存在newCh中更新完成并退出循环，此时，while结束的原因就是：oldStartIdx > oldEndIdx
   * 这种情况下，将newStartIdx到newEndIdx的节点创建dom添加到页面上
   * 2.如果newStartIdx > newEndIdx，证明，oldCh中的节点没有全部更新完毕，此时while结束的原因是newStartIdx > newEndIdx，
   * 这种情况下，oldCh中节点没参与更新的原因就是newCh不存在这些节点，因此需要删除这些dom
   * 
   */
  if (oldStartIdx > oldEndIdx) {
    refElm = isUndef(newCh[newEndIdx + 1]) ? null : newCh[newEndIdx + 1].elm;
    addVnodes(parentElm, refElm, newCh, newStartIdx, newEndIdx, insertedVnodeQueue);
  } else if (newStartIdx > newEndIdx) {
    removeVnodes(parentElm, oldCh, oldStartIdx, oldEndIdx);
  }
}

```

分析：vue的diff算法使用了双向指针的操作进行循环，每一次循环，有4种可能命中相同dom进行移动或更新，这样的好处就是减少了循环次数，降低时间复杂度。其中，vue使用了`sameVnode`方法去判断2个vnode是否相似，来看一下这个函数：

```
function sameVnode (a, b) {
  return (
    a.key === b.key && (
      (
        a.tag === b.tag &&
        a.isComment === b.isComment &&
        isDef(a.data) === isDef(b.data) &&
        sameInputType(a, b)
      ) || (
        isTrue(a.isAsyncPlaceholder) &&
        a.asyncFactory === b.asyncFactory &&
        isUndef(b.asyncFactory.error)
      )
    )
  )
}
```
相同节点的判断逻辑很简单：
1. `key`相同
2. `tag`相同，也就是标签，例如div、span
3. 都是vue组件或者都是文本，文本没有data属性
4. 如果是input组件，type相同

总体思路分析：

在处理数组更新时，vue使用双向指针进行4种情况的vnode匹配，如果某一种情况下新老vnode相同，则进行更新dom操作，如果新老vnode的顺序不同，则移动dom节点。
在一次循环中，如果没有发现某种情况vnode相同，则以新的vnode左指针为更新对象去处理，首先会查询key值是否存在老的vnode列表中，如果存在，则进行移动更新，并且将老的vnode设置为null(目的是只命中一次)，如果不存在，创建新的dom插入页面。
在循环完成后，根据diff算法的特点，会有2种不同的情况，根据指针的判断处理这2种情况下的dom操作。

如果不理解上面说的2种情况，可以举例说明： 

1. list新增数据

```
# 初始数据：
list: [1,2,3];
# 用户操作：
this.list.push(4)

# list改变后的数据
list: [1,2,3,4]
```

diff算法如下：

| 步骤 | oldStartIdx | oldEndIdx | newStartIdx | newEndIdx | 结果 |
|  ----  | ----  | ----|  ----  | ----  | ---- |
| 1 | 0 | 2 | 0 | 3 | sameVnode(oldStartVnode, newStartVnode) === true |
| 2 | 1 | 2 | 1 | 3 | sameVnode(oldStartVnode, newStartVnode) === true |
| 3 | 2 | 2 | 2 | 3 | sameVnode(oldStartVnode, newStartVnode) === true |
| 4 | 3 | 2 | 3 | 3 | oldStartIdx > oldEndIdx退出循环 |

通过3次循环对比，vue已经将1，2，3完成了对比，并且vnode相同，因此dmo不更新，退出循环时，
oldStartIdx = 3， oldEndIdx = 2，命中上面说的2种情况 的第一种，因此执行`addVnodes`,`addVnodes`中将会把newStartIdx后的dom进行dom创建，在例子中是[4]，所以完成了整个更新过程。

2. list删除数据
```
# 初始数据
list: [1,2,3]

# 用户操作
this.list = [1,3]

#list改变后的数据
list  = [1,3]
```
diff算法如下：

| 步骤 | oldStartIdx | oldEndIdx | newStartIdx | newEndIdx | 结果 |
|  ----  | ----  | ----|  ----  | ----  | ---- |
| 1 | 0 | 2 | 0 | 1 | sameVnode(oldStartVnode, newStartVnode) === true |
| 2 | 1 | 2 | 1 | 1 | sameVnode(oldEndVnode, newEndVnode) === true |
| 4 | 1 | 1 | 1 | 0 | newStartIdx > newEndIdx退出循环 |

通过2次循环对比，vue已经将1，3完成了对比，并且vnode相同，因此dmo不更新，退出循环时，
newStartIdx = 1，newEndIndex = 0， 命中上面说的2种情况的第二种，因此执行`removeVnodes`进行dom删除，删除范围是[1,1]，也就是例子中的[2]

## 结尾

* 本文只针对vue的渲染部分做解析，对语法树vnode并不做解析，可自行阅读。
* 了解vue的更新机制，需要详细了解数据驱动机制，针对vue数据驱动部分（data -> view）在其他文章有详细讲解。