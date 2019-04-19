# js汉字转拼音

## 前言
项目中需要根据gps获取城市定位，就gps+百度的地图来做了，有个问题就是定位返回的信息只有汉字，需要映射到路由上就必须将汉字转为拼音了。其实并不是什么新鲜的需求和新鲜的技术，记录下来主要是为了分析下实现的方法。
## 准备
汉字转拼音的字典就不需要手写了，网上一搜一大堆[传送门](https://github.com/shanger/wheels/blob/master/js/pinyin.js)。
## 开始

```
import Pinyin form './*/pinyin.js'
function ConvertPinyin (str1) {
    // 图简单 过滤掉无关字符
  var _str1 = str1.replace(/[^\u4e00-\u9fa5]/g, '')
  var str2 = _str1.length
  var result = ''
  for (var i = 0; i < str2; i++) {
    var val = _str1.substr(i, 1)
    var name = arraySearch(val)
    if (name !== false) {
      result += name
    }
  }
  return result
}
// 遍历字典
function arraySearch (str) {
  for (var name in PinYin) {
    if (PinYin[name].indexOf(str) !== -1) {
      return name
    }
  }
  return false
}

```
上述代码没放在pinyin.js里的原因是因为大家的实现方式可能不一样，要求也不一样，比如有的要大写首字母啥的，其实数据字典也可以定制化，我看到有的还要音调真的是会玩儿。
## 原理分析
>Unicode只有一个字符集，中、日、韩的三种文字占用了Unicode中0x3000到0x9FFF的部分。Unicode目前普遍采用的是UCS-2,它用两个字节来编码一个字符，一般用16进制来表示。

我们来看一下这段代码：
```
for (var name in PinYin) {
    if (PinYin[name].indexOf(str) !== -1) {
      return name
    }
}
```
因为JavaScript 允许采用\uxxxx形式表示一个字符，`\u5594`即表示‘喔’，所以我们才可以在`Pinyin`里map到‘喔’的拼音‘o’，原理非常的简单。困难的是字符和\uxxxx的对应，本文提到的数据字典是我从网上找的，match了一下只有6763个汉字（es6针对这个做了扩展[es6 字符串拓展](http://es6.ruanyifeng.com/#docs/string)），然而汉字大概有六万多个，所以他可能不能满足你的全部需求，如果有需要可以自己去找映射关系[Unicode汉字编码表](http://www.cnblogs.com/whiteyun/archive/2010/07/06/1772218.html)。
## 疑问
网上找到的资源都是用`\uxxxx`这种形式来match字符的，那么为什么不直接用字符来匹配呢？
一开始想了很多，是不是有性能问题，但是没有能有效的论证。所以就在想是不是有什么奇怪的字不能直接处理，于是找到了一个‘丨’，这个字丨，是一个象形文字，是一个多音字，读音主要有shù、gǔn，外文名叫Radical Line，意思是指古姓氏。[|百科](https://baike.baidu.com/item/%E4%B8%A8)。好神奇，但是不是我猜测的结果：
```
'|'.indexOf('|') // 0
'\u4e28'.indexOf('|') // -1
```
卧槽？这是什么情况，原来这个真的很难打出来，正确结果应该是这样的。
```

'丨'.indexOf('丨') // 0
'\u4e28'.indexOf('丨') // 0
```
并没有论证我的猜想。233
## 总结
其实内容不多，知识点也不多，但还是挺有趣的，有兴趣可以继续深入。