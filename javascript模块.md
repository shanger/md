# javascript模块体系
## 前言
入职大半年接手了公司的不少项目，其中大部分项目都采用了类似的技术架构，目录结构、代码风格、技术选型都几乎一致，除了一些项目中对部分内容进行了升级和优化，很多东西都是复用的。复用虽然有各种各样的好处，但有一点是让我比较紧张的，那就是一旦这个复用的部分出了问题那所有的项目都有影响，所以最近开始排查一些项目通用组件和通用方法，大问题没找到却发现了自己对js模块上理解的欠缺。
## 举例
* vue插件
```
// $loading
const Loading = {}

Loading.install = function(Vue){
    Vue.prototype.$loading = ()=>{
        //...
    }
}
module.exports = Loading
```
* vue单文件组件
```
<template>
{!------}
</template>
<script>
import bala from './bala.js'
export default{
    props:{}
    //...
}
</script>
```

简单列出两种场景，以上分别使用了CommonJs和ES6 module的语法，因为webpack支持以上两种模块语法，所以之前并没有意识到，最主要的原因还是对js的几种模块规范不太了解，写代码的时候只是仿别人，所以才写出了这样的代码。
那么就在重构代码的同时梳理下js的几种模块规范。
## AMD , CMD , CommonJs 和 ES6 module
### CommonJs
> CommonJS 规范是为了解决 JavaScript 的作用域问题而定义的模块形式，可以使每个模块它自身的命名空间中执行。该规范的主要内容是，模块必须通过 module.exports 导出对外的变量或接口，通过 require() 来导入其他模块的输出到当前模块作用域中

eg:
```
// moduleA.js
module.exports = function( value ){
    return value * 2;
}
```
```
// moduleB.js
var multiplyBy2 = require('./moduleA');
var result = multiplyBy2(4);
```
特点:

1. commonjs是同步的，多用于服务端。
2. 输出值的copy。
3. 导出的module.exports 是一个对象，且该对象在被require时生成。
4. 参考第3点，当循环加载的时候，只输出被已执行的部分。
5. 重复加载某个模块，该模块并不会重复执行，而是取缓存。

**着重说一点--模块的缓存**

通过一个例子看一下：
```
// moduleA.js
exports.message = "hi";
console.log('ss')
exports.say = function () {
  console.log(message);
}
```
```
// moduleB.js
require('./module1.js');
require('./module1.js');
// ss 只输出了一次
```
那么，如何清除缓存呢？所有缓存的模块都保存在`require.cache`中，删除缓存可以这样做:
```
// 删除某个缓存
// require.resolve 查询某个模块文件的带有完整绝对路径的文件名
delete require.cache[require.resolve('./module1.js')]

// 删除所有模块的缓存
Object.keys(require.cache).forEach(function(key) {
  delete require.cache[key];
})
```
再测试下上面的例子：
```
require('./module1.js');
delete require.cache[require.resolve('./module1.js')]
require('./module1.js');
// ss
// ss
```
有趣的拓展：[一行 delete require.cache 引发的内存泄漏血案](https://zhuanlan.zhihu.com/p/34702356)

### AMD/CMD
将AMD和CMD放在一起的原因是因为这两种规范都应用于浏览器端。由于服务器端的所有资源模块都放在本地磁盘，可以同步加载完成，但是如果把大量资源在客户端同步加载完成，显然是很难让人接受的，于是异步加载的AMD/CMD应运而生。

#### AMD

`require([dependencies], function(){})`

接受两个参数：

1. `dependencies` 数组，所依赖的模块
2. 回调函数，当前面的模块加载成功后，被调用

#### CMD

```
/*
* require 用来获取其他模块提供的接口
* exports 向外提供模块接口
* module 一个对象
*/
define(function (require, exports, module) {
    //依赖就近原则，在哪里使用，在哪里引入
    var $ = require('jquery');
    //输出模块中定义的方法
    exports.sayHello = function () {
        $('#hello').toggle('slow');
    };
});
```
#### AMD与CMD的区别

* AMD推崇依赖前置，在定义模块的时候就要声明其依赖的模块
* CMD推崇就近依赖，只有在用到某个模块的时候再去require
*  两者代码质量有差异。RequireJS 是没有明显的 bug，SeaJS 是明显没有 bug [--玉伯](https://www.zhihu.com/question/20342350) 

### ES6 module

ES6 module提倡一个文件一个模块的思想，主要有两个命令：export和import，用于模块向外提供接口和引入其他模块接口。推荐下阮一峰阮老师的书，写的非常详细[Module 的语法](http://es6.ruanyifeng.com/#docs/module)。我这里不做过多的叙述，主要想记录两点：
1. 以目前的各种工具库，可在服务端和浏览器端同时支持ES6模式
2. ES6 module是编译时加载，所以没办法引用模块本身，导入的是值的引用。编译时加载的好处就是可以静态分析。

## 参考文档

* [Module 的语法](http://es6.ruanyifeng.com/#docs/module)
* [LABjs、RequireJS、SeaJS 哪个最好用？为什么？](https://www.zhihu.com/question/20342350)
* [nodejs 工具文档](https://nodejs.org/api/modules.html#modules_require_id)
* [webpack的模块](https://www.webpackjs.com/concepts/modules/)
