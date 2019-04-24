# vue 页面切换动画

## 前言
在之前的公司移动端页面采用的是backbone+jq+handlebar的体系，每个单页切换路由时每个页面都不会销毁，in->out->reverse的来回切换，动画效果也做的不错，体验比较舒服。后来技术选型调整，新功能完全由vue来做了，后续旧功能的迭代也用vue重构，可能是因为当时时间紧迫加之对vue的不熟悉，transition的这部分被忽略掉了，后来也没有重视起来以至于至今都没有提起，后续我自己开发中也没有注意到这方面的体验问题。直到最近，公司新项目h5由我主要负责，才想起来在体验方面还有很多要做的。

## 知识准备

vue内置组件`transition`提供了元素/组件进入的动画的过渡功能。
```
<transition name="fade" mode="out-in">
  <component :is="view"></component>
</transition>
```
几个主要参数：
> * name - string，用于自动生成 CSS 过渡类名。例如：name: 'fade' 将自动拓展为.fade-enter，.fade-enter-active等。默认类名为 "v"
> * appear - boolean，是否在初始渲染时使用过渡。默认为 false。
> * css - boolean，是否使用 CSS 过渡类。默认为 true。如果设置为 false，将只通过组件事件触发注册的 JavaScript 钩子。
> * type - string，指定过渡事件类型，侦听过渡何时结束。有效值为 "transition" 和 "animation"。默认 Vue.js 将自动检测出持续时间长的为过渡事件类型。
> * mode - string，控制离开/进入的过渡时间序列。有效的模式有 "out-in" 和 "in-out"；默认同时生效。
在进入/离开的过渡中，会在6个 class 间切换：

* v-enter：定义进入过渡的开始状态。在元素被插入之前生效，在元素被插入之后的下一帧移除。

* v-enter-active：定义进入过渡生效时的状态。在整个进入过渡的阶段中应用，在元素被插入之前生效，在过渡/动画完成之后移除。这个类可以被用来定义进入过渡的过程时间，延迟和曲线函数。

* v-enter-to: 2.1.8版及以上 定义进入过渡的结束状态。在元素被插入之后下一帧生效 (与此同时 v-enter 被移除)，在过渡/动画完成之后移除。

* v-leave: 定义离开过渡的开始状态。在离开过渡被触发时立刻生效，下一帧被移除。

* v-leave-active：定义离开过渡生效时的状态。在整个离开过渡的阶段中应用，在离开过渡被触发时立刻生效，在过渡/动画完成之后移除。这个类可以被用来定义离开过渡的过程时间，延迟和曲线函数。

* v-leave-to: 2.1.8版及以上 定义离开过渡的结束状态。在离开过渡被触发之后下一帧生效 (与此同时 v-leave 被删除)，在过渡/动画完成之后移除。

## 上手

了解到以上知识后，我们可以开始尝试应用了，因为讨论的是页面切换的动画，例子会结合vue-router。
```
<template>
    <div id="app">
        <transition :name="transitionName">
            <router-view class="view"></router-view>
        </transition>
    </div>
</template>

<script>
export default {

    name: 'App',
    data () {
        return {
            transitionName: 'slide-left'
        }
    },
    watch: {

        '$route' (to, from) {

            const toDepth = to.path.split('/').length
            const fromDepth = from.path.split('/').length
            this.transitionName = toDepth < fromDepth ? 'slide-right' : 'slide-left'

        }
    }
}

```
上面主要就是通过监听路由切换来决定此次动画的方向，原则就是进入内页左滑，反之则右滑，应该不会有太多问题。我个人觉得最难的就是怎么写这个动画，一开始疯狂测试改来改去都感觉不自然，然后就放弃了从网上copy了一份如下：

```
.slide-left-enter, .slide-right-leave-active {
    opacity: 0;
    transform: translate(50px, 0);
}

.slide-left-leave-active, .slide-right-enter {
    opacity: 0;
    transform: translate(-50px, 0);
}

.view{
    position: absolute;
    width:100%;
    transition: transform .8s cubic-bezier(.55,0,.1,1),opacity .8s cubic-bezier(.55,0,.1,1);
}
```
这里有几个点要给介绍下：

* `transition-property` 最好不要用`all`，既然是页面切换就只关注我们页面切换中调整的属性就好了，否则可能会有预期之外的动画，比如某些自适应元素大小变化，会比较别扭。
* `transition-timing-function` 过渡自不自然很大程度取决于动画函数写的怎么样，贝塞尔曲线也是我欠缺的知识，先埋个坑吧，后续填上。有个站点可以预览贝塞尔曲线的动画效果,可以去玩玩。[cubic-bezier](http://cubic-bezier.com/#.17,.67,.83,.67)
* 动画元素脱离文档流 这部分是考虑到性能问题具体知识也比较多，我这边提供几个连接，方便拓展知识
    * [CSS动画性能——重绘与重排](https://www.cnblogs.com/Kuro-P/p/8676771.html)
    * [详谈层合成（composite）](https://juejin.im/entry/59dc9aedf265da43200232f9)
* 被缓存的组件的子组件会放在他的父组件了一起被缓存，（keepalive只处理第一个组件）

# transition 和 keep-alive

有的时候页面切换我们希望保存一个页面的状态，比如某个页面有分页，我们就会希望把这个页面的状态保存下来，vue同样内置了组件`keep-alive`来帮助我们实现。那么当 `transition` 遇上 `keep-alive`会出现什么情况呢？
以下两种写法，你的第一直觉应该怎么写？（可能大佬们理解彻底不会出现这种犹豫）
```
{{!-- 方案一--}}
<transition :name="transitionName">
    <keep-alive include="home">
        <router-view class="view"></router-view>
    </keep-alive>
</transition>

{{!-- 方案二--}}
<keep-alive include="home">
    <transition :name="transitionName">
        <router-view class="view"></router-view>
    </transition>
</keep-alive>
```
我一开始选择的是方案二，然后就发现，keep-alive并没有生效，立刻头大了，这是怎么回事？于是开始回头看看这两个组件。
> transition 元素作为单个元素/组件的过渡效果

> `keep-alive` 包裹动态组件时，会缓存不活动的组件实例，而不是销毁它们。和 `transition`相似，`keep-alive`是一个抽象组件：它自身不会渲染一个 DOM 元素，也不会出现在父组件链中

再看下keep-alive的实现
```
// 分析第二种写法
// render函数

// 获取第一个子节点也只处理第一个节点
const slot = this.$slots.default
const vnode: VNode = getFirstComponentChild(slot)
const componentOptions: ?VNodeComponentOptions = vnode && vnode.componentOptions
if (componentOptions) {
    // check pattern
    const name: ?string = getComponentName(componentOptions) // transition
    const { include, exclude } = this
    // 命中即不是keep-alive的直接返回vnode
    if (
    // not included
    (include && (!name || !matches(include, name))) ||
    // excluded
    (exclude && name && matches(exclude, name))
    ) {
        return vnode
    }
    // 没有命中走缓存
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
        // prune oldest entry
        if (this.max && keys.length > parseInt(this.max)) {
            pruneCacheEntry(cache, keys[0], keys, this._vnode)
        }
    }

    vnode.data.keepAlive = true
}

```

通过测试我们知道第二种写法是没有办法实现我想要的结果的，不过我发现了个奇技淫巧：
测试代码：
```
<keep-alive include="ss">
    <!-- <transition> -->
        <compont-wrapper v-if="keepAlive"></compont-wrapper>
    <!-- </transition> -->
</keep-alive>

Vue.component('compont-wrapper', {
    name:'cc',
    data: function () {
    return {
        a:1
    }
    },
    created(){
        console.log('ss')
    },
    template: '<div><p>6666666666</p></div>'
})
```
首先我们看到，include和name的匹配逻辑，如果不满足name in include 的条件或者满足了name in exclude的条件，会直接返回不会存进缓存；transtion作为内置组件`name: 'transition',`,那我把它写进include不就行了？
想法是美好的，满足了缓存条件确实被写进了缓存，但是也仅仅是transtion被打上了keepAlive的标签写入了缓存，因为keepalive只处理了第一个子组件，内部组件不会被缓存。这种情况下基本不能满足我们的使用场景，权当是图个乐了。
贴上两个方法：
```
    // platforms/web/runtime/components
    // in case the child is also an abstract component, e.g. <keep-alive>
    // we want to recursively retrieve the real component to be rendered
    function getRealChild (vnode: ?VNode): ?VNode {
        const compOptions: ?VNodeComponentOptions = vnode && vnode.componentOptions
        if (compOptions && compOptions.Ctor.options.abstract) {
            return getRealChild(getFirstComponentChild(compOptions.children))
        } else {
            return vnode
        }
    }

    // core/vdom/helpers/get-first-component-child
    export function getFirstComponentChild (children: ?Array<VNode>): ?VNode {
        if (Array.isArray(children)) {
            for (let i = 0; i < children.length; i++) {
            const c = children[i]
            if (isDef(c) && (isDef(c.componentOptions) || isAsyncPlaceholder(c))) {
                return c
            }
            }
        }
    }
```

![normal](http://www.ilmiao.com/uploads/images/7c2fa8bd404a2.jpg)









