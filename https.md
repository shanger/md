# TCP和TLS
## 前言
由于http是超文本传输协议，信息是明文传输，且当下的网络环境比较恶劣，被劫持和篡改的情况时有发生，于是更安全的https越发成为主流。而让传输过程更加安全的关键就是https在tcp和http之间增加了TLS。他提供的主要服务有：认证用户和服务器，确保数据发送到正确的客户机和服务器、加密数据以防止数据中途被窃取、维护数据的完整性三大功能。
### TCP
TCP是一种面向连接的、可靠的、基于字节流的传输层协议。本文主要讨论的是建立连接以及传输的安全性问题，即主要讨论建立连接的过程下文的TLS也是同样。下面就来看看tcp建立连接的三次握手过程。

![tcp三次握手](https://www.ilmiao.com/uploads/images/http2.jpg)

1. 建立连接时，客户端发送syn包（syn=j）到服务器，并进入SYN_SENT状态，等待服务器确认；SYN：同步序列编号（Synchronize Sequence Numbers）
2. 服务器收到syn包，必须确认客户的SYN（ack=j+1），同时自己也发送一个SYN包（syn=k），即SYN+ACK包，此时服务器进入SYN_RECV状态
3. 客户端收到服务器的SYN+ACK包，向服务器发送确认包ACK(ack=k+1），此包发送完毕，客户端和服务器进入ESTABLISHED（TCP连接成功）状态，完成三次握手。
#### 一些概念
* 未连接队列
服务端会为那些服务端已经收到客户端syn包且已经发送syn+ack,但尚未收到客户端确认的连接维护一个队列，该连接标记为syn_RECV状态，等收到客户端确认消息后，删除记录，修改服务端状态。
* backlog参数
表示内核为相应套接字排队的最大连接数。

### TLS
![https](https://www.ilmiao.com/uploads/images/http1.jpg)

前文说到，https在传输层tcp和应用层http中间加入了TLS。那么TSL的握手过程都做了哪些操作呢？

![TSL握手过程](https://www.ilmiao.com/uploads/images/http3.jpg)
1. Client Hello 
客户端向服务端发送Client Hello消息，消息包含了客户端生成的随机数Random1、客户端支持的加密套件（Ciphers）和SSL Version
2. Server Hello + Certificate + Server Hello Done
    * 服务端收到客户端的Hello消息后，从客户端发来的Ciphers中选择一套，加上自己生成的Random2，这时客户端和服务端有了两个随机数（生成对称秘钥）。

    * 服务端下发证书给客户端验证身份，客户端验证通过后取出证书的公钥。
    * 通知客户端Server Hello过程结束。
3. Certificate Verify + Client Key Exchange Change Cipher Spec
    * 客户端验证通过后取出证书的公钥，生成Random3,用取出的公钥加密Random3生成PreMaster Key
    * 将这个 PreMaster Key 传给服务端，服务端再用自己的私钥解出这个 PreMaster Key 得到客户端生成的 Random3。至此，客户端和服务端都拥有 Random1 + Random2 + Random3，两边再根据同样的算法就可以生成一份秘钥，握手结束后的应用层数据都是使用这个秘钥进行对称加密。
    * 客户端通知服务端后面再发送的消息都会使用前面协商出来的秘钥加密
4. Change Cipher Spec
服务端通知客户端后面再发送的消息都会使用前面协商出来的秘钥加密
5. 所有的应用层数据都会用这个秘钥加密后再通过 TCP 进行可靠传输

***抓包查看***

![client hello](https://www.ilmiao.com/uploads/images/http4.jpg)

有兴趣的可以下载[wireshark](https://www.wireshark.org/download.html)抓包看一下

### TLS握手优化
TLS的加入保证了数据的加密和数据完整性，但相比http也增加了数据传输前的等待时间，下面就有几点优化的方法：
* False Start
客户端在发送change Cipher Spec Finished的同时，发送应用数据，其实这时并没有完成握手过程。
    * 服务端必须在 Server Hello 握手中通过NPN表明自己支持的 HTTP 协议，例如：http/1.1、http/2
    * 使用支持前向安全性（Forward Secrecy）的加密算法
* ECC Certificate
减少证书的大小可选择EEC证书
* Session Resumption
    会话复用，复用之前连接计算的对称密钥，省去Client Key Exchange的过程。
    * Session Identifier
    Session Identifier（会话标识符），是 TLS 握手中生成的 Session ID。服务端可以将 Session ID 协商后的信息存起来，浏览器也可以保存 Session ID，并在后续的 ClientHello 握手中带上它，如果服务端能找到与之匹配的信息，就可以完成一次快速握手。
    缺点：
        1. 多台机器的session的同步问题
        2. session的时效问题，有效期长占用内存，有效期短起不到复用效果。
    * Session Ticket
    Session Ticket 是用只有服务端知道的安全密钥加密过的会话信息，最终保存在浏览器端。在client hello带上该信息，服务端解密成功即可复用。

### 参考资料
* [TCP三次握手](https://baike.baidu.com/item/%E4%B8%89%E6%AC%A1%E6%8F%A1%E6%89%8B/5111559)

* [TLS 握手优化详解](https://imququ.com/post/optimize-tls-handshake)

* [SSL/TLS 握手过程详解](https://www.jianshu.com/p/7158568e4867)
