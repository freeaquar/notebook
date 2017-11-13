## Requests - HTTP 认证模块

---

Author: freeaquar

Created at: 2017/09/06

Updated at: 2017/09/06

--------------

### 1. Requests 是什么

用官方说法是这么说的

> **Requests** is the only *Non-GMO* HTTP library for Python, safe for human consumption.
>
> **Requests** 是 Python 唯一的非转基因 HTTP 库, 供人安全食用

很俏皮的一句描述, 看得出来开发者是一个比较有意思的人, 简而言之就是 **Requests** 库对底层行为进行封装和抽象, 让我们可以更简单的方式进行 `HTTP` 相关的操作, 更有意思的是, 文档里还有一个像模像样的警告:

> **Warning:** Recreational use of the Python standard library for HTTP may result in dangerous side-effects, including: security vulnerabilities, verbose code, reinventing the wheel, constantly reading documentation, depression, headaches, or even death.
>
> 警告: 使用 Python 的 HTTP 标准库可能导致危险的结果, 包括: 安全漏洞, 冗长的代码, 重复制造轮子, 不断的阅读文档, 沮丧, 头疼, 甚至死亡

那到底有多简单呢? 这里使用官方的例子

```python
>>> r = requests.get('https://api.github.com/user', auth=('user', 'pass'))
>>> r.status_code
200
>>> r.headers['content-type']
'application/json; charset=utf8'
>>> r.encoding
'utf-8'
>>> r.text
u'{"type":"User"...'
>>> r.json()
{u'disk_usage': 368627, u'private_gists': 484, ...}
```

至此, 仅仅一行代码, 你已经完成了一次 `HTTP` 请求, 还是基于 `基本认证` 的.

了不起...不过 `HTTP` 我懂, 但这个 `基本认证` 是什么呢

别着急, 本文会对 `HTTP` 协议中认证方式做一个比较详细的介绍, 并对 **Requests** 里的 `auth.py` 模块源码进行剖析



### 2. 本次 Requests 阅读内容

本次所读源码为 `v2.18.4` 版本, 我们阅读分析 **Requests** 的 `HTTP ` 认证相关模块, 阅读分析分为以下几个步骤

1. `HTTP` 相关知识介绍
2. `Requests` 认证相关使用
3. 认证模块 `auth.py` 详细解读


相较于 `HTTP` 协议的知识, 源码并不算的多. 现在, 我们从第一部分开始讲起, 如果你已经具备了相应的知识, 可以直接阅读下一小节, 如果这三点你都了如指掌, 那么......欢迎指正



### 3. HTTP 相关知识介绍

想要比较深入的理解 **Requests** 认证相关的源码, 需要提前了解一部分 `HTTP` 协议认证相关的知识

> 参考:
>
> rfc: https://tools.ietf.org/html/rfc7235
>
> mozilla: https://developer.mozilla.org/en-US/docs/Web/HTTP/Authentication

#### 3.1 预备知识

简单来说, `HTTP` 协议有两种认证方式, 基本认证(Basic Authentication)和升级版的摘要认证(Digest Authentication). 而相关的 `HTTP` 状态码主要有 401: 未授权 和 407: 要求代理身份认证

接下来我们一起看看交互流程

##### 3.1.1 基本的 HTTP 认证框架

下图是一次完整的 `HTTP` 基本认证交互流程

如果第一次请求的时候可以直接带上 `Authorization` 包头, 可以省去下图中的第一次请求, 摘要认证流程和基本认证流程相同, 只是实现细节上稍许不同, 具体后文会详述

![The general HTTP authentication framework](https://mdn.mozillademos.org/files/14689/HTTPAuth.png)



##### 3.1.2 认证相关包头描述

认证相关的包头总共有四种, 分别对应 401 未授权时候服务端/客户端的包头, 和 407 要求代理身份认证时候服务端/客户端的包头

401/407 状态码对服务端/客户端来说, 认证的过程是完全相同的, 唯一的区别就是使用不同的包头(请求头 /返回头), 接下来我们看看服务端/客户端具体使用的包头

- 两个单词释义

  > authentication: n. 认证; 鉴定
  >
  > authenticate: v. 证明...是真的; 证实
  >
  > 如果验证失败, 服务端会返回头 XXX-Authenticate, 意思是要求客户证明自己是合法的, 而客户端再次请求的时候带上认证信息: Authorization/Proxy-Authorization


- [Authorization](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Authorization)

  > Authorization: <type> <credentials>
  >
  > 客户端请求头: 包含认证信息(credentials)来向服务端证明自己的身份
  >
  > 通常是在服务端返回 401 未授权(Unauthorized)状态和 WWW-Authenticate 头之后

- [WWW-Authenticate](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/WWW-Authenticate)

  > WWW-Authenticate: <type> realm=<realm>
  >
  > 服务端响应头: 定义获取资源的认证(authentication)方法

- [Proxy-Authorization](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Proxy-Authorization)

  > Proxy-Authorization: <type> <credentials>
  >
  > 客户端请求头: 包含认证信息(credentials)来向代理服务端证明自己的身份
  >
  > 通常是在服务端返回 407 代理服务器授权需要(Proxy Authentication Required)状态和 Proxy-Authenticate 头之后

- [Proxy-Authenticate](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Proxy-Authenticate)

  > Proxy-Authenticate: <type> realm=<realm>
  >
  > 服务端响应头: 定义获取资源的认证(authentication)方法, 被用于获取代理服务器之后的资源

##### 3.1.3 攻击手段

当我们讨论分析一个认证方法的时候, 很重要的一点就是, 已知的攻击手段是否能构成威胁

下文列出了几种后文会用到的攻击手段, 可以适当理解下, 方便后文阅读, 如果有更多兴趣, 可以点击链接进一步学习

- [暴力破解法(穷举攻击)](https://zh.wikipedia.org/wiki/%E6%9A%B4%E5%8A%9B%E7%A0%B4%E8%A7%A3%E6%B3%95)

  > 即将密码进行逐个推算直到找出真正的密码为止, 辅以字典来缩小密码组合的范围
  >
  > 从密码学角度考虑，不认为穷举法是有效的破解方法

- [选择明文攻击](https://zh.wikipedia.org/wiki/%E9%80%89%E6%8B%A9%E6%98%8E%E6%96%87%E6%94%BB%E5%87%BB)

  > 攻击者拥有加密机的访问权限，可构造任意明文所对应的密文, 以此来得到一些有利的信息
  >
  > **中途岛海战**: 一次成功的选择明文攻击 -- 美军故意透露出假情报(明文) 来诱使日军发报(密文), 从而得知“AF”指的是中途岛而非阿留申群岛


- [彩虹表](https://zh.wikipedia.org/wiki/%E5%BD%A9%E8%99%B9%E8%A1%A8)

  > 预计算的哈希链集, 更多: https://www.zhihu.com/question/19790488
  >

- [重放攻击](https://zh.wikipedia.org/wiki/%E9%87%8D%E6%94%BE%E6%94%BB%E5%87%BB)

  > 是一种网络攻击，它恶意的欺诈性的重复或拖延正常的数据传输

- [中间人攻击](https://zh.wikipedia.org/wiki/%E4%B8%AD%E9%97%B4%E4%BA%BA%E6%94%BB%E5%87%BB)

  > 指攻击者与通讯的两端分别创建独立的联系，并交换其所收到的数据，使通讯的两端认为他们正在通过一个私密的连接与对方直接对话，但事实上整个会话都被攻击者完全控制


##### 3.1.4 字段描述

后文描述认证相关知识时候, 会用到一些术语, 此处对一些术语做了一些描述

```
realm 	- 认证域 - 简单来说, 就是
qop 	- 保护质量
nonce 	- 服务器密码随机数
cnonce 	- 客户端密码随机数
nc 		- 请求计数
challenge - 质疑, 挑战
```

以上比较难懂的应该是 realm/qop

realm: 所有的身份验证方案(authentication schemes) , 每次服务端对客户端 challenge 的时候必须要带上, 其对应的值是一个字符串, 一般由源服务器生成. 相同  realm 的资源可以被同一个用户名/密码访问

> The realm directive (case-insensitive) is required for all
>    authentication schemes that issue a challenge. The realm value
>    (case-sensitive), in combination with the canonical root URL (the
>    absoluteURI for the server whose abs_path is empty; see section 5.1.2
>    of [2]) of the server being accessed, defines the protection space.
>    These realms allow the protected resources on a server to be
>    partitioned into a set of protection spaces, each with its own
>    authentication scheme and/or authorization database. The realm value
>    is a string, generally assigned by the origin server, which may have
>    additional semantics specific to the authentication scheme. Note that
>    there may be multiple challenges with the same auth-scheme but
>    different realms.

qop: 影响请求时候的摘要方式, 如果客户端展示该字段, 则必须是服务端 `WWW-Authenticate` 值中的一个

> Indicates what "quality of protection" the client has applied to
> the message. If present, its value MUST be one of the alternatives
> the server indicated it supports in the WWW-Authenticate header.
> These values affect the computation of the request-digest. Note
> that this is a single token, not a quoted list of alternatives as
> in WWW- Authenticate.  This directive is optional in order to
> preserve backward compatibility with a minimal implementation of
> RFC 2069 [6], but SHOULD be used if the server indicated that qop
> is supported by providing a qop directive in the WWW-Authenticate
> header field.



好了, 至此我们已经掌握了基本的预备知识点和相关术语, 接下来, 我们开始详细讲解各种 `HTTP` 认证步骤. 由于 401 与 407 状态码除了使用的包头不同, 认证方式完全相同, 我们主要以 401 为例进行深入讲解



#### 3.2 401 - 未授权(Unauthorized)

> 401 语义即 "未授权", 即用户没有必要的凭据, 如果当前请求已经包含了 `Authorization` ，那么 401 响应代表着服务器验证已经拒绝了那些证书
> 认证方式: `HTTP 基本认证`, `HTTP 摘要认证`

##### 3.2.1 [HTTP 基本认证](https://zh.wikipedia.org/wiki/HTTP%E5%9F%BA%E6%9C%AC%E8%AE%A4%E8%AF%81)
> 是一种用来允许客户端请求时候, 带上 `用户名`, `口令` 形式的身份凭据的一种验证方式
> 因为安全原因, 很少在可公开访问的互联网网站上使用, 后来的机制 `HTTP 摘要认` 证是为替代基本认证而开发, 允许密钥以相对安全的方式在不安全的通道上传输
> 最初定义: `HTTP 1.0` 规范([RFC 1945](https://tools.ietf.org/html/rfc1945))

###### 优点
- 基本所有流行网页浏览器支持认证

###### 缺点
- 明文传输的密钥和口令很容易被拦截
- `HTTP` 没有为服务器提供一种方法指示客户端丢弃这些被缓存的密钥, 并没有一种有效的方法来让用户登出

###### 样例 -- 认证信息

- 用户名是 `Aladdin`, 口令是 `open sesame`
- 则拼接后的结果就是 `Aladdin:open sesame`
- 然后再将其用 `Base64` 编码(目的是转换为与 `HTTP` 协议兼容的字符集), 得到 `QWxhZGRpbjpvcGVuIHNlc2FtZQ==`

###### 样例 -- 认证过程

1. 客户端请求(没有认证信息):
```
GET /private/index.html HTTP/1.0
Host: localhost
```

2. 服务端应答:
  一个服务端的 401 返回头必须包含 `WWW-Authenticate`字段
  格式: `WWW-Authenticate: <type> realm=<realm>`
```
HTTP/1.0 401 Authorization Required
Server: HTTPd/1.0
Date: Sat, 27 Nov 2004 10:18:15 GMT
WWW-Authenticate: Basic realm="Secure Area"
Content-Type: text/html
Content-Length: 311

<!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 4.01 Transitional//EN"
 "http://www.w3.org/TR/1999/REC-html401-19991224/loose.dtd">
<HTML>
  <HEAD>
    <TITLE>Error</TITLE>
    <META HTTP-EQUIV="Content-Type" CONTENT="text/html; charset=ISO-8859-1">
  </HEAD>
  <BODY><H1>401 Unauthorized.</H1></BODY>
</HTML>
```

3. 客户端的请求
  用户名: "Aladdin”, 口令: "open sesame", 结果: "Aladdin:open sesame", 再将其用 `Base64` 编码, 得到 "QWxhZGRpbjpvcGVuIHNlc2FtZQ=="：
```
GET /private/index.html HTTP/1.0
Host: localhost
Authorization: Basic QWxhZGRpbjpvcGVuIHNlc2FtZQ==
```

4. 服务端的应答：
```
HTTP/1.0 200 OK
Server: HTTPd/1.0
Date: Sat, 27 Nov 2004 10:19:07 GMT
Content-Type: text/html
Content-Length: 10476
```
(跟随一个空行，随后是需凭据页的HTML文本)

##### 3.2.2 [HTTP 摘要认证](https://zh.wikipedia.org/wiki/HTTP%E6%91%98%E8%A6%81%E8%AE%A4%E8%AF%81)
> 摘要访问认证是一种协议规定的 `Web` 服务器用来同网页浏览器进行认证信息协商的方法。它在密码发出前，先对其应用哈希函数，这相对于 `HTTP 基本认证` 发送明文而言，更安全。
> 从技术上讲，摘要认证是使用随机数来阻止进行密码分析的 `MD5` 加密哈希函数应用
>
> 摘要访问认证最初由 `RFC 2069`  (HTTP的一个扩展：摘要访问认证)中被定义, 随后被`RFC 2617`  (HTTP 认证：基本及摘要访问认证), 具体区别见后文

###### 优点

- 密码并非直接在摘要中使用, 而是 HA1 = MD5(username:realm:password)
- 引入客户端随机数(cnonce), 能防止`选择明文攻击`, eg: `彩虹表`
- 服务端随机数(nonce)允许包含时间戳, 防止重放攻击

###### 缺点

- 有意成为安全的折中, 它没有被设计为替换强认证协议
- `RFC 2617` 中的许多安全选项都是可选的。如果服务器没有指定保护质量(qop)，客户端将以降低安全性的早期的 `RFC 2069 ` 的模式操作
- 容易受到`中间人攻击`

###### 生成 response 方法

`RFC 2069`  标准(HTTP的一个扩展: 摘要访问认证)

```
HA1 = MD5(A1) = MD5(username:realm:password)
HA2 = MD5(A2) = MD5(method:digestURI)
response = MD5(HA1:nonce:HA2)
```

`RFC 2617`  标准(HTTP 认证: 基本及摘要访问认证)

> 引入了一系列安全增强的选项: 保护质量, 随机数计数器由客户端增加, 客户端生成随机数
>
> 为了防止: [选择明文攻击](https://zh.wikipedia.org/wiki/%E9%80%89%E6%8B%A9%E6%98%8E%E6%96%87%E6%94%BB%E5%87%BB)的[密码分析](https://zh.wikipedia.org/wiki/%E5%AF%86%E7%A0%81%E5%88%86%E6%9E%90)
>
> qop 未指定情况下, 遵循 `RFC 2069`

```
HA1 = MD5(A1) = MD5(username:realm:password)

如果 qop 值为 "auth" 或 未指定
HA2 = MD5(A2) = MD5(method:digestURI)
如果 qop 值为 "auth-int"
HA2 = MD5(A2) = MD5(method:digestURI:MD5(entityBody))

如果 qop 值为 "auth" 或 "auth-int", 那么如下计算 response
response = MD5(HA1:nonce:nc:cnonce:qop:HA2)
如果 qop 未指定
response = MD5(HA1:nonce:HA2)
```

###### 样例 -- response 生成

```
HA1 = MD5( "Mufasa:testrealm@host.com:Circle Of Life" )
       = 939e7578ed9e3c518a452acee763bce9

HA2 = MD5( "GET:/dir/index.html" )
    = 39aff3a2bab6126f332b942af96d3366

Response = MD5( "939e7578ed9e3c518a452acee763bce9:\
                 dcd98b7102dd2f0e8b11d0f600bfb0c093:\
                 00000001:0a4f113b:auth:\
                 39aff3a2bab6126f332b942af96d3366" )
         = 6629fae49393a05397450978507c4ef1
```

###### 样例 -- 认证过程

1. 客户端请求 (无认证)

```
GET /dir/index.html HTTP/1.0
Host: localhost
```
(跟随一个新行，形式为一个回车再跟一个换行)

2. 服务器响应

```
HTTP/1.0 401 Unauthorized
Server: HTTPd/0.9
Date: Sun, 10 Apr 2005 20:26:47 GMT
WWW-Authenticate: Digest realm="testrealm@host.com",
                        qop="auth,auth-int",
                        nonce="dcd98b7102dd2f0e8b11d0f600bfb0c093",
                        opaque="5ccc069c403ebaf9f0171e9517f40e41"
Content-Type: text/html
Content-Length: 311

<!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 4.01 Transitional//EN"
 "http://www.w3.org/TR/1999/REC-html401-19991224/loose.dtd ">
<HTML>
  <HEAD>
    <TITLE>Error</TITLE>
    <META HTTP-EQUIV="Content-Type" CONTENT="text/html; charset=ISO-8859-1">
  </HEAD>
  <BODY><H1>401 Unauthorized.</H1></BODY>
</HTML>
```

3. 客户端请求 (用户名 "Mufasa", 密码 "Circle Of Life")

```
GET /dir/index.html HTTP/1.0
Host: localhost
Authorization: Digest username="Mufasa",
                     realm="testrealm@host.com",
                     nonce="dcd98b7102dd2f0e8b11d0f600bfb0c093",
                     uri="/dir/index.html",
                     qop=auth,
                     nc=00000001,
                     cnonce="0a4f113b",
                     response="6629fae49393a05397450978507c4ef1",
                     opaque="5ccc069c403ebaf9f0171e9517f40e41"
```
(跟随一个新行，形式如前所述)

4. 服务器响应

```
HTTP/1.0 200 OK
Server: HTTPd/0.9
Date: Sun, 10 Apr 2005 20:27:03 GMT
Content-Type: text/html
Content-Length: 7984
```
(随后是一个空行，然后是所请求受限制的HTML页面).

###### 注意

- 后续交互

> 服务器仅在每次 "401" 响应后发行新的nonce
>
> 验证通过后, 客户端可以提交一个新的请求, 重复使用服务器密码随机数(nonce), 但是提供新的客户端密码随机数(cnonce)
>
> 在后续的请求中, 十六进制请求计数器(nc)必须比前一次使用的时候要大, 否则攻击者可以简单的使用同样的认证信息重放老的请求 -- 服务端确保每个发出去的 nonce 计数器是增加的
>
> 改变 改变 HTTP 方法和/或计数器数值都会导致不同的 `response` 值

- 过期
> 服务器应当记住最近所生成的服务器密码随机数nonce的值, 也可以在发行每一个密码随机数nonce后, 记住过一段时间让它们过期
>
> 如果客户端使用了一个过期的值, 服务器应该响应 "401" 状态码，并且在认证头中添加 `stale=TRUE`，表明客户端应当使用新提供的服务器密码随机数 `nonce` 重发请求, 而不必提示用户其它用户名和口令
>
>
>
> 在生成后立刻过期服务器密码随机数nonce是不行的, 因为客户端将没有任何机会来使用这个nonce

- SIP

> SIP: 会话发起协议(Session Initiation Protocol, 缩写 SIP )是一个由 IETF MMUSIC 工作组 开发的协议, 作为标准被提议用于创建, 修改和终止包括视频, 语音, 即时通信, 在线游戏和虚拟现实等多种多媒体元素在内的交互式用户会话
>
> SIP 基本上使用了同样的摘要认证算法. 它由 `RFC 3261`  定义



#### 3.3 407 - 要求代理身份验证 (Proxy authentication required)

> 与401响应类似, 只不过客户端必须在代理服务器上进行身份验证
>
> 代理服务器必须返回一个Proxy-Authenticate用以进行身份询问. 客户端可以返回一个 `Proxy-Authorization`信息头用以验证



现在, 我们已经比较深入的了解了 `HTTP` 的基本认证和摘要认证, 可以看到, 摘要认证的整个过程还是比较繁复的, 那我们如果使用 **Requests** 需要怎么做呢



### 4. Requests 认证相关使用

不得不说, **Requests** 确实是为人类提供的 `HTTP` 库, 上面那纷繁的认证过程, 你只需要寥寥几行就可以实现

> 文档: http://docs.python-requests.org/en/master/user/authentication/?highlight=auth

#### 4.1 Basic Authentication

许多 web 服务器需要使用 HTTP 的 Basic Authentication, 而通过 **Requests** 非常容易实现只需要:

```python
>>> from requests.auth import HTTPBasicAuth
>>> requests.get('https://api.github.com/user', auth=HTTPBasicAuth('user', 'pass'))
<Response [200]>
```

事实上, HTTP 的 Basic Authentication 非常常见, 所以 **Requests** 提供了一个方便的简写形式

```python
>>> requests.get('https://api.github.com/user', auth=('user', 'pass'))
<Response [200]>
```



#### 4.2 Digest Authentication

另一个非常流行的 `HTTP` 认证方式是 Digest Authentication, 而通过 **Requests** 非常容易实现, 只需要:

```python
>>> from requests.auth import HTTPDigestAuth
>>> url = 'http://httpbin.org/digest-auth/auth/user/pass'
>>> requests.get(url, auth=HTTPDigestAuth('user', 'pass'))
<Response [200]>
```



没了?

没了! 就这么简单!

和伪代码一样的简单, 可以说, 没学过 `Python` 的人也能大概看懂意思, 那 **Requests** 是如何实现的呢, 请看下一小节源码解读

### 5. auth.py 模块详细解读

所有 `HTTP` 认证相关的源码都在 **Requests** 的 `auth.py` 模块中, 逻辑比较简单并且代码并不算多, 所以读起来还是比较轻松的

#### 5.1 模块包含
- - `_basic_auth_str`
- 类型
    - `AuthBase` -- 基类
    - `HTTPBasicAuth` -- HTTP 基本认证
    - `HTTPProxyAuth` -- HTTP 代理身份基本认证
    - `HTTPDigestAuth` -- HTTP 摘要认证

#### 5.2 详解
##### _basic_auth_str
方法用于将 用户名/密码 整理成 `HTTP 基本认证` 的格式

用户名/密码转换为 `latin1` 格式编码

注意: 传入的用户名/密码, 如果是非 `string `类型的, 在 3.0.0 后将不被支持



##### AuthBase(object)

所有认证类型的基类, 继承该类的类, 都需要自行实现 `__call__` 方法, 否则调用的时候会抛出 `NotImplementedError` 异常

```python
class AuthBase(object):
    """Base class that all auth implementations derive from"""

    def __call__(self, r):
        raise NotImplementedError('Auth hooks must be callable.')
```



##### HTTPBasicAuth(AuthBase)

HTTP 基本认证, 由上文知道, 需要将用户名密码按一定规则生成凭证(`_basic_auth_str` 方法 ), 并作为请求头 `Authorization` 字段的值

同时实现了实例的相等/不等的判断 -- 认为用户名/密码相同的实例相同(因为最终生成的凭证是相同的)

```python
def __eq__(self, other):
    return all([
        self.username == getattr(other, 'username', None),
        self.password == getattr(other, 'password', None)
    ])

    def __ne__(self, other):
        return not self == other
```



##### HTTPProxyAuth(HTTPBasicAuth)

通过前文我们可以知道, 这个认证本质上和 `HTTPBasicAuth` 一样, 区别仅仅是使用的不同状态码和包头字段

所以, 该对象继承自 `HTTPBasicAuth`, 只修改 `__call__` 方法, 将生成的证书放到请求包头的 `Proxy-Authorization` 字段中



##### HTTPDigestAuth(AuthBase)

与 `HTTPBasicAuth` 一样实现了实例的相等判断

注册了两个 `response` 的回调方法

- 重定向
- 401 状态码

关于注册回调方法, `requests` 只允许以下几个时机添加回调方法, 每个回调时机使用列表存储多个回调方法
```python
# args: 参数传递到 Request 对象之前
# pre_request: Request 对象发送之前
# post_request: Request 对象发送之后
# response: Request 对象的返回包体

HOOKS = ('args', 'pre_request', 'post_request', 'response')
```
因此, 这两个回调方法是在收到返回之后执行的

- 如果返回码是 401: 按照 `HTTP 摘要认证` 方法生成认证信息并添加到请求头 `Authorization` 字段里, 重新发起请求, 将返回包体输出
- 如果是重定向: 清零重定向之后的 401 的计数器



摘要认证会稍微复杂点, 我们先来看看实例化的时候做了什么: 每个实例都使用单独的线程空间:  `_thread_local`

```python
def __init__(self, username, password):
    self.username = username
    self.password = password
    # Keep state in per-thread local storage
    self._thread_local = threading.local()
```
然后, 我们看看调用 `HTTPDigestAuth ` 实例的时候是做了什么

```python
def __call__(self, r):
    # Initialize per-thread state, if needed
    # 初始化当前线程状态(为了保证数据的线程安全)
    self.init_per_thread_state()
    # If we have a saved nonce, skip the 401
    # 如果之前已经记录过服务端 401 时候返回的 `nonce` 信息, 直接生成 `Authorization ` 证书
    # 否则再次通过 401 获取
    if self._thread_local.last_nonce:
        r.headers['Authorization'] = self.build_digest_header(r.method, r.url)
    try:
        # 获取上一次的 `file-like` 对象的指针位置
        # 连接复用时候可以知道上次请求状态
        self._thread_local.pos = r.body.tell()
    except AttributeError:
        # In the case of HTTPDigestAuth being reused and the body of
        # the previous request was a file-like object, pos has the
        # file position of the previous body. Ensure it's set to
        # None.
        self._thread_local.pos = None
    # 注册回调方法 401/redirect
    r.register_hook('response', self.handle_401)
    r.register_hook('response', self.handle_redirect)
    self._thread_local.num_401_calls = 1

    return r
```

如果当前实例已经被初始化了, 不再处理, 否则对县城内变量初始化, 加这个判断的原因, 我认为认为是尽可能多的复用服务端的 `nonce`, 减少服务端不必要的质疑回复

```python
def init_per_thread_state(self):
    # Ensure state is initialized just once per-thread
    if not hasattr(self._thread_local, 'init'):
        self._thread_local.init = True        	# 是否初始化过
        self._thread_local.last_nonce = ''      # 最近一次服务端返回的 `nonce` 参数
        self._thread_local.nonce_count = 0      # `nonce` 计数器
        self._thread_local.chal = {}            # 服务端返回头里的一些认证相关信息. eg: qop...
        self._thread_local.pos = None           # 前一次请求的文件类型的指针位置
        self._thread_local.num_401_calls = None # 服务端返回 401 状态码的计数, 避免无限 401 循环
```

如果触发 401 回调方法, 后文的 `r` 代表返请求返回的对象

1. 如果 `pos` 不为空, 将原始 `request` 指向上次的位置, 将上次未完成的请求完成

   ```python
   if self._thread_local.pos is not None:
       r.request.body.seek(self._thread_local.pos)
   ```

2. 如果满足摘要认证的条件, 会需要生成凭证, 然后在请求, 并将返回的 `Response` 返回, 否则返回之前的 `Response`

   由于涉及到链接复用, 所以需要读取完当前连接里的内容然后释放. 因为如果连接中包含上一次请求的返回数据, 被其他请求复用的时候, 就无法知道返回数据是属于谁的了

   ```python
   # Consume content and release the original connection
   # to allow our new request to reuse the same one.
   r.content
   r.close()
   ```

3. 此处为 2 满足条件时候, 重新请求的代码, 具体实现是先拷贝原始 `request` 对象, 并为之添加 `cookies` 和 `Authorization`, 发送请求, 将原始 `request` 存入历史, 并将本次返回的 `Response` 返回到外层, 并将其交给下一个回调函数

   ```python
   # 将之前请求的 requests 对象加上这次返回的 cookies, 重新构造 requests
   prep = r.request.copy()
   extract_cookies_to_jar(prep._cookies, r.request, r.raw)
   prep.prepare_cookies(prep._cookies)

   # 在请求头上加上认证信息并重新请求
   prep.headers['Authorization'] = self.build_digest_header(
       prep.method, prep.url)
   _r = r.connection.send(prep, **kwargs)
   _r.history.append(r)
   _r.request = prep

   return _r
   ```

总结起来就是, 外层方法只需要正常去调用该实例, 线程安全, 重定向, 需要认证重新请求这些必要做的事情都已经被处理了, 而且对上层调用者透明.

更好的是, 摘要认证请求时候的一些优化(复用链接, 记录之前服务端的 `nonce`) 也已经实现, 上层调用者可以忽略
