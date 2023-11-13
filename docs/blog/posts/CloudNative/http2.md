---
title: 如何在 Nginx 上支持 HTTP/2
cover: https://s2.loli.net/2022/12/26/zN9BEclKUFpyDtw.png
tags: [Devops]
date: 2022-12-26
categories: [Network,Ops]
---

# 如何在 Nginx 上支持 HTTP/2

HTTP/2（原名HTTP 2.0）即超文本传输协议第二版，使用于万维网。HTTP/2 主要基于 SPDY 协议，通过对 HTTP 头字段进行数据压缩、对数据传输采用多路复用和增加服务端推送等举措，来减少网络延迟，提高客户端的页面加载速度。HTTP/2 没有改动 HTTP 的应用语义，仍然使用 HTTP 的请求方法、状态码和头字段等规则，它主要修改了 HTTP 的报文传输格式，通过引入二进制分帧实现性能的提升。

<!-- more -->

### 编译带有 HTTP/2 支持的 Nginx

默认编译的 Nginx 并不包含 HTTP/2 模块，我们需要加入参数来编译。

我们首先记录下 Nginx 原先的编译配置：

```shell
/usr/local/nginx/sbin/nginx -V
```

得到下面的内容：

```shell
nginx version: nginx/1.20.2
built by gcc 4.8.5 20150623 (Red Hat 4.8.5-44) (GCC)
built with OpenSSL 1.0.2k-fips  26 Jan 2017
TLS SNI support enabled
configure arguments: --prefix=/usr/local/nginx --add-dynamic-module=../ngx-fancyindex-master/ --add-dynamic-module=../njs/nginx/ --with-http_addition_module --with-http_ssl_module --with-compat --with-http_stub_status_module
```

我们需要将到 Nginx 的`configure arguments`复制下来，到安装包的`./configure`中加入`--with-http_v2_module`，如果没有 ssl 支持，还需要加入`--with-http_ssl_module`

参考：

```shell
./configure --prefix=/usr/local/nginx --add-dynamic-module=../ngx-fancyindex-master/ --add-dynamic-module=../njs/nginx/ --with-http_addition_module --with-http_ssl_module --with-compat --with-http_stub_status_module --with-http_v2_module --with-http_sub_module
```

### Nginx 配置文件

主要是配置 Nginx 的 `server` 块：在`listen 443 ssl`的后面加一个`http2`即可

```nginx
server {
listen 443 ssl http2 ;
server_name mirrors.qlu.edu.cn;

ssl_certificate /.../public.crt;
ssl_certificate_key /.../private.key;

...
```
