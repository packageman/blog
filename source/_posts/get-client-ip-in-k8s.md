---
title: Kubernetes 中如何获取客户端真实 IP
date: 2020-02-22 17:36:15
tags:
  - Kubernetes
  - Nginx
  - HTTP
---

## 前言

开发过程中，往往会遇到根据用户访问的客户端 IP 做一些统计相关的需求，而目前大部分的系统架构都使用了负载均衡或一些代理服务，这时就要有一种方案可以获取到用户真实的 IP 而不是负载均衡或代理的 IP。

## 背景

下图为我目前在用的系统架构：

![architecture](/images/client-ip-in-k8s/architecture_v1.png)
<!-- more -->
## 方案

### 使用 X-Forwarded-For 请求头

在客户端访问服务器的过程中如果需要经过 HTTP 代理或者负载均衡服务器，[X-Forwarded-For](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/X-Forwarded-For) 可以被用来获取最初发起请求的客户端的 IP 地址。

根据以上规范，在 nginx 的 access 日志中添加 X-Forwarded-For（`$http_x_forwarded_for`）的请求头信息用于获取客户端真实 IP。

```shell
log_format access '$remote_addr - $remote_user [$time_local] '
                '"$request" $status $body_bytes_sent '
                '"$http_referer" "$http_user_agent" $request_time "$http_x_forwarded_for"';
```

部署后测试发现 nginx 打印出来的 X-Forwarded-For 的值 139.224.xxx.234（负载均衡的 IP）。

```shell
192.168.1.32 - - [22/Feb/2020:18:58:19 +0800] "GET /images/mysql-count/mysql_count_explain_with_where.png HTTP/2.0" 200 38358 "https://merrychris.com/2017/11/02/mysql-count-takes-too-long/" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_12_6) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/80.0.3987.100 Safari/537.36" 0.088 - "139.224.xxx.234"
```

结合架构图与上述结果分析，猜测是 **nginx-ingress-controller 这个服务在转发请求给应用程序（upstream）时将 X-Forwarded-For 设置成了负载均衡的 IP**。

进入 nginx-ingress-controller 中查看相应的 nginx 配置文件：

```conf
...
map '' $the_real_ip {
        default          $remote_addr;
}
...
proxy_set_header X-Real-IP              $the_real_ip;

proxy_set_header X-Forwarded-For        $the_real_ip;

proxy_set_header X-Forwarded-Host       $best_http_host;
proxy_set_header X-Forwarded-Port       $pass_port;
proxy_set_header X-Forwarded-Proto      $pass_access_scheme;
...
```

以上代码证实了我的猜想。


**特别说明：X-Forwarded-For 请求头是不可靠的，客户端可以人为伪造。**

### 设置 use-forwarded-headers 配置项

查看 nginx-ingress-controller 文档，发现可以通过设置 [`use-forwarded-headers: true`](https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/configmap/#use-forwarded-headers) 使得 nginx-ingress-controller 将 X-Forwarded-For 设置为负载均衡传过来的客户端真实 IP。

修改 nginx-ingress-controller 的 configmap 文件如下：

![nginx-ingress-configmap](/images/client-ip-in-k8s/nginx_ingress_configmap.png)

再次测试发现应用服务日志中已经记录了客户端的真实 IP。（说明：nginx-ingress-controller 使用的是热更新，修改 configmap 后无需重启服务便可生效）

通过上面的代码片段可以发现 X-Forwarded-For 就是 `$the_real_ip`（=`$remote_addr`），访问日志中的 `$http_x_forwarded_for` 之所以变成了真实的客户端 IP 是因为使用了 nginx 的 [ngx_http_realip_module](http://nginx.org/en/docs/http/ngx_http_realip_module.html) 的 `real_ip_header` 配置，以下是 nginx-ingress-controller 中的相关代码片段：

```shell
{{/* Enable the real_ip module only if we use either X-Forwarded headers or Proxy Protocol. */}}
{{/* we use the value of the real IP for the geo_ip module */}}
{{ if or $cfg.UseForwardedHeaders $cfg.UseProxyProtocol }}
{{ if $cfg.UseProxyProtocol }}
real_ip_header      proxy_protocol;
{{ else }}
real_ip_header      {{ $cfg.ForwardedForHeader }};
{{ end }}
```

其中 `ForwardedForHeader` 的默认值就是 X-Forwarded-For，当 `UseForwardedHeaders` 为 true，ngx_http_realip_module 会将接收到的 X-Forwarded-For header 中的值设置为 `$remote_addr`。

### 使用四层负载均衡

目前阿里云的负载均衡同时支持四层和七层（文中在使用的）两种模式，对于七层的负载均衡需要采用如上配置，若为四层可省略 `use-forwarded-headers` 的修改，因为此时 nginx-ingress-controller 接收到的 IP 包源地源并未改变。

### 开启 toa 模块

可用于应用程序在用户态调用 `getpeername` 方法来获取客户端真实 IP，目前未涉及，只是做为扩展在这里提及一下。

## 参考

- https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers/X-Forwarded-For
- https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/configmap/#use-forwarded-headers
- https://help.aliyun.com/document_detail/54007.html?spm=5176.11783253.0.0.3b111eb9i5NtqX
- https://help.aliyun.com/document_detail/119658.html?spm=5176.10695662.1996646101.searchclickresult.4d14fabevR9RrZ&aly_as=cT70fjhX
- http://nginx.org/en/docs/http/ngx_http_realip_module.html
- https://cloud.tencent.com/document/product/1014/31124