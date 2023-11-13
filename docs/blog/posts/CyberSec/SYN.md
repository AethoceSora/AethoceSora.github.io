---
title: 记录镜像站遭遇 SYN 泛洪攻击的诊断与防御
tags: [Cyberspace Security]
date: 2023-01-07
categories: [Network,Ops,CyberSec]
---

# 镜像站遭遇 SYN 泛洪攻击的诊断与防御

## 异常的连接数

近日，镜像站莫名遭到多次攻击，症状表现为 TCP 连接数异常上升，久久得不到释放。后台监控检测到大量 5XX 错误，CPU 负载高，正常业务受到较大影响。

通过使用 netstat 命令可以统计服务器当前的各种 TCP 状态：

```shell
netstat -n | awk '/^tcp/ {++S[$NF]} END {for(a in S) print a, S[a]}'
```

我站在受攻击时的状态列表如下：

```shell
TIME_WAIT 168
CLOSE_WAIT 9
SYN_SENT 3
FIN_WAIT1 229
FIN_WAIT2 113
ESTABLISHED 234
SYN_RECV 1272
CLOSING 3
LAST_ACK 97
```

可以明显发现 SYN_RECV 连接数异常增多。

仅通过这一现象就可以判断为服务器受到了 SYN Flood 泛洪攻击。

<!-- more -->

## SYN Flood 泛洪攻击

### 什么是 SYN Flood 攻击

SYN 泛洪（半开连接攻击）是一种拒绝服务 (DDoS) 攻击，旨在耗尽可用服务器资源，致使服务器无法传输合法流量。攻击者通过重复发送初始连接请求 (SYN) 数据包，可击垮目标服务器计算机上的所有可用端口，导致目标设备在响应正常业务流量时表现迟钝乃至全无响应。

### SYN Flood 攻击的原理

SYN 泛洪攻击，是利用 TCP 连接的握手过程中的缺陷实现的。

攻击者通过发送大量伪造的 TCP 连接请求，常用假冒的 IP 或 IP 号段发来海量的请求连接的第一个握手包（SYN 包），被攻击服务器回应第二个握手包（SYN+ACK 包），因为对方是假冒 IP，对方永远收不到包且不会回应第三个握手包。导致被攻击服务器保持大量 SYN_RECV 状态的“半连接”，并且会重试默认 5 次回应第二个握手包，塞满 TCP 等待连接队列。

![SYN Flood DDoS 攻击动画](https://www.cloudflare.com/img/learning/ddos/syn-flood-ddos-attack/syn-flood-attack-ddos-attack-diagram-2.png)

### 防御 SYN Flood

#### 增大 Nginx 的并发限制

适度调整 Nginx 主配置文件中的`worker_rlimit_nofile` 和`worker_connections`

可参考如下配置：

```nginx
user root root;
worker_processes 4;
worker_rlimit_nofile 65535;

#error_log logs/error.log;
#error_log logs/error.log notice;
#error_log logs/error.log info;

#pid logs/nginx.pid;
events {
        use epoll;
        worker_connections 65535;
}
```

然后通过`nginx -s reload`命令热重载 Nginx 配置文件

#### 调整系统 TCP 栈参数

 在 sysctl 配置文件中，调整如下参数可有效减轻 SYN Flood 攻击带来的损害。

- `net.core.netdev_max_backlog`

  接受自网卡、但未被内核协议栈处理的报文队列长度

- `net.ipv4.tcp_max_syn_backlog`

  SYN_RCVD 连接状态的最大个数

- `net.ipv4.tcp_abort_on_overflow`

  超出处理能力时，对新来的 SYN 直接回报 RST，丢弃连接

#### 增大 linux 的最大文件句柄数限制

使用`ulimit -n` 查看本系统的最大文件句柄数限制，默认是 1024

可以通过修改`/etc/security/limts.conf`来改变该限制：

在文件的最后追加：

```
* soft nofile 655360
* hard nofile 655360
```

最前面的`*`代表全局设置。
