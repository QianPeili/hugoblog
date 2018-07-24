---
title: "一次CORS请求经历"
date: 2018-07-23T20:44:39+08:00
categories: 
 - note
tags: 
 - http
---


客户端写了一个基于WebGL的网页游戏，测试阶段涉及到跨域请求，按照之前的经验，我简单地给服务端的Response添加了一个头信息：`Access-Control-Allow-Origin: *`。

有个接口直接自己测试OK，但是客户端侧却收到了`404`的状态码，看日志发现原本应该是`POST`方法的请求变成了`OPTIONS`，并且原本发送的 json 数据也并没有发送，在网上查找了一下，发现是因为触发了 CORS(Cross-Origin Resource Sharing: 跨域资源共享)机制。


CORS 的官方解释是：

> 对那些可能对服务器数据产生副作用的 HTTP 请求方法（特别是 GET 以外的 HTTP 请求，或者搭配某些 MIME 类型的 POST 请求），浏览器必须首先使用 OPTIONS 方法发起一个预检请求（preflight request），从而获知服务端是否允许该跨域请求。服务器确认允许之后，才发起实际的 HTTP 请求。在预检请求的返回中，服务器端也可以通知客户端，是否需要携带身份凭证（包括 Cookies 和 HTTP 认证相关数据）

跨域请求被分成了两类，简单请求和非简单请求。简单请求的标准：

> 
> - 请求方式为 HTTP/1.1 GET 或者 POST，如果是POST，则请求的Content-Type为以下之一： application/x-www-form-urlencoded, multipart/form-data, 或text/plain
> 
> - 在请求中，不会发送自定义的头部（如X-Modified）

对于简单请求，浏览器不会发起预检请求，因此，只需要我之前那样在服务端返回一个 `Access-Control-Allow-Origin: *` 头信息就可以了。

我们客户端请求的 `Content-Type` 为 `application/json`，属于非简单请求，因此浏览器会先发送一个 `OPTIONS` 方法的预检请求。

了解原因后提供一个对应 url 的 OPTIONS 请求接口，并且在 Response 添加了如下头信息：

	Access-Control-Allow-Origin: *
	Access-Control-Allow-Methods: POST, GET, OPTIONS
	Access-Control-Allow-Headers: Content-Type

客户端就可以正常执行请求了。

### 简单请求详细标准

- 使用下列方法之一：
	- GET
	- HEAD
	- POST
- Fetch 规范定义了对 CORS 安全的首部字段集合，不得人为设置该集合之外的其他首部字段。该集合为：
	- Accept
	- Accept-Language
	- Content-Language
	- Content-Type （需要注意额外的限制）
	- DPR
	- Downlink
	- Save-Data
	- Viewport-Width
	- Width
- Content-Type 的值仅限于下列三者之一：
	- text/plain
	- multipart/form-data
	- application/x-www-form-urlencoded
- 请求中的任意XMLHttpRequestUpload 对象均没有注册任何事件监听器；XMLHttpRequestUpload 对象可以使用 XMLHttpRequest.upload 属性访问。
- 请求中没有使用 ReadableStream 对象。

### 非简单请求详细标准

- 使用了下面任一 HTTP 方法：
	- PUT
	- DELETE
	- CONNECT
	- OPTIONS
	- TRACE
	- PATCH
- 人为设置了对 CORS 安全的首部字段集合之外的其他首部字段。该集合为：
	- Accept
	- Accept-Language
	- Content-Language
	- Content-Type (but note the additional requirements below)
	- DPR
	- Downlink
	- Save-Data
	- Viewport-Width
	- Width
- Content-Type 的值不属于下列之一:
	- application/x-www-form-urlencoded
	- multipart/form-data
	- text/plain
- 请求中的XMLHttpRequestUpload 对象注册了任意多个事件监听器。
- 请求中使用了ReadableStream对象。

## 参考文章

- [跨域资源共享 CORS 详解 - 阮一峰的网络日志](http://www.ruanyifeng.com/blog/2016/04/cors.html)
- [HTTP访问控制（CORS） - HTTP | MDN](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Access_control_CORS)
- [Server-Side Access Control - HTTP | MDN](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Server-Side_Access_Control)