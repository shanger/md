# 从浏览器输入URL拓展开
## 前言
“从浏览器输入URL到页面加载的过程发生了什么？”，这个问题经历过面试就应该在各个博客平台中见到过，内容丰富程度不一，有的可能比较简单寥寥几条但也能基本描述完整过程，但有心的博主就会从每个阶段一一展开分析，我也是受此影响发现每个环节都隐藏着丰富的知识点，而这些知识点可能会成为以后可以优化进阶的点，整理和掌握都很有必要。于是本篇笔记是一篇用来留坑的，后面会慢慢补上。
## 寥寥几条版
1. DNS解析
2. TCP连接
3. 发送HTTP请求
4. 服务器处理请求并返回HTTP报文
5. 浏览器解析渲染页面

## 主要坑位

### 根据域名查找IP
关键词：dns、dns缓存

[DNS解析过程及部分知识点拓展](https://www.ilmiao.com/article/js/19)
### 建立连接
关键词：http、https、http2、三次握手、四次挥手、https的优化、缓存、负载均衡、gzip
* [https知识整理](https://www.ilmiao.com/article/js/11)
* [TLS 握手优化详解](https://imququ.com/post/optimize-tls-handshake)
* [四次挥手](https://www.ilmiao.com/article/js/20)
### 页面解析
关键词：dom、cssom、layer tree、

## 传送门
如果您等不及可以先看看下面这位大佬的文章！早学习！早受益！
[从输入URL到页面加载的过程？如何由一道题完善自己的前端知识体系！](http://www.dailichun.com/2018/03/12/whenyouenteraurl.html)