# 柯里化

## 前言
我自己有一种体会就是有很多知识是自己看的懂、写的出但从来不会在自己的开发过程中用到的。最近才认识到这是因为自己对该知识缺乏深刻认识自然然对应用场景知之甚少，这都是因为自己存在“我知道了”的侥幸心理，非常不利于自己的提升。本文记录的内容算是一个节点，后续会提高对自己所学知识的要求争取做到知其然并知其所以然。
## 定义
在计算机科学中，柯里化（Currying）是把接受多个参数的函数变换成接受一个单一参数(最初函数的第一个参数)的函数，并且返回接受余下的参数且返回结果的新函数的技术。-- [百度百科](https://baike.baidu.com/item/%E6%9F%AF%E9%87%8C%E5%8C%96)
## 理解

```
function sum(a,b,c){
    console.log(a + b +c)
}
sum(1,2,3) // 6

```
以上方法，如果按柯里化的方式调用函数该怎么写？
>把接受多个参数的函数变换成接受一个单一参数
`sum(1)(2)(3)`
如上，`sum(1)`应当返回一个函数，因为后面要接着执行`fn(2)(3)`,那么这种函数该如何实现呢？
```
function curry(fn,cArgs){
    return function(){
        var args = [].slice.call(arguments)
        cArgs && args = args.concat(cArgs)
        // 跳出递归的条件 fn的参数需要指定
        if (args.length < fn.length) {
            return curry(fn, args);
        }
        return fn.apply(null, args);
    }
}
function fn(a,b,c){
    console.log(a + b +c)
}
var sum = curry(fn)
sum(1)(2)(3) // 6
```
## 应用场景
通过上面`curry`函数可以看到，柯里化函数的本质就是闭包，递归的作用域嵌套。那么利用这些特性就可以做一些事情了。
* 参数的复用

```
function curry(regards){
    return function(a){
        console.log(regards+a')
    }
}
var sayhai = curry('过年好！')
sayhai('张三') // 过年好！张三
sayhai('李四') // 过年好！李四
```
每次sayhai不用重复传入‘过年好！’，达到复用
* 延迟计算

计算一天内每顿饭平均花多少钱

```
var currying = function(fn) {
    var args = [].slice.call(arguments,1)
    return function() {
        if(arguments.length === 0){
            return fn.apply(null, args)
        }else{
            // 累计传入参数
            args = args.concat([].slice.call(arguments))
        }
    };
};

var getAverage = currying(function() {
    var count = 0
    Array.prototype.forEach.call(arguments,ele=>{
        count += ele
    })
    console.log(count/3)
});

getAverage(6);
getAverage(20);
getAverage(18);
getAverage(); //14.666666666666666

```
## 注意事项
* 闭包和多次作用域嵌套会带来更多的花销
