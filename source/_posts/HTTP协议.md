---
title: HTTP协议
date: 2020-04-13 19:40:00
tags:
    - 网络
    - HTTP
categories:
    - 基础
    - 网络
---

## 1 URI统一资源标识符

### 1.1 什么是URI

- **URL**：Uniform Resource Locator，表示资源的位置， 期望提供查找资源的方法
- **URN**：Uniform Resource Name，期望为资源提供持久 的、位置无关的标识方式，并允许简单地将多个命名空间映射到单个URN命名 空间
  - 例如磁力链接 magnet:?xt=urn:sha1:YNCKHTQC5C 
- **URI**：Uniform Resource Identifier，用以区分资源，是 URL 和 URN 的超集，用以取代 URL 和 URN 概念



严格地说，URI 不完全等同于网址，它包含有 URL 和 URN两个部分，在 HTTP 世界里用的网址实际上是 URL——统一资源定位符（Uniform Resource Locator）。但因为URL 实在是太普及了，所以常常把这两者简单地视为相等。

### 1.2 URI的组成

![](http://base422.oss-cn-beijing.aliyuncs.com/neturi2.png)



- URI 通常由 scheme、host:port、path 和 query 四个部分组成，有的可以省略；
- scheme 叫“方案名”或者“协议名”，表示资源应该使用哪种协议来访问；
- “host:port”表示资源所在的主机名和端口号;
- path 标记资源所在的位置；
- query 表示对资源附加的额外要求；
- 在 URI 里对“**@&/”等特殊字符和汉字必须要做编码，否则服务器收到 HTTP 报文后会无法正确处理**



## 2 常见方法

- **GET**：主要的获取信息方法，大量的性能优化都针对该方法，**幂等方法**
- **HEAD**：类似 GET 方法，但服务器不发送 BODY，用以获取 HEAD 元数据，**幂等方法**
- **POST**：常用于提交 HTML FORM 表单、新增资源等
- **PUT**：更新资源，带条件时是**幂等方法** 
- **DELETE**：删除资源，**幂等方法** 
- **CONNECT**：建立 tunnel 隧道 
- **OPTIONS**：显示服务器对访问资源支持的方法，幂等方法 
- **TRACE**：回显服务器收到的请求，用于定位问题。有安全风险（**已不使用**）



## 3 HTTP Code

### 3.1 分类

RFC 标准把状态码分成了五类，用数字的第一位表示分类:

- **1××**：**请求已接收到，需要进一步处理才能完成，HTTP1.0 不支持**；
- **2××**：**成功**，报文已经收到并被正确处理；
- **3××**：**重定向**，资源位置发生变动，需要客户端重新发送请求;
- **4××**：**客户端错误**，请求报文有误，服务器无法处理;
- **5××**：**服务器错误**，服务器在处理请求时内部发生了错误

### 3.2 1xx

- **100 Continue**：**上传大文件前使用**
- **101 Switch Protocols**：**协议升级使用** ,由客户端发起请求中携带 Upgrade: 头部触发，如升级 websocket 或者 http/2.0

### 3.3 2xx

- **200 OK**:是最常见的成功状态码，表示一切正常，服务器如客户端所期望的那样返回了处理结果，如果是非 HEAD请求，通常在响应头后都会有 body 数据
- **204 No Content**:是另一个很常见的成功状态码，它的含义**与“200 OK”基本相同，但响应头后没有 body 数据**
- **206 Partial Content**：使用 **range 协议时返回部分响应内容时的响应码**，状态码 206 通常还会伴随着头字段“**Content-Range**”，表示响应报文里 body 数据的具体范围，供客户端确认，例如“Content-Range: bytes 0-99/2000”，意思是此次获取的是总计 2000 个字节的前 100 个字节。

### 3.4 3xx

- **301 Moved Permanently**：资源**永久性的重定**向到另一个 URI 中
- **302 Found**：资源**临时的重定向**到另一个 URI 
- **304 Not Modified**：当客户端拥有可能过期的缓存时，会携带缓存的标识 etag、时间等信息询问服务器缓存是否仍可复用，而304是告诉客户端可以**复用缓存**
-  **307 Temporary Redirect**：**类似302**，但明确重定向后请求方法必须**与原请求方法相同**，不得改变
- **308 Permanent Redirect**：**类似301**，但明确重定向后请求方法必须**与原请求方法相同**，不得改变

### 3.5 4xx

- **400 Bad Request**：服务器认为客户端出现了错误，但**不能明确判断为以下哪种错误**时使用此错误码。例如HTTP请求格式错误
- **401 Unauthorized**：**用户认证信息缺失或者不正确**，导致服务器无法处理请求
- **403 Forbidden**：服务器理解请求的含义，但**没有权限执行**此请求
- **404 Not Found**：服务器没有找到对应的资源

### 3.6 5xx
- **500 Internal Server Error**：服务器**内部错误**，且不属于以下错误类型
- **501 Not Implemented**：**服务器不支持实现请求所需要的功能** 
- **502 Bad Gateway**：代理服务器**无法获取到合法响应**
- **503 Service Unavailable**：服务器资源**尚未准备好处理当前请求**
- **504 Gateway Timeout**：代理服务器无法及时的**从上游获得响应**



## 4 HTTP连接流程



![](http://base422.oss-cn-beijing.aliyuncs.com/httpduring.png)

## 5 长连接与短连接

### 5.1 短连接

HTTP 协议最初（0.9/1.0）是个非常简单的协议，通信过程也采用了简单的“请求 - 应答”方式。

**它底层的数据传输基于 TCP/IP，每次发送请求前需要先与服务器建立连接，收到响应报文后会立即关闭连接。**

因为客户端与服务器的整个连接过程很短暂，不会与服务器保持长时间的连接状态，所以就被称为“短连接”（shortlived connections）。早期的 HTTP 协议也被称为是“无连接”的协议。

![](http://base422.oss-cn-beijing.aliyuncs.com/nethttpshort.png)

### 5.2 长连接

针对短连接暴露出的缺点，HTTP 协议就提出了“长连接”的通信方式，也叫“持久连接”（persistent connections）、“连接保活”（keep alive）、“连接复用”（connection reuse）。
其实解决办法也很简单，用的就是“成本均摊”的思路，**既然 TCP 的连接和关闭非常耗时间，那么就把这个时间成本由原来的一个“请求 - 应答”均摊到多个“请求 - 应答”上**。
这样虽然不能改善 TCP 的连接效率，但基于“分母效应”，每个“请求 - 应答”的无效时间就会降低不少，整体传输效率也就提高了。
这里我画了一个短连接与长连接的对比示意图

![](http://base422.oss-cn-beijing.aliyuncs.com/nethttplong.png)

### 5.3 连接相关的头字段

- **Keep-Alive**：长连接
  - 客户端请求长连接 
    	- Connection: Keep-Alive
  - 服务器表示支持长连接
  	- Connection: Keep-Alive
  -  客户端复用连接
  -  HTTP/1.1 默认支持长连接 
- **Close**：短连接
- 对代理服务器的要求
  - 不转发 Connection 列出头部，该头部仅与当前连接相关



### 5.4 Connection作用范围

**Connection 仅针对当前连接有效!**

![](http://base422.oss-cn-beijing.aliyuncs.com/netkeepalive.png)



### 5.5 桥头堵塞问题

因为 HTTP 规定报文必须是“一发一收”，这就形成了一个先进先出的“串行”队列。队列里的请求没有轻重缓急的优先级，只有入队的先后顺序，排在最前面的请求被最优先处理。
**如果队首的请求因为处理的太慢耽误了时间，那么队列里后面的所有请求也不得不跟着一起等待，结果就是其他的请求承担了不应有的时间成本**

![](http://base422.oss-cn-beijing.aliyuncs.com/httpheadblocking.png)

#### 5.5.1 性能优化

因为“请求 - 应答”模型不能变，所以“队头阻塞”问题在HTTP/1.1 里无法解决，只能缓解，有什么办法呢？

**通过HTTP 里的“并发连接”（concurrentconnections）来缓解此问题，也就是同时对一个域名发起多个长连接，用数量来解决质量的问题。**

一个客户端**最多对一个域名发起6~8个并发连接**

若果是多个域名,一共可以发起并发的数量为:**域名数乘以(6~8)**，通过多个域名提高并发的技术我们称为**域名分片**

**总结:“队头阻塞”问题会导致性能下降，可以用“并发连接”和“域名分片”技术缓解**





## 6 HTTP Range规范

### 6.1 Http Range

- 允许服务器基于客户端的请求只发送响应包体的一部分给到客户端，而客户端 自动将多个片断的包体组合成完整的体积更大的包体
  - 支持断点续传

  - 支持多线程下载

  - 支持视频播放器实时拖动
- 服务器通过 Accept-Range 头部表示是否支持 Range 请求
  - Accept-Ranges = acceptable-ranges
  - 例如:
    - Accept-Ranges: bytes：支持 
    - Accept-Ranges: none：不支持

### 6.2 Range 请求范围的单位 

基于字节，设包体总长度为 10000 

- 第 1 个 500 字节：bytes=0-499

- 第 2 个 500 字节
  - bytes=500-999 
  - bytes=500-600,601-999 •
  - bytes=500-700,601-999

- 最后 1 个 500 字节
  -  bytes=-500 
  - bytes=9500

- 仅要第 1 个和最后 1 个字节：bytes=0-0,-1 

**通过Range头部传递请求范围，如：Range: bytes=0-499**

### 6.3 Range 条件请求 

- 如果客户端已经得到了 Range 响应的一部分，并想在这部分响应未过期 的情况下，获取其他部分的响应
  - 常与 If-Unmodified-Since 或者 If-Match 头部共同使用
- If-Range = entity-tag / HTTP-date
  - 可以使用 Etag 或者 Last-Modified

### 6.4 服务器响应

- **206 Partial Content** :**正常返回**,Content-Range 头部：显示当前片断包体在完整包体中的位置
- **416 Range Not Satisfiable** :**请求范围不满足实际资源的大小**，其中 Content-Range 中的 complete- length 显示完整响应的长度，例如:Content-Range: bytes */1234
- **200 OK** :服务器**不支持 Range 请求时**，则以 200 返回完整的响应包体

### 6.5 多重范围与 multipart 

- 请求：
  - Range: bytes=0-50, 100-150
- 响应:
  - Content-Type：multipart/byteranges; boundary=…



## 7 Cookie

### 7.1 什么是Cookie

保存在客户端、由浏览器维护、表示应用状态的 HTTP 头部

- 存放在内存或者磁盘中
- 服务器端生成 Cookie 在响应中通过 Set-Cookie 头部告知客户端（允许多 个 Set-Cookie 头部传递多个值）
- 客户端得到 Cookie 后，后续请求都会 自动将 Cookie 头部携带至请求中

### 7.2 Cookie属性

- **expires-av = "Expires=" sane-cookie-date** 
  - cookie 到日期 sane-cookie-date 后失效
- **max-age-av = "Max-Age=" x**
  - cookie 经过x 秒后失效。max-age 优先级高于 expires
- **domain-av = "Domain=" domain-value**
  - 指定 cookie 可用于哪些域名，默认可以访问当前域名
- **path-av = "Path=" path-value** 
  - 指定 Path 路径下才能使用 cookie
- **secure-av = "Secure“** 
  - 只有使用 TLS/SSL 协议（https）时才能使用 cookie
- **httponly-av = "HttpOnly“** 
  - 不能使用 JavaScript（Document.cookie 、XMLHttpRequest 、Request APIs）访问到 cookie

### 7.3 Cookie 在协议设计上的问题

- Cookie 会被附加在每个 HTTP 请求中，所以**无形中增加了流量**
- 由于在 HTTP 请求中的 Cookie 是明文传递的，**所以安全性成问题**（除非用 HTTPS）
- Cookie 的大小不应超过 4KB，故对于复杂的存储需求来说是不够用的

### 7.4 Cookie的应用

- **身份识别**
- **广告追踪**



## 8 同源策略

### 8.1 定义

**所有的浏览器都遵守同源策略,服务器不管你是不是同源都返回数据！**，这个策略能够保证一个源的动态脚本不能读取或操作其他源的http响应和cookie，这就使浏览器隔离了来自不同源的内容，防止它们互相操作。所谓同源是指`协议、域名和端口`都一致的情况。

简单的来说，出于安全方面的考虑，页面中的JavaScript无法访问其他服务器上的数据，即“同源策略”。而跨域就是通过某些手段来绕过同源策略限制，实现不同服务器之间通信的效果。

### 8.2 举例

举例说明：

![](http://base422.oss-cn-beijing.aliyuncs.com/netcrof.png)

### 8.3 解决方案－CORS

如果站点 A 允许站点 B 的脚本访问其资源，必须在 HTTP 响应中显式的告知浏览器：站点 B 是被允许的 :

- 访问站点 A 的请求，浏览器应告知该请求来自站点 B

- 站点 A 的响应中，应明确哪些跨域请求是被允许的



#### 8.3.1 简单请求 

何为简单请求？ 

- GET/HEAD/POST 方法之一
- 仅能使用 CORS 安全的头部：Accept、Accept-Language、Content-Language、Content-Type 
- Content-Type 值只能是： text/plain、multipart/form-data、application/x-www-form-urlencoded 三者其中之一 

处理方式:

- 请求中携带 Origin 头部告知来自哪个域
- 响应中携带 Access-Control-Allow-Origin 头部表示允许哪些域
- 浏览器放行

![](http://base422.oss-cn-beijing.aliyuncs.com/netsimplerequest.png)



#### 8.3.2 预检请求

简单请求以外的其他请求,访问资源前需要先发起 prefilght 预检请求（方法为 OPTIONS）询问何种请求是被允许的:

- 预检请求头部
  - Access-Control-Request-Method
  - Access-Control-Request-Headers
- 预检请求响应 
  - Access-Control-Allow-Methods
  - Access-Control-Allow-Headers
  - Access-Control-Max-Age

![](http://base422.oss-cn-beijing.aliyuncs.com/netprerequest.png)

### 8.4 响应头部

- **Access-Control-Allow-Methods**
  - 在 preflight 预检请求的响应中，告知客户端后续请求允许使用的方法
- **Access-Control-Allow-Headers**
  - 在 preflight 预检请求的响应中，告知客户端后续请求允许携带的头部
- **Access-Control-Max-Age**
  - 在 preflight 预检请求的响应中，告知客户端该响应的信息可以缓存多久
- **Access-Control-Expose-Headers**
  - 告知浏览器哪些响应头部可以供客户端使用，默认情况下只有 Cache-Control、Content-Language、 Content-Type、Expires、Last-Modified、Pragma 可供使用 
- **Access-Control-Allow-Origin**
  - 告知浏览器允许哪些域访问当前资源，```*```表示允许所有域。为避免缓存错乱，响应中需要携带 Vary: Origin 
- **Access-Control-Allow-Credentials**
  - 告知浏览器是否可以将 Credentials 暴露给客户端使用，Credentials 包含 cookie、authorization 类头部、 TLS证书等

## 9 缓存机制

### 9.1 缓存首部字段

|      头字段       | 说明                                                         |   备注   |
| :---------------: | :----------------------------------------------------------- | :------: |
|    **Pragma**     | http1.0产物,控制缓存行为和http1.1中的Cache-Control功能类似   | 历史产物 |
|    **Expires**    | http1.0产物,Expires的值为服务端返回的到期时间，即下一次请求时，请求时间小于服务端返回的到期时间，直接使用缓存数据。 | 历史产物 |
| **Cache-Control** | http1.1 时期的缓存方案                                       | 主流使用 |



- 如果使用了Pragma: 'no-cache'的话，再设置Expires或者Cache-Control，就没有用了，说明Pragma的权值比后两者高。
- 如果设置了Expires之后，客户端在需要请求数据的时候，首先会对比当前系统时间和这个Expires时间，如果没有超过Expires时间，则直接读取本地磁盘中的缓存数据，不发送请求。

### 9.2 Cache-Control

#### 9.2.1 作为请求头字段

![](http://base422.oss-cn-beijing.aliyuncs.com/netcache3.png)



#### 9.2.2 作为响应字段



![](http://base422.oss-cn-beijing.aliyuncs.com/netcache4.png)



### 9.3 缓存校验字段

#### 9.3.1 Last-Modified

服务器将资源传递给客户端时，会将资源最后更改的时间以“Last-Modified: GMT”的形式加在实体首部上一起返回给客户端。

```http
Last-Modified: Fri, 22 Jul 2016 01:47:00 GMT
```

客户端会为资源标记上该信息，下次再次请求时，会把该信息附带在请求报文中一并带给服务器去做检查，若传递的时间值与服务器上该资源最终修改时间是一致的，则说明该资源没有被修改过，直接返回304状态码，内容为空，这样就节省了传输数据量 。

如果两个时间不一致，则服务器会发回该资源并返回200状态码，和第一次请求时类似。这样保证不向客户端重复发出资源，也保证当服务器有变化时，客户端能够得到最新的资源。一个304响应比一个静态资源通常小得多，这样就节省了网络带宽。

至于传递标记起来的最终修改时间的请求报文为：

**If-Modified-Since: Last-Modified-value**

示例为:

```http
If-Modified-Since: Thu, 31 Mar 2016 07:07:52 GMT
```

该请求首部告诉服务器如果客户端传来的最后修改时间与服务器上的一致，则直接回送304 和响应报头即可。
当前各浏览器均是使用的该请求首部来向服务器传递保存的 Last-Modified 值。

**Last-Modified 存在一定问题，如果在服务器上，一个资源被修改了，但其实际内容根本没发生改变，会因为Last-Modified时间匹配不上而返回了整个实体给客户端（即使客户端缓存里有个一模一样的资源**）。

#### 9.3.2 ETag

为了解决上述Last-Modified可能存在的不准确的问题，Http1.1还推出了 ETag 实体首部字段。 服务器会通过某种算法，给资源计算得出一个唯一标志符（比如md5标志），在把资源响应给客户端的时候，会在实体首部上“ETag: 唯一标识符”一起返回给客户端。例如：

```http 
Etag: "5d8c72a5edda8d6a:3239"
```

客户端会保留该 ETag 字段，并在下一次请求时将其一并带过去给服务器。服务器只需要比较客户端传来的ETag跟自己服务器上该资源的ETag是否一致，就能很好地判断资源相对客户端而言是否被修改过了。
如果服务器发现ETag匹配不上，那么直接以常规GET 200回包形式将新的资源（当然也包括了新的ETag）发给客户端；如果ETag是一致的，则直接返回304知会客户端直接使用本地缓存即可。

那么客户端是如何把标记在资源上的 ETag 传回给服务器的呢？请求报文中有两个首部字段可以带上 ETag 值：

**If-None-Match: ETag-value**
示例为:

```http
If-None-Match: "5d8c72a5edda8d6a:3239" 
```

告诉服务端如果 ETag 没匹配上需要重发资源数据，否则直接回送304 和响应报头即可。 当前各浏览器均是使用的该请求首部来向服务器传递保存的 ETag 值。

### 9.4 缓存流程图

![](http://base422.oss-cn-beijing.aliyuncs.com/netcachelc.png)




































