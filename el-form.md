
主要内容是elementUI的应用和组件定义的分享和讨论。
主要背景是我们组业务主要在营销相关的各种后台技术选型选的elementUI是我们比较常用的框架，所以在应用上积累了一些经验主要还是应用上的可能性一些经验发现了更多elementUI本身的特性或者Vue的新知识点从而更好的服务业务，
还有就是为了服务我们前几个月同步发展起来的几个运营平台我们做了一个基础库`base-be-fe`，在这个库的初级阶段我们迅速从各业务系统中抽离出大量公用方法、组件，并短时间内在各个运营系统中接入了base，后续迭代过成功也发现了我们在组件定义上的一些不合理的地方，考虑后续迭代成本我们希望在组件设计和功能上有个规范定义，避免由于大家开发风格的不一致降低这个库的好用程度。

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
都是返回`el-form`的组件实例，主要是这里有个`isNested`用来判断是否是嵌套，来做样式的处理，其他没有想到。
```
export function resolveInject (inject: any, vm: Component): ?Object {
  if (inject) {
    // inject is :any because flow is not smart enough to figure out cached
    const result = Object.create(null)
    const keys = hasSymbol
      ? Reflect.ownKeys(inject)
      : Object.keys(inject)

    for (let i = 0; i < keys.length; i++) {
      // ...
      while (source) {
        if (source._provided && hasOwn(source._provided, provideKey)) {
          result[key] = source._provided[provideKey]
          break
        }
        // 不论组件层次有多深，并在起上下游关系成立的时间里始终生效
        source = source.$parent
      }
      if (!source) {
        // ...
      }
    }
    return result
  }
}

```

### 父组件和子组件的联系

`el-form`访问`el-formitem`实例并不是通过$children来实现的，而是`el-formitem`自己主动将自己的信息注册到了`el-form`。
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
我们先不讨论这样做的优劣，这样做的目的出于功能上的考量，通过上报的方式来过滤没有`prop`属性即不需要做校验的的item，其实这里还是要讨论组件设计的问题。这样的做方法会让父组件保存冗余的`el-formitem`实例信息，牺牲了内存换来了什么？`prop`的意义也仅就这一个用处，如果不需要校验没必要给每个item添加这个属性。如果我们不通过`dispatch`的方式将这些formitem存储到form的实例上，当我们使用他们的时候就要从`chlidrend`中通过判断组件的name和有没有`prop`来过滤。

前面有提到form校验的收集通过prop，那么不需要校验的元素尽量不要添加prop属性。

### 校验
模式：required、Pattern、Range、Length

* string: Must be of type string. This is the default type.
* number: Must be of type number.
* boolean: Must be of type boolean.
* method: Must be of type function.
* regexp: Must be an instance of RegExp or a string that does not generate an exception when creating a new RegExp.
* integer: Must be of type number and an integer.
* float: Must be of type number and a floating point number.
* array: Must be an array as determined by Array.isArray.
* object: Must be of type object and not Array.isArray.
* enum: Value must exist in the enum.
* date: Value must be valid as determined by Date
* url: Must be of type url.
* hex: Must be of type hex.
* email: Must be of type email.
* any: Can be any type.

#### validator

* rule: 当前校验字段在 descriptor 中所对应的校验规则
  field 
  fullField
  required
  type
  validator
* value: 当前校验字段的值
* callback: 在校验完成时的回调
* source: 传入 validate 方法的 object，也就是需要校验的对象
* options: 传入的额外选项

除了async-validator,本身提供的几种类型的校验，特殊的校验需要使用validator、asyncValidator方法，但是这两个个方法都不允许传入其他参数，需要额外数据的就在作用域内取。
这部分一方面介绍async-validator内置的校验类型，还有就是想让大家参考下他们的校验方法。
```
  integer(value) {
    return types.number(value) && parseInt(value, 10) === value;
  }
```

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
组件化开发的思想是我们大家都比较熟悉的而且是应用非常多的一种编程思想，但是可能因为一些客观原因比如个人的编码习惯、业务的实现思路不一致导致组件的风格不一致，规范的定义是为了减少大家由于代码风格和实现差异带来的疑问和不适应；我们组件化的目标是为了模块拆分、代码复用，是一种解耦操作，所以要对组件的功能和独立性有一定的要求；组件的功能能力定义尽量完善，避免因为初期设计的缺陷导致升级不够平滑对已有代码逻辑有较大入侵。
* 提供完整的功能
  尽可能完整，减少因为少了些使用频率低但功能而进行的迭代
* 独立性
  独立性不是一个外部方法都不引入，是减少对业务代码的入侵。提供完整可用的输出，非组件功能可以提供各个节点的钩子。
其实在我们的开发场景中，为了使代码逻辑清晰会将一部分业务代码组件化，虽然复用程度可能不高，但也一定保证功能逻辑的内聚。