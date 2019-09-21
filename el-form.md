
主要内容是elementUI的应用和组件定义的分享和讨论。
主要背景是我们组业务主要在营销相关的各种后台技术选型选的elementUI是我们比较常用的框架，所以在应用上积累了一些经验主要还是应用上的可能性一些经验发现了更多elementUI本身的特性或者Vue的新知识点从而更好的服务业务，
还有就是为了服务我们前几个月同步发展起来的几个运营平台我们做了一个基础库`base-be-fe`，在这个库的初期我们也是经历了一些推广的痛苦，也发现了我们在组件定义上的一些不合理的地方，考虑后续迭代成本我们希望在组件设计和功能上有个规范定义，避免由于大家开发风格的不一致降低这个库的好用程度。

## Part1 el-form和el-formitem
这个组件应该是我们开发配置后台最常用的组件了，充分了解它充分利用它是我们最近一直在code的东西。分三部分来解读。
### 父子组件通信
* props
* events
* $parent/$children
* provide/inject

前两种就不细谈了，比较有意思的是后面两种，而且是两种都有用到有些地方用了provide的数据，有的地方则通过$parent来取值。
```
// provide

inject: ['elForm'],
// elForm
// item marginleft
ret.marginLeft = this.elForm.autoLabelWidth;
// fontsize
return this.elForm.size;
// 事件
this.elForm && this.elForm.$emit('validate', this.prop, !errors, this.validateMessage || null);

// $parent
computed:{
  form() {
      let parent = this.$parent;
      let parentName = parent.$options.componentName;
      while (parentName !== 'ElForm') {
        if (parentName === 'ElFormItem') {
          this.isNested = true;
        }
        parent = parent.$parent;
        parentName = parent.$options.componentName;
      }
      return parent;
    }
}
this.form.labelWidth
this.form.labelPosition
this.form.model

```
都是返回`el-form`的组件实例，主要是这里有个`isNested`用来判断是否是嵌套，来做样式的处理，其他没有想到。看了下提交记录`provide`确实在`form`之后，猜测可能是没有迁移完毕。在有大量数据要通信需要写繁琐的props仅仅是为了获取某些数据的时候可以考虑下使用。

### 父组件和子组件的联系

`el-form`访问`el-formitem`实例并不是通过$children来实现的，而是`el-formitem`自己主动将自己的信息提交给了`el-form`。
```
  // formitem
  if (this.prop) {
    this.dispatch('ElForm', 'el.form.addField', [this]);
  }
  // form
  this.$on('el.form.addField', (field) => {
    if (field) {
      this.fields.push(field);
    }
  });

```
我们先不讨论这样做的优劣，这样做的目的出于功能上的考量，通过上报的方式来过滤没有`prop`属性即不需要做校验的的item，其实这里还是要讨论组件设计的问题。这样的做方法会让父组件保存冗余的`el-formitem`实例信息，牺牲了内存换来了什么？


### restfields

提这个跟`el-formitem`的本身的设计是没有关系的，主要我个人的误区，我以为reset应该是还我一个空的表单，但我们的使用场景不是这样的。
```
 mounted() {

  if (this.prop) {
    // ...
    let initialValue = this.fieldValue;
    if (Array.isArray(initialValue)) {
      initialValue = [].concat(initialValue);
    }
    Object.defineProperty(this, 'initialValue', {
      value: initialValue
    });
    // ...
  }
},

resetField() {
  // ... 
  if (Array.isArray(value)) {
    prop.o[prop.k] = [].concat(this.initialValue);
  } else {
    prop.o[prop.k] = this.initialValue;
  }
  // ...

}

```
这个逻辑是没有问题的，我们有些场景是用dialog做编辑。
```
 <div class="el-dialog__body" v-if="rendered"><slot></slot></div>
 mounted() {
  if (this.visible) {
    this.rendered = true;
  }
 }
```
```
  this.formData = {
    // ...
  }
  this.dialogVisible = true
```

问题就在于只有dialog展示的时候才会渲染`el-form`，这个时候`initialValue`是你渲染后的初始化的数据，rest到也就是初始化的数据。

## 组件的设计规范
