---
title: HTTP首部字段整理与理解
date: 2020-01-10 15:23:24
tags: [计算机网络]
---

# HTTP报文组成
## 请求报文
```
请求行
请求首部1
请求首部2
...
请求首部n
// 此处有个空行
报文主体
```

其中，请求首部由以下组成：
1. 请求首部字段
2. 通用首部字段
3. 实体首部字段

## 响应报文
```
状态行
响应首部1
响应首部2
...
响应首部n
// 此处有个空行
报文主体
```

其中，响应首部由以下组成：
1. 响应首部字段
2. 通用首部字段
3. 实体首部字段

## 首部字段
### 请求首部字段
只有在请求报文上使用的首部字段
### 响应首部字段
只有在响应报文上使用的首部字段
### 通用首部字段
请求报文和响应报文都可以拥有的首部字段
### 实体首部字段
请求报文和响应报文都可以拥有的首部字段，**用于描述报文主体的信息**

## 主要的首部字段拾遗
### 通用首部字段
#### cache-control
用于控制请求报文和响应报文的缓存信息，其中有多个字段。不同的字段用逗号分隔。

**请求指令：**

（用于请求报文首部）
* no-cache。缓存倒是缓存，但是每次请求都要强制验证资源是否过期。
* no-store。不缓存，每次请求都要求是新的响应资源。
* max-age。响应的保存的最大时间。
* max-stale。即使资源过期了，也可以接收响应。但超过的时间必须在max-stale之内。
* min-fresh。期望在指定时间内的响应仍然有效。

**响应指令：**

（用于响应报文首部）
* public。任意的服务器都可以缓存
* private。只有特定用户可以缓存
* no-cache。缓存，但每次请求都必须先验证。
* no-store。真的不缓存。
* no-transform。代理不可更改媒体类型。
* max-age。缓存的最大时间。
* s-maxage。公共缓存服务器的缓存的最大时间。

#### connection
connection可以用于控制连接状态，包括协议更改升级、连接断开等。

持久连接
```
Connection: Keep-Alive
```

升级为Websocket协议
```
Upgrade: Websocket
Connection: Upgrade
```

断开连接
```
Connection: close
```

#### Date
用于表面创建HTTP报文的时间和日期

#### Pragma
HTTP1.0遗留，主要为了向后兼容而定义，唯一形式：
```
Pragma: no-cache
```

虽然是通用首部，但只用在客户端发送的请求中，要求所有中间服务器不返回缓存的资源。但如果中间服务器以HTTP1.1为基准，那么会采用`cache-control: no-cache`指定缓存处理方式。不过为了兼容HTTP1.0的中间服务器，一般请求都会同时含有下面两个首部字段：
```
cache-control: no-cache
pragma: no-cache
```

#### Upgrade
用于通信协议的变更

```
Upgrade: Websocket
Connection: Upgrade
```

#### Via
用于追踪报文的传输路径，各个代理服务器会往Via首部添加自身的服务器信息。

### 请求首部字段
请求首部字段是描述请求报文的字段，仅存在于请求报文当中。
#### Accept
Accept指明本次请求接收的资源类型，例如当我们请求一个页面：
```
Accept: text/html,application/xhtml+xml,application/xml;q=0.9
```
表明本次请求接收`text/html`、`application/xhtml+xml`、`application/xml`类型。需要注意的是`application/xml;q=0.9`说明`application/xml`类型的权重为0.9，如果不指定权重，默认是1。如上，`text/html`、`application/xhtml+xml`的权重为1。

#### Accept-Charset
请求接收的字符码或者字符码集，同样支持权重
```
Accept-Encoding: utf-8
```

#### Accept-Encoding
接收的资源编码类型，一般来说可以有`gzip`、`compress`，或者不压缩`deflate`。同样支持权重。
```
Accept-Encoding: gzip,compress
```

#### Accept-Language
接收语言集合，同样支持权重。
```
Accept-Language: zh-CN,zh;q=0.9,en;q=0.8
```

#### Authorization
用于告知服务器，用户代理的验证信息
```
Authorization: Basic <token>
// or
Authorization: Bearer <token>
// others..
```

#### Host
Host是唯一一个必选字段，用于告知服务器具体是请求哪个域名的信息。因为现在的服务器可以拥有多台虚拟主机，必须通过Host字段告知服务器请求的具体是哪一台。
```
Host: www.baidu.com
```

#### If-modified-since
对应该资源上次返回的`last-modified`字段，如果自`last-modified`之后资源修改过，则请求新的资源，常用于缓存机制

#### If-none-match
对应该资源上次返回的`Etag`字段，如果请求的资源`Etag`不相等（说明资源修改过）则请求新资源。如果相等的话，不返回资源，而是返回状态码为304的响应报文。

#### If-match
与`If-none-match`相反，如果请求的资源`Etag`相等，则返回该资源。否则，返回状态码412，要求客户端重新请求。

#### If-Range
If-Range配合range一起使用，如果If-Range字段的值若是跟ETag或者是更新的日期时间一致，那就作为范围请求处理。
```
If-Range: <your etag>
Range: bytes=5001-10000
```

如果`If-Range`的Etag与服务器资源一致，服务器返回状态码206。
```
206 Partial Content
Content-Range: bytes 5001-10000/10000
Content-Length: 5000
```

如果不一致，则返回全部资源
```
200 OK
Etag: <new etag>
```

#### Range
请求部分资源时带上
```
Range: bytes=5001-10000
```

#### Max-Forwards
在OPTIONS和TRACE请求下，可以添加该字段表明该报文可经过的服务器最大数。当Max-Forwards为0时，则由该服务器直接返回响应。

#### Referer
表明请求方是哪个Web页面

#### User-Agent
用户代理，通常用户服务端根据用户代理判断用户是PC端还是移动端，从而返回响应版本的WEB页面。也有通过用户代理去判断是不是爬虫脚本在请求资源，从而防止被爬虫，但该防御方法早已被识破。

### 响应首部字段
响应首部字段是服务器在返回给客户端的响应报文中添加了字段。
#### Accept-Range
用于表示是否可处理请求的范围，只有两个值：`bytes`和`none`，分别代表可处理、不可处理。

#### Age
表示响应在多久之前创建的，如果对于缓存服务器，它返回报文中字段`age: 600`表明该响应是在10分钟前创建后缓存的。

#### Etag
表示资源的唯一识别码，生成规则又服务端确定。常用于校验缓存。

#### Location
当返回重定向的状态码时，如302，响应报文中添加该字段，便于用户代理跳转到新的URL。

Koa中的`ctx.redirect`就是以此实现。
```
Location: http://www.qq.com
```

#### Server
服务器HTTP程序的名称、版本等信息

#### Vary
用于控制缓存，仅对请求中含有相同的Vary字段信息的请求返回缓存。

例如，如果第一次请求：
```
Accept-Language: zh-CN
```

服务端响应报文返回：
```
Vary: Accept-Language
```

如果第二次请求头：
```
Accept-Language: zh-CN
```

则使用缓存


如果第二次请求头：
```
Accept-Language: en-us
```

则不使用缓存

### 实体首部字段
实体首部字段可存在于请求报文和响应报文当中，用于描述报文实体的内容。
#### Allow
允许的请求方法
```
Allow: GET,POST // 仅允许GET和POST请求
```

### Content-* 类型
Content-*类型的字段描述报文主体内容：
1. `encoding`：编码
2. `language`：语言码
3. `length`：长度
4. `location`：所在位置

`Content-Location`与`Location`不同，如果我们从浏览器地址栏输入的是`https://www.qq.com`，那么响应报文的`Location`就是`https://www.qq.com`，但一般来说，WEB页面都会有一个默认的HTML，一般是`index.html`，实际上我们请求的资源是`https://www.qq.com/index.html`，因此，`Content-Location`的值为`https://www.qq.com/index.html`。

5. `md5`：通过对资源进行MD5算法计算出来的值，再通过Base64编码写入。用于矫正接收的资源是否完整。浏览器会在接收完资源后进行校验，以前用来防止中间人篡改资源。但现在由于报文中`Content-Md5`也是可以篡改的，所以实际上不能防止中间人篡改。
6. `type`：返回的主体类型

### Expires
在HTTP/1.0协议中，主要通过响应报文中`Expires`字段指定一个过期时间，浏览器在请求前会检测该字段是否过期，判断是否需要请求。由于这个字段是一个绝对的时间，并且以本机时间作对比。因此，修改本机时间可以使其失效。

### Last-modified
资源的修改时间，HTTP/1.0协议中用于协商缓存，下次客户端请求时会带上该值到字段`If-modified-Since`，用于服务端判断是否需要请求新的资源。
