---
title: Http 头部字段
date: 2018-02-30 22:47:19
tags:
- 计算机网络 
- Http
categories:
- 计算机网络 
---



## HTTP 报文格式

![image](https://user-images.githubusercontent.com/16668676/44859151-2465cc80-aca6-11e8-9374-e7af115b58b1.png)





### HTTP 请求报文

在请求中，HTTP 报文由**方法、URI、HTTP 版本、HTTP 首部字段**等部分构成。

使用首部字段是为了给浏览器和服务器提供报文主体大小、所使用的语言、认证信息等内容。



![](https://ws4.sinaimg.cn/large/006tKfTcgy1fqj12dx3guj30wy0acq7h.jpg)





### HTTP 响应报文

在响应中，HTTP 报文由 HTTP 版本、状态码（数字和原因短语）、
HTTP 首部字段 3 部分构成。

![](https://ws3.sinaimg.cn/large/006tKfTcgy1fqj174ztg9j30ws08wgq2.jpg)





## HTTP 首部字段

首部和方法配合工作，共同决定了客户端和服务器能做什么事情。	

在请求和响应报文中都可以**用首部来提供信息**，有些首部是某种报文专用的，有些首部则更通用一些。可以将首部分为五个主要的类型。

### 通用首部字段

通用首部字段是指，**请求报文和响应报文双方都会使用的首部**。

| 首部字段名        | 说明                       |
| ----------------- | -------------------------- |
| **Cache-Control** | 控制缓存的行为             |
| Connection        | 逐跳首部、连接的管理       |
| Date              | 创建报文的日期时间         |
| Pragma            | 报文指令                   |
| Trailer           | 报文末端的首部一览         |
| Transfer-Encoding | 指定报文主体的传输编码方式 |
| **Upgrade**       | 升级为其他协议             |
| Via               | 代理服务器的相关信息       |
| Warning           | 错误通知                   |

#### Cache-Control

通过指定首部字段 Cache-Control 的指令，就能操作缓存的工作机制。

`Cache-Control: private, max-age=0, no-cache`

指令的参数是可选的，多个指令之间通过“,”分隔。首部字段 CacheControl 的指令可用于请求及响应时。



##### 缓存请求指令

| 指令                | 参数   | 说明                         |
| ------------------- | ------ | ---------------------------- |
| no-cache            | 无     | 强制向源服务器再次验证       |
| no-store            | 无     | 不缓存请求或响应的任何内容   |
| max-age = [ 秒]     | 必需   | 响应的最大Age值              |
| max-stale( = [ 秒]) | 可省略 | 接收已过期的响应             |
| min-fresh = [ 秒]   | 必需   | 期望在指定时间内的响应仍有效 |
| no-transform        | 无     | 代理不可更改媒体类型         |
| only-if-cached      | 无     | 从缓存获取资源               |
| cache-extension     | -      | 新指令标记（token）          |

换言之，无参数值的首部字段可以使用缓存。



> 从字面意思上很容易把 no-cache 误解成为不缓存，但事实上 no-cache 代表**不缓存*过期*的资源**，缓存会向源服务器进行有效期确认后处理资源，也许称为 `do-notserve-from-cache-without-revalidation`更合适。**`no-store `才是真正地不进行缓存**，

##### max-age

使用 HTTP/1.1 版本的缓存服务器遇到同时存在 Expires 首部字段的情况时，会**优先处理 max-age 指令**，而**忽略掉 Expires 首部字**段。
而HTTP/1.0 版本的缓存服务器的情况却相反，max-age 指令会被忽略掉

##### only-if-cached 指令

`Cache-Control: only-if-cached`
使用 only-if-cached 指令表示客户端仅在缓存服务器本地缓存目标资源的情况下才会要求其返回。换言之，该指令**要求缓存服务器不重新加载响应，也不会再次确认资源有效性**。若发生请求缓存服务器的本地缓存无响应，则返回状态码 504 Gateway Timeout。



```
 This field's name "only-if-cached" is misleading. It actually means "do not use the network". It is set by a client who only wants to make a request if it can be fully satisfied by the cache. Cached responses that would require validation (ie. conditional gets) are not permitted if this header is set.
 
 该字段名有误导性。实际意义是「不要使用网络」。它由一个只有在缓存能够完全满足需求的情况下才想发出请求的客户端。如果设置了该 header，则不允许使用需要验证的缓存响应（ 比如。条件 get 请求）。
```



### 请求首部字段



| 首部字段名            | 说明                                          |
| --------------------- | --------------------------------------------- |
| Accept                | 用户代理可处理的媒体类型                      |
| Accept-Charset        | 优先的字符集                                  |
| Accept-Encoding       | 优先的内容编码                                |
| Accept-Language       | 优先的语言（自然语言）                        |
| Authorization         | Web认证信息                                   |
| Expect                | 期待服务器的特定行为                          |
| From                  | 用户的电子邮箱地址                            |
| Host                  | 请求资源所在服务器                            |
| If-Match              | 比较实体标记（ETag）                          |
| **If-Modified-Since** | 比较资源的更新时间                            |
| If-None-Match         | 比较实体标记（与 If-Match 相反）              |
| If-Range              | 资源未更新时发送实体 Byte 的范围请求          |
| If-Unmodified-Since   | 比较资源的更新时间（与If-Modified-Since相反） |
| Max-Forwards          | 最大传输逐跳数                                |
| Proxy-Authorization   | 代理服务器要求客户端的认证信息                |
| Range                 | 实体的字节范围请求                            |
| Referer               | 对请求中 URI 的原始获取方                     |
| TE                    | 传输编码的优先级                              |
| User-Agent HTTP       | 客户端程序的信息                              |

### 响应首部字段

| 首部字段名         | 说明                         |
| ------------------ | ---------------------------- |
| Accept-Ranges      | 是否接受字节范围请求         |
| Age                | 推算资源创建经过时间         |
| ETag               | 资源的匹配信息               |
| Location           | 令客户端重定向至指定URI      |
| Proxy-Authenticate | 代理服务器对客户端的认证信息 |
| Retry-After        | 对再次发起请求的时机要求     |
| Server             | HTTP服务器的安装信息         |
| Vary               | 代理服务器缓存的管理信息     |
| WWW-Authenticate   | 服务器对客户端的认证信息     |



### vary

- 首部字段 Vary 可对缓存进行控制。源服务器会向代理服务器传达关于本地缓存使用方法的命令。
- 从代理服务器接收到源服务器返回包含 Vary 指定项的响应之后，若再要进行缓存，仅对请求中含有相同 Vary 指定首部字段的请求返回缓存。即使对相同资源发起请求，但由于 Vary 指定的首部字段不相同，因此必须要从源服务器重新获取资源。



### 实体首部字段

实体首部字段是包含在请求报文和响应报文中的**实体部分所使用的首部**，用于补充内容的更新时间等与实体相关的信息。

有很多首部可以用来**描述 HTTP 报文的负荷**。

实体首部提供了**有关实体及其内容的大量信息**，从有关**对象类型的信息**，到能够**对资源使用的各种有效的请求方法**。总之，**实体首部可以告知报文的接收者它在对什么进行处理**。下表中 列出了实体的信息性首部。



| 首部字段名         | 说明                             |
| ------------------ | -------------------------------- |
| Allow              | 资源**可支持的HTTP方法**         |
| Content-Encoding   | 实体主体适用的编码方式           |
| Content-Language   | 实体主体的自然语言               |
| **Content-Length** | **实体主体的大小**（单位：字节） |
| Content-Location   | 替代对应资源的URI                |
| **Content-MD5**    | 实体主体的报文摘要               |
| **Content-Range**  | 实体主体的位置范围               |
| **Content-Type**   | 实体主体的**媒体类型**           |
| **Expires**        | 实体主体**过期的日期时间**       |
| **Last-Modified**  | 资源的**最后修改日期时间**       |

### 扩展首部

扩展首部是非标准的首部，由应用程序开发者创建，但还未添加到已批准的 HTTP 规范中去。即使不知道这些扩展首部的含义，HTTP 程序也要接受它们并对其进行转发。





## 参考资料

- 《图解 HTTP》
- 《HTTP 权威指南》



由于本人水平有限，可能出于误解或者笔误难免出错，如果发现有问题或者对文中内容存在疑问请在下面评论区告诉我，谢谢！



