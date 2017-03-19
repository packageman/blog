---
title: HTTP 缓存应用
date: 2017-03-18 17:25:15
tags:
  - HTTP
---

## 前言

最近项目组安排了一个关于 HTTP 静态资源缓存的分享，我也借此机会去了解了下有关 HTTP 缓存的知识。和目前我们线上环境所使用的静态资源缓存策略。

## 基础知识

### Cache-Control

HTTP Header *Cache-Control* 在 HTTP1.1 中引入，旨在替代 HTTP1.0 中 *Expires* 和 *Pragma*。*Cache-Control* 既可以用于请求头中，也可以用于响应头中，这里我们先讨论响应头中的。

*Cache-Control* 作为响应头时主要有以下三个作用：

- 控制缓存的可缓存性(Cacheability)
- 控制缓存的到期(Expiration)
- 控制缓存是否需要重新验证和重新加载(Revalidation and reloading)

*可缓存性*：

- `private`：表示它只应该存在本地缓存
- `public`：表示它既可以存在共享缓存(CDN, 代理服务器等)，也可以被存在本地缓存
- `no-cache`：表示不论是本地缓存还是共享缓存，在使用它以前必须用缓存里的值来重新验证
- `no-store`：表示不允许被缓存
- `only-if-cached`: 表明如果缓存存在，只使用缓存，无论原始服务器数据是否有更新

*到期时间*:

- `max-age=<seconds>`: 设置缓存时间，设置单位为秒。本地缓存和共享缓存都可以
- `s-maxage=<seconds>`: 覆盖 max-age 属性。只在共享缓存中起作用。私有缓存中它被忽略

*重新验证和(重新加载)*

- `must-revalidate`: 缓存必须在使用之前验证旧资源(stale resources)的状态，并且不可使用过期资源。
- `proxy-revalidate`: 与 `must-revalidate` 相同，但它仅适用于共享缓存(例如代理)，并被私有缓存忽略


如 `Cache-Control: public max-age=3600 s-maxage=7200` 表示 本地缓存中缓存 1 小时，共享缓存中缓存 2 小时。

### ETag/If-None-Match, Last-Modified/If-Modified-Since

上面我们提到的 *重新验证* 就是通过 *ETag/If-None-Match* 或 *Last-Modified/If-Modified-Since* 来进行的。

*ETag* 用于 HTTP 响应头中，它是文档版本的标识符。通常是内容的 MD5 摘要值。如：

```
ETag: "33a64df551425fcc55e4d42a148795d9f25f89d4"
```

*If-None-Match* 用于 HTTP 请求头中，用于验证缓存中的内容相对于服务器是否是最新的。当客户端第一次访问某一资源时，如果响应中设置了可以被缓存(如：*Cache-Control: max-age=3600*), 客户端会将响应内容连同响应头一起缓存起来(其中就包含 *ETag*，假设值为 "33a64df551425fcc55e4d42a148795d9f25f89d4")，客户端再次请求同一资源时，会先从本地缓存中寻找并验证其是否过期，如果已经过期并且设置了需要进行重新验证，客户端会在发起的请求中加入 *If-None-Match* 头，值为缓存中 *ETag* 的值，即 "33a64df551425fcc55e4d42a148795d9f25f89d4"。

```
If-None-Match: "33a64df551425fcc55e4d42a148795d9f25f89d4"
```

服务端收到请求后验证 *ETag* 的值是否与本地的一致，如果一致则返回 304 Not Modified 的响应，客户端在收到该响应后会直接使用本地的缓存数据，否则服务端返回实际的请求内容。这一过程称作 **缓存协商**

*Last-Modified/If-Modified-Since* 与 *ETag/If-None-Match* 同样的原理，只是 *Last-Modified/If-Modified-Since* 中使用的是文件更新时间。使用 *ETag* 要比 *Last-Modified* 对缓存的控制更加精确一些，因为对于同一个资源文件 *ETag* 的值是不变的，而 *Last-Modified* 可能会改变(如在系统升级的时候，替换了服务器上的文件，导致其更新时间改变)，从而产生了一次多余的数据获取。

当 *ETag* 和 *Last-Modefied* 同时使用时，*ETag* 优先级要高于 *Last-Modefied*。

## 应用

对于我们线上的环境是 **在入口文件设置 *Cache-Control: no-cache* 并开启 *ETag*([nginx 中 *Etag* 是默认开启的](http://nginx.org/en/docs/http/ngx_http_core_module.html#etag)) 即每次都会进行重新验证，而对于其它资源文件设置一个很大的缓存过期时间，*Cache-Control: max-age=31536000* （一年），对于除入口文件以外的资源文件以文件内容的 MD5 摘要值命名(现在很多前端编译工具都可以做到，比如 [webpack](https://webpack.github.io/))。**

这样一来可以最大化利用缓存，而且当有新版本上线时。**由于入口文件每次都要重新验证，从而保证客户端获取到的入口文件始终是最新的。而在入口文件中引入的其它资源文件，如果其内容改变了，必会导致其文件名改变(基于内容的 MD5 摘要值变了)，从而导致本地缓存 miss， 客户端便会拉取服务端最新的资源文件。如果内容没变，文件名也不改变，客户端将直接从缓存中获取资源。**

以下是对应的 nginx 配置：

```
server {
    listen 443;
    server_name example.com;

    location / {
        try_files $uri $uri/ /index.html;
        expires -1;
    }
    
    location ~* \.(css|js|jpg|png|gif)$ {
        expires 365d;
    }
}
```

## 特别说明

在使用 Chrome 浏览器(版本 56.0.2924.87)的时候：

- 按 *F5/Ctrl+R* 刷新时会先从本地缓存中加载资源，如果缓存过期，再进行 *缓存协商*
- 按 *Ctrl+F5/Ctrl+Shift+R* 进行强制刷新时将既不使用本地缓存也不进行 *缓存协商* (这是通过在请求头中设置 *Cache-Control: no-cache* 实现的)

扩展：

- 如果客户端的本地缓存还没有过期并且没有设置重新验证，这时如何让请求可以跳过 *本地缓存* 而不跳过 *缓存协商* 呢？答案是在请求头中加入 *Cache-Control: max-age=0*。

以下是 [rfc2616 的部分说明](https://www.w3.org/Protocols/rfc2616/rfc2616-sec14.html#sec14.9.3)：

>**End-to-end reload**
>&nbsp;&nbsp;&nbsp;&nbsp;The request includes a "no-cache" cache-control directive or, for compatibility with HTTP/1.0 clients, "Pragma: no-cache". Field names MUST NOT be included with the no-cache directive in a request. The server MUST NOT use a cached copy when responding to such a request.
>
>**Specific end-to-end revalidation**
>&nbsp;&nbsp;&nbsp;&nbsp;The request includes a "max-age=0" cache-control directive, which forces each cache along the path to the origin server to revalidate its own entry, if any, with the next cache or server. The initial request includes a cache-validating conditional with the client's current validator.
>
>**Unspecified end-to-end revalidation**
>&nbsp;&nbsp;&nbsp;&nbsp;The request includes "max-age=0" cache-control directive, which forces each cache along the path to the origin server to revalidate its own entry, if any, with the next cache or server. The initial request does not include a cache-validating conditional; the first cache along the path (if any) that holds a cache entry for this resource includes a cache-validating conditional with its current validator.

## 思考

- HTTP 缓存在 CDN(代理服务器) 中的应用是怎样的？
- HTTP 缓存在分布式系统中需要注意的地方

## 参考

- https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Cache-Control
- https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/ETag
- https://zhuanlan.zhihu.com/p/25512679
- https://github.com/fouber/blog/issues/6
- http://stackoverflow.com/questions/1046966/whats-the-difference-between-cache-control-max-age-0-and-no-cache