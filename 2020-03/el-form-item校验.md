# el-form-item校验的一个场景

实际业务中我们可能会遇到根据依赖条件控制表单元素展示隐藏的场景，因为考虑到展示频次或者其他限制用了`v-show`而不是`v-if`，我们的表单元素校验规则也需要动态化，即隐藏的元素在隐藏时的规则应该为空，否则提交时会校验到这个隐藏的元素。但是当我们将该元素展示的时候却发现怎么都不能触发校验了。本文就主要分析下这个原因。

## 解决过程
我的场景是这样的：
```
<el-form-item
  v-for="(item,index) in group"
  v-show="condition"
  :key="index"
  :rules="condition?item.rules:[]"></el-form-item>

```
由于官方建议类似条件应该在计算group的时候处理，`if`是不建议使用的就偷懒用了`v-show`，于是就导致了上述展示后校验不触发的问题。尝试的hack方法就是主动去调用formitem的`validate`。但是这种hack方法有局限性，局限在于当该组件并没有展示隐藏的过程，会有重复触发的问题。最终解决方法还是从`v-show`入手，重新编译应该就没问题了吧：
```
<template v-for="(item,index) in group">
<el-form-item
  v-if="condition"
  :key="index"
  :rules="condition?item.rules:[]"></el-form-item>
</template>

```
以上，果然解决了。

## 发现原因

问题时解决了，但是原因还不清楚。做了对照组实验，发现display并不会影响rules的正确性，也不存在展示隐藏后`change/blur`不触发的问题，那既然触发了事件并向上`dispatch`了那问题是不是出现在监听阶段（当然这个是事后知道的，当时并没有那么敏锐），查看源码发现:
```
mounted() {
  if (this.prop) {
    this.dispatch('ElForm', 'el.form.addField', [this]);

    let initialValue = this.fieldValue;
    if (Array.isArray(initialValue)) {
      initialValue = [].concat(initialValue);
    }
    Object.defineProperty(this, 'initialValue', {
      value: initialValue
    });
    // 有prop才会去绑定校验规则
    this.addValidateEvents();
  }
},
methods:{
  addValidateEvents() {
    const rules = this.getRules();
    // 且校验规则不为空才会注册监听
    if (rules.length || this.required !== undefined) {
      this.$on('el.form.blur', this.onFieldBlur);
      this.$on('el.form.change', this.onFieldChange);
    }
  },
}
```
至此问题就明白了。form item的校验逻辑，是在mounted的时候去注册监听blur、change的方法，这个注册会判断rules不管是form的rules还是直接给formitem的rules，如果在mounted的时候rules为空或者空数组，就不会注册了，动态计算的rules即使有改动，如果不触发mounted的话也不会再监听到内部表单元素的事件。也解释了为什么隐藏的元素再次展示的时候无法通过事件触发校验的问题。
