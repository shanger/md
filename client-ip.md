# 谁动了我的数据库

## 前言
入行两年多，终于在前一段下定决心搭建自己的个人博客了。买服务器买域名（服务器腾讯的，域名阿里的，至今还在用ip访问233），koa+sequelize+mysql，一通修修补补勉强上线了，目前只支持管理部分的文章更新、上传，访客部分的查看功能，上述基本弄好之后就停滞了一段时间开始进入疯狂加班的阶段。
元旦节前抽空打开首页抽了一眼居然分页了300多条，当时就惊了，这提莫都是哪来的？查看数据库发现都是一些脏数据，当时也没想着查看日志，就把之前没加的操作权限加上了才告一段落。
后来心血来潮打算去看看这都是谁做的骚操作，打开日志之后惊了怎么全都是127.0.0.1？
## 查明原因
这是怎么回事？查了一通发现是自己埋的坑，第一次买服务器上来搭建好环境之后，装了jenkins和Nginx想自己玩儿，后来发现自己￥99的便宜货根本弄不来jenkins，一构建就重启，jenkins玩儿不了nginx得能玩儿吧，于是就写了反向代理到我的博客，这样做之前的host和remoteadderss都会丢失，也导致了我拿不到真实的客户端ip，真的是有趣，哈哈哈哈
**具体原因**

* 访问nginx转发后的服务
```
curl http://94.191.25.51
remoteAddress: 127.0.0.1
```
* 直接访问 node服务
```
curl http://94.191.25.51:3003
remoteAddress: 116.236.215.22
```
这就导致了我日志里看到的都是:
```
[2019-02-14T14:31:15.767] [INFO] resLogger - 
*************** response log start ***************
request method: GET
request originalUrl:  /article/js/33
request client ip:  ::ffff:127.0.0.1 
user agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/67.0.3396.62 Safari/537.36
request query:  {}
response time: 16
response status: 200
*************** response log end ***************
```

## 解决问题
问题找到了，那么怎么去解决呢？
上面也说到了，经过nginx代理后原先请求的host和remoteAdderss都会丢失，那我们就把他在转发时候保存下来。这就需要修改nginx的配置了。
* nginx
```
location / {
    proxy_pass http://*****;
    proxy_set_header Host $host:$server_port; 
    proxy_set_header X-Real-IP $remote_addr; #保留代理之前的真实客户端ip
    proxy_set_header X-Real-PORT $remote_port;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
}
```
>$proxy_add_x_forwarded_for
>每次经过proxy转发都会有记录,格式就是client, proxy1, proxy2,以逗号隔开各个地址

* node
```
    app.proxy = true
    clientIp = ctx.request.ip
```
* 看看koa是怎么做的 
```
    // ips
    get ips() {
        const proxy = this.app.proxy;
        const val = this.get('X-Forwarded-For');
        // ips:[client, proxy1, proxy2]
        return proxy && val
        ? val.split(/\s*,\s*/)
        : [];
    },
    // ip
    get ip() {
        if (!this[IP]) {
            // ips的第一个即使clientip
            this[IP] = this.ips[0] || this.socket.remoteAddress || '';
        }
        return this[IP];
    }
```

以上。

## 参考资料
[http://www.cnblogs.com/huangjingzhou/articles/2155892.html](http://www.cnblogs.com/huangjingzhou/articles/2155892.html)

[https://github.com/koajs/koa/issues/599](https://github.com/koajs/koa/issues/599)

[https://imququ.com/post/x-forwarded-for-header-in-http.html](https://imququ.com/post/x-forwarded-for-header-in-http.html)