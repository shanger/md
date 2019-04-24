# CSS书写规范
*模块化结构化*   
*本文讨论的书写规范主要针对以下三点问题的优化*
* *团队化开发*
* *css选择器性能*
* *方便自动化测试*

## 团队开发
团队开发最重要的是风格统一，认识一致。风格统一方便代码阅读和协同工作，认识一致减少沟通成本。
在css命名规范上为达到开篇说到的三点，团队成员进行了讨论也在个人开发习惯上做出了一些改变和牺牲。接下来针对我们自己备选的几种方案一一分析。  
### BEM
blockName-eleName-modifier 模块-节点-修饰符
第一次认识到这种命名方式对我个人来说是眼前一亮的，以前总是头疼css的命名总是会遇到起名困难的尴尬，而且为了追求阅读体验开始了一种变态的结构写法:

`.left-article .header .title h2 { font-weight:550; }`

不像是写选择器，而像是在写xpath。上面这个例子还不是我写过最极端的，最长的我甚至写过七八层的嵌套，当然首先要考虑自己的html写的是否合理，嵌套过多显然是不推荐的。那么上面这行用BEM改造过后该如何书写呢？

`.leftArticleTitle-h2 { font-weight:550; }`

过多的嵌套会影响css性能（这个会在后面分析），我们在书写css的时候在保证准确选择到我们想要的元素的同时兼顾效率是我们想达到的。我上面的写法可能会被吐槽，大家要借鉴的是BEM的设计思想--抽象类和扁平化，做好这两点就是成功的应用。

### scope
一个尚在实验中的功能 [:scope](https://developer.mozilla.org/en-US/docs/Web/CSS/:scope)，但有些框架已经通过其他方式实现了相似功能（比如vue scoped)。我们前端框架就是vue所以在解决css权重问题的时候会用到这种方式。
先说立场，我个人是不推荐使用这种方式的，类似以作用域的方式解决问题可以采用其他方法，比如上面说到的BEM。我不推荐的原因是因为vue的实现方式是在css和html上加上唯一标记

`html`

`<div data-v-7b0680fe class="home"></div>`

`css`

`.home[data-v-7b0680fe]{width:500px;}`

以增加权重的方式达到模块化的目的，这种方式增加了选择器的复杂度，影响效率。

### module
css module
还是以vue为例，首先将css以模块分开，通过生成的className达到模块化的目的，其中className规则可在loader中配置。还是BEM设计思想的问题只要抽象做好完全可以规避冲突。


    <style module>
        .red {
            color: red;
        }
        .bold {
            font-weight: bold;
        }
    </style>
    <template>
    <p :class="$style.red">
        This should be red
    </p>
    </template>
    // loader 配置
    localIdentName: '[name]-[local]-[hash:base64:5]'


## css选择器性能
前面说到vue实现的scope有性能方面的问题，那么接下来就分析下css选择器的原理。
首先，排版引擎解析css选择器的方式是从右边往左解析。接下来分析为什么会这样做。

![浏览器解析过程](https://www.ilmiao.com:8081/uploads/images/11f29b0e3f59a.jpg)

我们分析在dom tree 和 style rules 建立Render tree的过程中，dom tree在style rules找到符合的selector并应用*（这里有个小知识就是为什么是给dom元素找样式规则，而不是根据样式规则找元素，是因为，即使没有style也是会生成render tree的所以才会根据dom来）*。两者并不是一一对应的，大量的节点不会被rules匹配，所以要有一个高效快速的匹配方式。

`div span p label`

以上面这个选择器为例反向推理，如果从左往右解析的话，路径就会是先找到div然后向下遍历，找不到label回溯到开始的div进行下次遍历，效率比较低。如果从右往左解析的话，就会先判断当前元素是不是label，不是就舍弃，是的话才会向上查找，在匹配率较低的场景下效率更高。
### css语法解析过程
1. 先创建CSSStyleSheet对象。将CSSStyleSheet对象的指针存储到CSSParser对象中。
2. CSSParser识别出一个simple-selector，形如"div"或者".class"。创建一个CSSParserSelector对象。
3. CSSParser识别出一个关系符和另一个simple-selecotr，那么修改之前创建的simple-selecotr, 创建组合关系符。
4. 循环第3步直至碰到逗号或者左大括号。
5. 如果碰到逗号，那么取出CSSParser的reuse vector，然后将堆栈尾部的CSSParserSelector对象弹出存入Vecotr中，最后跳转至第2步。如果碰到左大括号，那么跳转至第6步。
6. 识别属性名称，将属性名称的hash值压入解释器堆栈。
7. 识别属性值，创建CSSParserValue对象，并将CSSParserValue对象存入解释器堆栈。
8. 将属性名称和属性值弹出栈，创建CSSProperty对象。并将CSSProperty对象存入CSSParser成员变量m_parsedProperties中。
9. 如果识别处属性名称，那么转至第6步。如果识别右大括号，那么转至第10步。
10. 将reuse vector从堆栈中弹出，并创建CSSStyleRule对象。CSSStyleRule对象的选择符就是reuse vector, 样式值就是CSSParser的成员变量m_parsedProperties。
11. 把CSSStyleRule添加到CSSStyleSheet中。
12. 清空CSSParser内部缓存结果。
13. 如果没有内容了，那么结束。否则跳转值第2步。

#### 深入阅读 
* [css样式表解析过程](https://blog.csdn.net/shuimuniao/article/details/8601588)

* [How Browsers Work: Behind the scenes of modern web browsers](https://www.html5rocks.com/en/tutorials/internals/howbrowserswork/)

### 几点认知
+ id 是非常高效的选择器。
*不知为何现在少用id似乎成了一种政治正确，总是顺手写成class后来还没有任何复用场景*
+ 避免深层次的嵌套（参见解析过程，遍历组合过程会变长）。
+ 属性选择器的匹配过程更慢（先检测是属性选择器，再match不同的选择器类型）。
+ 合理使用继承属性，避免重复书写样式。
+ 慎用 ChildSelector [css解析规则](https://blog.csdn.net/shuimuniao/article/details/8601588)

## 自动化测试
会有这部分考虑的原因我们的页面中可操作元素没有唯一标志，导致我们自动化测试的同学很头疼。xpath在操作频繁更改dom结构的场景中有些吃力。反思：
* 没有写单元测试，做自动化的时候才暴露问题。
* 在模板中绑定事件，没有查询dom的需求。
总之，我们要应测试同学的要求在页面中给可操作元素添加唯一标志，因为这部分完全与业务无关，算是一个单独模块，于是上面讨论的结果就有了第一次全方位的应用场景。

