# 正则表达式

## 基础
基础用法简单过一下，建议参考百度百科。
## 一些概念
### 贪婪模式

默认是贪婪模式，就是说在满足该模式的情况下，会尽可能的匹配更多的。

eg: 
```
'<p>第一句</p><p>第二句了</p>'.match(/<p>.*<\/p>/)
````
我们有很多的匹配数量的元字符，*，+，{n}，{n,}，{n,m}

像上面的如果我们只想匹配到第一句该如何操作
1. 变成非贪婪模式
`'<p>第一句</p><p>第二句了</p>'.match(/<p>.*?<\/p>/)`
2. \b ^$ (没有成功再考虑)

其他贪婪的例子
{n,}
'hahaha.....oha'.2match(/(ha){2,}/)
+
'hahaha.....oha'.match(/(ha)+/)

### 捕获组
 捕获组就是把正则表达式中子表达式匹配的内容，保存到内存中以数字编号或显式命名的组里，方便后面引用。

普通捕获组：(Expression)

命名捕获组：(?<name>Expression) es2018 对于正则的拓展
eg: 
```
const RE_DATE = /(?<year>\d{4})-(?<month>\d{2})-(?<day>\d{2})/;
const matchObj = RE_DATE.exec('1999-12-31');
const year = matchObj.groups.year; // 1999
const month = matchObj.groups.month; // 12
const day = matchObj.groups.day; 
```

'13243189551'.replace(/(\d{3})(\d+)(\d{4})/,'$1****$3')

'#f60'.replace(/([0-9a-fA-F])/g,'$1$1') 

### 