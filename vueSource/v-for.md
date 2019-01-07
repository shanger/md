# v-for源码解析
*vue版本v2.5.19*

两个问题：
* v-for的实现方式
* 列表渲染时key的策略
## v-for的实现方式
源码中处理v-for逻辑的场景有很多，具体大家可以逐步去查看，我们先看看它做了哪些东西。

src/core/instance/render-helpers/render-list.js

    export function renderList (
        val: any,
        render: (
            val: any,
            keyOrIndex: string | number,
            index?: number
        ) => VNode
    ): ?Array<VNode> {
        let ret: ?Array<VNode>, i, l, keys, key
        if (Array.isArray(val) || typeof val === 'string') {
            ret = new Array(val.length)
            for (i = 0, l = val.length; i < l; i++) {
            ret[i] = render(val[i], i)
            }
        } else if (typeof val === 'number') {
            ret = new Array(val)
            for (i = 0; i < val; i++) {
            ret[i] = render(i + 1, i)
            }
        } else if (isObject(val)) {
            keys = Object.keys(val)
            ret = new Array(keys.length)
            for (i = 0, l = keys.length; i < l; i++) {
            key = keys[i]
            ret[i] = render(val[key], key, i)
            }
        }
        if (isDef(ret)) {
            (ret: any)._isVList = true
        }
        return ret
    }


v-for 支持数字、数组、obj三种形式渲染列表。大家可以从编译阶段开始看，入口在src/platforms/web/entry-runtime-with-compiler.js。因为涉及到其他的函数会有很多，带着问题读，读到不明白的先记录下来，先捋顺逻辑，然后回头再详细分析实现，整个过程大概是这样的：
generate->genElement ->(el.for)genFor->_l

## 列表渲染时key的策略

vue推荐我们使用key,在列表渲染时就地复用，但是它的复用策略是怎么样的呢？先上代码：

*sameVnode*

    function sameVnode (a, b) {
        return (
            a.key === b.key && (    // key相等
            (
                a.tag === b.tag &&  // 相同标签
                a.isComment === b.isComment && // 同为注释
                isDef(a.data) === isDef(b.data) &&  // 都有data
                sameInputType(a, b) // 若是input 则type相同
            ) || (
                isTrue(a.isAsyncPlaceholder) &&
                a.asyncFactory === b.asyncFactory &&
                isUndef(b.asyncFactory.error)
            )
            )
        )
    }

从patch过程开始分析，看看数据更新后是如何映射到dom中的

    function patch (oldVnode, vnode, hydrating, removeOnly) {
        if (isUndef(vnode)) {
            if (isDef(oldVnode)) invokeDestroyHook(oldVnode)
            return
        }

        let isInitialPatch = false
        const insertedVnodeQueue = []

        if (isUndef(oldVnode)) {
            // empty mount (likely as component), create new root element
            isInitialPatch = true
            createElm(vnode, insertedVnodeQueue)
        } else {
        const isRealElement = isDef(oldVnode.nodeType)
        if (!isRealElement && sameVnode(oldVnode, vnode)) {
            // patch existing root node
            patchVnode(oldVnode, vnode, insertedVnodeQueue, null, null, removeOnly)
        } else {
            // ...
        }
        invokeInsertHook(vnode, insertedVnodeQueue, isInitialPatch)
        return vnode.elm
    }

patch 主要做了以下几点：
1. 如果vnode不存在，且oldNode存在则出发hook销毁oldNode
2. 如果不存在oldNode则新建
3. 如果存在oldNode
    * oldNode是真实dom，且vnode和oldNode相同执行patchVonde
    * 否则先以oldNode创建个空的node并替换oldNode，新建node

patchVnode 
当vnode不是文本或者注释节点，oldVnode和vnode都有子节点且不完全一致（oldCh !== ch），执行updateChildren，这个方法也是我们此次主要分析的内容
:

*updateChildren*

    while (oldStartIdx <= oldEndIdx && newStartIdx <= newEndIdx) {
      if (isUndef(oldStartVnode)) {
        oldStartVnode = oldCh[++oldStartIdx] // Vnode has been moved left
      } else if (isUndef(oldEndVnode)) {
        oldEndVnode = oldCh[--oldEndIdx]
      } else if (sameVnode(oldStartVnode, newStartVnode)) {
        patchVnode(oldStartVnode, newStartVnode, insertedVnodeQueue, newCh, newStartIdx)
        oldStartVnode = oldCh[++oldStartIdx]
        newStartVnode = newCh[++newStartIdx]
      } else if (sameVnode(oldEndVnode, newEndVnode)) {
        patchVnode(oldEndVnode, newEndVnode, insertedVnodeQueue, newCh, newEndIdx)
        oldEndVnode = oldCh[--oldEndIdx]
        newEndVnode = newCh[--newEndIdx]
      } else if (sameVnode(oldStartVnode, newEndVnode)) { // Vnode moved right
        patchVnode(oldStartVnode, newEndVnode, insertedVnodeQueue, newCh, newEndIdx)
        canMove && nodeOps.insertBefore(parentElm, oldStartVnode.elm, nodeOps.nextSibling(oldEndVnode.elm))
        oldStartVnode = oldCh[++oldStartIdx]
        newEndVnode = newCh[--newEndIdx]
      } else if (sameVnode(oldEndVnode, newStartVnode)) { // Vnode moved left
        patchVnode(oldEndVnode, newStartVnode, insertedVnodeQueue, newCh, newStartIdx)
        canMove && nodeOps.insertBefore(parentElm, oldEndVnode.elm, oldStartVnode.elm)
        oldEndVnode = oldCh[--oldEndIdx]
        newStartVnode = newCh[++newStartIdx]
      } else {
        if (isUndef(oldKeyToIdx)) oldKeyToIdx = createKeyToOldIdx(oldCh, oldStartIdx, oldEndIdx)
        idxInOld = isDef(newStartVnode.key)
          ? oldKeyToIdx[newStartVnode.key]
          : findIdxInOld(newStartVnode, oldCh, oldStartIdx, oldEndIdx)
        if (isUndef(idxInOld)) { // New element
          createElm(newStartVnode, insertedVnodeQueue, parentElm, oldStartVnode.elm, false, newCh, newStartIdx)
        } else {
          vnodeToMove = oldCh[idxInOld]
          if (sameVnode(vnodeToMove, newStartVnode)) {
            patchVnode(vnodeToMove, newStartVnode, insertedVnodeQueue, newCh, newStartIdx)
            oldCh[idxInOld] = undefined
            canMove && nodeOps.insertBefore(parentElm, vnodeToMove.elm, oldStartVnode.elm)
          } else {
            // same key but different element. treat as new element
            createElm(newStartVnode, insertedVnodeQueue, parentElm, oldStartVnode.elm, false, newCh, newStartIdx)
          }
        }
        newStartVnode = newCh[++newStartIdx]
      }
    }


处理逻辑：
* 如果oldStartVnode和newStartVnode相同，调用patchVnode，然后oldStartVnode和newStartVnode指向下一个节点
* 如果oldEndVnode和newEndVnode相同，调用patchVnode，然后oldStartVnode和newStartVnode指向前一个一个节点
* 如果oldStartVnode和newEndVnode相同，调用patchVnode进行patch，如果canMove，那么可以把oldStartVnode.elm移动到oldEndVnode.elm之后，然后把oldStartVnode设置为下一个节点，newEndVnode设置为上一个节点
* 如果oldEndVnode和newStartVnode相同，调用patchVnode进行patch，如果canMove，那么可以把oldEndVnode.elm移动到oldStartVnode.elm之前，然后把newStartVnode设置为下一个节点，oldEndVnode设置为上一个节点
* 如果以上都不匹配，则在oldChildren中寻找和newStartVnode相同key的节点，找不到则新建

* 如果找到了vnodeToMove，检查这两个节点是否相同，如果相同，调用patchVnode，如果canMove把vnodeToMove插入到oldStartVnode之前，如果key相同但不是相同节点，按新节点对待。把newStartVnode设置为下一个节点，重复以上
以上就是diff的具体操作，这样做有什么好处呢？先说结论：没有设置key的话会重新创建节点，就没有复用了。

