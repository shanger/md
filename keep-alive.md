# vue keep-alive
### 使用方法
#### 介绍
<keep-alive> 包裹动态组件时，会缓存不活动的组件实例，而不是销毁它们。和 <transition> 相似，<keep-alive> 是一个抽象组件：它自身不会渲染一个 DOM 元素，也不会出现在父组件链中。
keep-alive 组件状态切换时，会触发activated和deactivated钩子
ps: keep-alive 只允许同时渲染一个子元素
#### 使用
* 基本用法

![基本用法](http://94.191.25.51/uploads/images/9924a0b20e8da.png)
* keep-alive 和 vue-router

 ![基本用法](http://94.191.25.51/uploads/images/8acc885ce67f8.png)

### 使用场景
* 保留组件状态
* 避免重新渲染提升性能
### 常见问题
上面有提到，keep-alive组件切换状态只会触发activated和deactivated钩子，如果存在时效性的处理需要在activated中处理 

### 源码分析
vue: v2.5.19
src/core/components/keep-alive.js
```
export default {
    name: 'keep-alive',
    abstract: true,
    props: {
        //  成功匹配的缓存
        include: patternTypes,
        // 匹配成功的不缓存
        exclude: patternTypes,
        //  最多缓存的组件实例
        max: [String, Number]
    },
    created () {
        // 缓存 vnode
        this.cache = Object.create(null)
        this.keys = []
    },
    destroyed () {
        for (const key in this.cache) {
            pruneCacheEntry(this.cache, key, this.keys)
        }
    },
    mounted () {    
        // 监察include exclude
        this.$watch('include', val => {
            pruneCache(this, name => matches(val, name))
        })
        this.$watch('exclude', val => {
            pruneCache(this, name => !matches(val, name))
        })
    },
    // keep-alive单独实现的render函数 keep-alive组件渲染时调用
    render () {
        const slot = this.$slots.default
        // keep-alive 只处理第一个子元素
        const vnode: VNode = getFirstComponentChild(slot)
        const componentOptions: ?VNodeComponentOptions = vnode && vnode.componentOptions
        if (componentOptions) {
            // check pattern
            const name: ?string = getComponentName(componentOptions)
            const { include, exclude } = this
            // 如果include / exclude匹配 就直接返回这个vnode
            if (
            // not included
            (include && (!name || !matches(include, name))) ||
            // excluded
            (exclude && name && matches(exclude, name))
            ) {
                return vnode
            }
        const { cache, keys } = this
        const key: ?string = vnode.key == null
        // same constructor may get registered as different local components
        // so cid alone is not enough (#3269)
        ? componentOptions.Ctor.cid + (componentOptions.tag ? `::${componentOptions.tag}` : '')
        : vnode.key
        if (cache[key]) {
        vnode.componentInstance = cache[key].componentInstance
        // make current key freshest
        remove(keys, key)
        keys.push(key)
        } else {
            cache[key] = vnode
            keys.push(key)
            // prune oldest entry 超出 缓存长度删除
            if (this.max && keys.length > parseInt(this.max)) {
                pruneCacheEntry(cache, keys[0], keys, this._vnode)
            }
        }
        // 这里标记keepAlive为true 后面会用到
        vnode.data.keepAlive = true
    }
    return vnode || (slot && slot[0])
    }
}
```
1. 组件patch过程
```
function createComponent (vnode, insertedVnodeQueue, parentElm, refElm) {
    let i = vnode.data
    if (isDef(i)) {
        //  执行init钩子函数时会用到 
        const isReactivated = isDef(vnode.componentInstance) && i.keepAlive
        if (isDef(i = i.hook) && isDef(i = i.init)) {
            // createComponent.init
            i(vnode, false /* hydrating */)
        }
        // after calling the init hook, if the vnode is a child component
        // it should've created a child instance and mounted it. the child
        // component also has set the placeholder vnode's elm.
        // in that case we can just return the element and be done.
        if (isDef(vnode.componentInstance)) {
            initComponent(vnode, insertedVnodeQueue)
            insert(parentElm, vnode.elm, refElm)
            if (isTrue(isReactivated)) {
                reactivateComponent(vnode, insertedVnodeQueue, parentElm, refElm)
            }
            return true
        }
    }
}
  ```
2. isReactivated 为 true时，在执行 init 钩子函数的时候不会再执行组件的 mount 过程
```
// inline hooks to be invoked on component VNodes during patch
const componentVNodeHooks = {
    init (vnode: VNodeWithData, hydrating: boolean): ?boolean {
        // 第二次渲染时已创建实例 且 keepAlive为true
        if (
            vnode.componentInstance &&
            !vnode.componentInstance._isDestroyed &&
            vnode.data.keepAlive
        ) {
            // kept-alive components, treat as a patch
            const mountedNode: any = vnode // work around flow
            componentVNodeHooks.prepatch(mountedNode, mountedNode)
        } else {
            const child = vnode.componentInstance = createComponentInstanceForVnode(
            vnode,
            activeInstance
        )
            child.$mount(hydrating ? vnode.elm : undefined, hydrating)
        }
    }
    //...
}
```
3.  回到createComponent ，当 isReactivated为true会调用eactivateComponent
```
function reactivateComponent (vnode, insertedVnodeQueue, parentElm, refElm) {
    let i
    // hack for #4339: a reactivated component with inner transition
    // does not trigger because the inner node's created hooks are not called
    // again. It's not ideal to involve module-specific logic in here but
    // there doesn't seem to be a better way to do it.
    let innerNode = vnode
    while (innerNode.componentInstance) {
        innerNode = innerNode.componentInstance._vnode
        if (isDef(i = innerNode.data) && isDef(i = i.transition)) {
            for (i = 0; i < cbs.activate.length; ++i) {
                cbs.activate[i](emptyNode, innerNode)
            }
            insertedVnodeQueue.push(innerNode)
            break
        }
    }
    // initComponent时缓存了 实例的$el
    // vnode.elm = vnode.componentInstance.$el
    // unlike a newly created component,
    // a reactivated keep-alive component doesn't insert itself
    insert(parentElm, vnode.elm, refElm)
}
```

### 组件的处理过程
组件的处理过程大致如下

 ![处理过程](http://94.191.25.51/uploads/images/4ef2dadb5b693.png)