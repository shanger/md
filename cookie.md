# cookie知识整理
## 前言
关于cookie的应用在满足我们各种开发需求的同时也给我们带来了很多问题。比如安全性问题，由于存放于客户端由http传输很容易被窃取即使设置了secure和httponly也不代表安全；流量问题，cookie会被放在request header中一并发送给服务端，而对有些服务这些信息是不必要的所以这部分开销当然也就没必要了，于是我们做了域名拆分；而让我起意做些cookie知识整理的主要原因并不是以上这两点。
## 事出有因
项目有个需要和app交互的部分，页面上需要展示个文案需要从app传入，定的方案是写进cookie，从node服务读取的时候发现乱码了“æ±æ”，这完全不可用了，后来了解到cookie写入最好不要写入非ASSII字符，app 编码utf-8后解决了问题。发现自己对cookie的了解还是不够，于是就想整理下cookie的相关知识。
## cookie的知识
### cookie的创建
#### 服务端创建
以koa为例：

`ctx.cookies.set(name, value, [options])`

通过 options 设置 cookie name 的 value :

* maxAge 一个数字表示从 Date.now() 得到的毫秒数
* signed cookie 签名值
* expires cookie 过期的 Date
* path cookie 路径, 默认是'/'
* domain cookie 域名
* secure 安全 cookie
* httpOnly 服务器可访问 cookie, 默认是 true
* overwrite 一个布尔值，表示是否覆盖以前设置的同名的 cookie (默认是 false). 如果是 true, 在同一个请求中设置相同名称的所有 Cookie（不管路径或域）是否在设置此Cookie 时从 Set-Cookie 标头中过滤掉。
#### 客户端创建
通过document.cookie创建

`document.cookie = name + '=' + escape(value) + ';expires=' + exp.toGMTString() + ';path=/'`
重点也是上面提到的问题，就是客户端cookie值的限制：
>此外，规范没提到，而且浏览器支持不一致的是非ASCII（Unicode）字符：在Opera和谷歌Chrome，它们会以UTF-8的编码的形式加到 Cookie头;在IE中，使用的是系统默认编码（本地指定编码，绝不是UTF-8）;火狐（和其他基于Mozilla的浏览器）使用的是每个UTF-16代码点的低字节（所以ISO-8859-1正常，但别的是错位的）;Safari浏览器简单粗暴的拒绝发送任何包含非ASCII字符的cookie。
--[cookie中的转义字符的方法是叫什么规范？](https://www.zhihu.com/question/46672990/answer/102290211)

**但是浏览器写入汉字并不会报错**，大佬告诉我这叫feature，我用node写了一下就给我报错了`argument value is invalid`，感觉还是很友好。上面也提到了非ASCII字符写入时最好编码一下。读取时用对应方法解码即可
### cookie配置的详解
#### name 和 value
有一些规范没提到但浏览器一直都支持的“feature”：
* name和value都可以为空字符串
* 如果字符串中并没有等号‘=’，浏览器把它当作name为空的Cookie，即Set-Cookie：foo和Set-Cookie：= foo是一样的。
* 当浏览器输出一个name为空的cookie，其中的等号‘=’会被忽略。因此，Set-Cookie：=foo和Set-Cookie： foo是一样的。
* 虽然等号‘=’周围的空格将被去掉，但逗号、空格在name和value中似乎也可以使用，控制字符（\ x00到\ x1F和\ 0x7F部分）是不允许使用的。

#### expires / maxAge
以上用来表示cookie的过期时间，如果不设置cookie就会是会话级的存储，会随浏览器关闭而清除。
* maxAge是一个数字，从Date.now()获取到的毫秒数。
* expires 过期时间的Date

#### path
cookie的路径，cookie使用权限的一种管理方式。比如：“/test”下的cookie，在“/ceshi”目录下并不能访问。默认值是“/”。

#### domain
cookie 域名，不同域名下的cookie不共享。

#### secure
安全cookie。设置为true时，只有在https协议下才能上传到服务器，在http协议下无法使用。

#### httponly
服务器可访问cookie，设置为true时，浏览器端无法获取使用。

#### samesite
允许服务器设置cookie不随跨域请求一起发送，这样可以在一定程度上防范跨站请求伪造攻击（CSRF）。
* Strict

Strict是最严格的防护，有能力阻止所有CSRF攻击。然而，它的用户友好性太差，因为它可能会将所有GET请求进行CSRF防护处理。
* Lax

只会在使用危险HTTP方法发送跨域cookie的时候进行阻止。

## 编码
其实这部分内容可以另写一篇记录，一方面觉得跟cookie合作是较好的应用场景，一方面又不打算写的太多，那就和cookie“合租”吧。
* escape unescape
已被w3c废弃，不再赘述。
* encodeURI
 > 函数通过将特定字符的每个实例替换为一个、两个、三或四转义序列来对统一资源标识符 (URI) 进行编码 (该字符的 UTF-8 编码仅为四转义序列)由两个 "代理" 字符组成)
* decodeURI
> 函数解码一个由encodeURI 先前创建的统一资源标识符（URI）或类似的例程
* encodeURIComponent
> 是对统一资源标识符（URI）的组成部分进行编码的方法。它使用一到四个转义序列来表示字符串中的每个字符的UTF-8编码（只有由两个Unicode代理区字符组成的字符才用四个转义字符编码）
* decodeURIComponent
> 方法用于解码由 encodeURIComponent 方法或者其它类似方法编码的部分统一资源标识符（URI）。

四个函数，使用编码解码方式要对应，而这两组函数区别就在于多了componet，意义也就是encodeURI用来编码整个URI，encodeURIComponent用于编码URI的一部分,比如 protocol、host、path、qurey。


## 参考链接
* [cookie中的转义字符的方法是叫什么规范？](https://www.zhihu.com/question/46672990/answer/102290211)
* [escape,encodeURI,encodeURIComponent有什么区别?](https://www.zhihu.com/question/21861899)
* [HTTP Cookie](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Cookies)
* [Set-Cookie](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers/Set-Cookie)