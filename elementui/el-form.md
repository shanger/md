#form 解析
## 前言
读elementUI实现的出发点是对resetFiled的不理解，每次调用都reset不到我data里返回的初始值，所以就比较好奇reset的值是什么东西？然后在看组件实现的过程中也发现了一些其他的新的知识，一些自己没有用过的特性，我有个体会就是，这种UI库是框架的高级使用demo。今天的分享其实是跟大家一起读下代码，共同思考下他们的实现思路。

## 我的疑惑
应用场景是dialog中的表单，表单的model在data中定义，每次关闭弹窗前resetFiled，但是发现并没有重置到我data中定义的结果，这让我很困惑。没办法只能手动重置表单数据。
### 看看reset值从哪来


重置操作
```
resetFiled(){
  // ...
  if (Array.isArray(value)) {
    prop.o[prop.k] = [].concat(this.initialValue);
  } else {
    prop.o[prop.k] = this.initialValue;
  }
  // ...
}
```
`initialValue`来源
```
computed:{
  fieldValue() {
    const model = this.form.model;
    if (!model || !this.prop) { return; }

    let path = this.prop;
    if (path.indexOf(':') !== -1) {
      path = path.replace(/:/, '.');
    }

    return getPropByPath(model, path, true).v;
  }
}
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

    this.addValidateEvents();
  }
}
```
代码没有什么特别的，简单解读下，这个initialValue是在form创建的时候绑定的model，到这都没有任何的问题。在dialog的场景中我们的表单会放在dialog组价的非具名slot中， 这个slot是在visible的情况下才会渲染，所以导致我们reset的是reset到第一次打开的时候 可能是编辑也可能是添加，reset可能是预期外的效果。
```
<div class="el-dialog__body" v-if="rendered"><slot></slot></div>

mounted() {
  if (this.visible) {
    this.rendered = true;
    this.open();
    if (this.appendToBody) {
      document.body.appendChild(this.$el);
    }
  }
}

```

## el-form的规划
![el-form规划](https://www.ilmiao.com/uploads/images/78765bcfe7d0f.jpg 'el-from')

## 应用
### 验证的触发
验证触发都是从表单元素dispatch上来的，我们在一个formitem中校验触发条件的书写要考虑到其中的表单元素能触发什么样的事件。其中需要注意的是upload并不会触发任何验证，因为我们不会给type=file的input绑定值，不触发事件是合理的，我们在input的事件出发后手动调用`validate`就好了。
ps:还有一点就是upload的组件会在上传时将input的值置为`null`，所以即使你选了连个同样的文件第二次也会触发input的change
表单元素|事件
--|:--:
input|blur change
data-picker|blur change
select|change
upload|none


## 几个设计上的考虑探讨

### fileds的收集
el-from 只处理有prop的fromItem,form的model是一个对象，prop是访问的路径也就是必要条件。所以在收集列表的时候就处理了，而不是在下发指令的时候过滤。
```
if (this.prop) {
  this.dispatch('ElForm', 'el.form.addField', [this]);
  this.dispatch('ElForm', 'el.form.removeField', [this]);
  // ...
}
```
### 依赖
有个疑问，form-item 获取form的方式有两种一种是通过父子链，一个是provide加inject 同时存在，部分用elForm，部分用form ，provide的优势就是可以向所有子组件传入依赖，后面表单元素和el-form建立联系就是是通过provide（form改成elFrom run test，结果基本一致）。
![el-form规划](https://www.ilmiao.com/uploads/images/21bf80629564.png 'el-from')

### clearable
这个是我遇到bug后去看的，select数据格式是int，清空之后给我弄成了空字符串。clear的逻辑比较粗暴,找了几个支持clearable的组件大家看一下，注意下就行了。
```
// input

clear() {
  this.$emit('input', '');
  this.$emit('change', '');
  this.$emit('clear');
}
// select
const value = this.multiple ? [] : '';
this.$emit('input', value);
this.emitChange(value);
// date
this.emitInput(null);
this.emitChange(null);

```
clearable的清除  清除完了之后是否考虑有初始值的情况

### el-input 

这里提到他是因为他没有用v-model这个让我比较好奇，毕竟vue的特色。这个是为了解决中文输入时，dom更新导致事件崩坏数据视图不同步的问题。
input setNativeInputValue 
https://github.com/ElemeFE/element/issues/14521

## elementUI版本
v2.10.1








