环境介绍：

| 操作系统       | IP地址           | 主机名              | Nginx版本 | 维护人 | 联系QQ      | 联系微信     |
| ---------- | -------------- | ---------------- | ------- | --- | --------- | -------- |
| CentOS 8.5 | 192.168.30.200 | host1.xiaohui.cn | 1.22.0  | 李晓辉 | 939958092 | Lxh_Chat |
| CentOS 8.5 | 192.168.30.201 | host2.xiaohui.cn | 1.22.0  |     |           |          |
| CentOS 8.5 | 192.168.30.202 | host3.xiaohui.cn | 1.22.0  |     |           |          |

# 准备hosts文件

```bash
cat >> /etc/hosts <<EOF
192.168.30.200 httpha.xiaohui.cn httpha
192.168.30.200 host1.xiaohui.cn host1
192.168.30.201 host2.xiaohui.cn host2
192.168.30.202 host3.xiaohui.cn host3
EOF
```

# 基本概念

NGINX提供了基于HTTP、HTTPS、TCP、UDP协议的负载均衡，在开源版本的nginx中，提供了被动的健康检查，这可以减少上游服务器的压力，在nginx plus版本中，同时提供了主动和被动两种健康检查，主动健康检查每隔一定的时间就会向上游服务器发起连接或请求，来确保其健康，但是这会带来一定的负载。

# 负载均衡案例

## HTTP 负载均衡

```bash
cat > /etc/nginx/conf.d/httpha.conf <<EOF
upstream load {
 server 192.168.30.201:80 weight=1;
 server 192.168.30.202:80 weight=2;
 server 192.168.30.203:80 backup;
}
server {
 listen 80;
 server_name httpha.xiaohui.cn;
 location / {
 proxy_pass http://load;
 }
}
EOF
```

1. 定义了一个上游服务器池：load，定义了两个主力服务器，并分配了合适的权重

2. 第三个服务器后面的backup可以是以下的几种
   
   | 选项           | 含义                      |
   | ------------ | ----------------------- |
   | down         | 目前宕机，不参与负载均衡            |
   | backup       | 当具有权重的正常服务器全宕机后，此服务器将启用 |
   | max_fails    | 最大请求失败次数                |
   | fail_timeout | 超过最大失败次数后，暂停多长时间        |
   | max_conns    | 最大连接数                   |

## TCP 负载均衡

nginx 使用stream模块来完成TCP端口负载均衡

要注意stream代码段不能写到http代码段内，要和http同级

```bash
mkdir /etc/nginx/stream.conf.d
vim /etc/nginx/nginx.conf
...
stream {
    include /etc/nginx/stream.conf.d/*.conf;
}
...
```

```bash
cat > /etc/nginx/stream.conf.d/tcp.conf <<EOF
upstream sshd {
    server 192.168.30.201:22 weight=1;
    server 192.168.30.202:22 weight=2;
}
  server {
    listen 10000;
    proxy_pass sshd;
}
EOF
```

## UDP负载均衡

listen 的端口号之后指明为UDP

```bash
cat > /etc/nginx/stream.conf.d/udp.conf <<EOF
upstream ntp {
 server 192.168.30.201:123 weight=2;
 server 192.168.30.202:123 weight=2;
 }
 server {
 listen 123 udp;
 proxy_pass ntp;
 }
EOF
```

如果允许同一个机器上的多个进程同时bind同一个端口号，可考虑以下用法：

reuseport参数让NGINX对于每个工作进程去创建一个单独的监听套接字，这允许内核在工作进程之间分配传入的连接，以处理在客户机和服务器之间发送的多个数据包

```bash
cat > /etc/nginx/stream.conf.d/udp.conf <<EOF
upstream ntp {
 server 192.168.30.201:123 weight=2;
 server 192.168.30.202:123 weight=2;
 }
 server {
 listen 123 udp reuseport;
 proxy_pass ntp;
 }
EOF
```

# 负载均衡类型

NGINX的负载均衡类型：

1. 最少连接

2. 最少时间

3. 轮询调度

4. 通用哈希

5. 随机调度

6. IP哈希

## 轮询调度

这是默认的负载均衡方法，它按照上游池中服务器列表的顺序分发请求。如果上游服务器的配置不同，还可以考虑加权轮询，权重的整数值越高，服务器在轮询中越受青睐，权重背后的算法只是加权平均的统计概率

## 最少连接

此方法通过将当前请求代理到具有最少打开连接数的上游服务器来负载均衡。在决定将连接发送到哪个服务器时，最小连接(如轮询)也要考虑权重。指令名是least_conn

## 最少时间

它只在NGINX Plus中可用，least time类似于least connections，它代理当前连接数最少的上游服务器，但更倾向于平均响应时间最低的服务器。该方法是最复杂的负载平衡算法之一，适合高性能web应用程序的需求。这个算法是在最少连接上的增益，因为少量连接并不一定意味着最快的响应。在使用该算法时，重要的是要考虑考虑服务请求时间的统计差异。有些请求可能自然需要更多的处理，因此需要更长的请求时间，从而增加了统计的范围。长请求时间并不总是意味着性能较低或服务器超负荷工作。然而，需要更多处理的请求可能是异步工作流的候选者。此指令必须指定header或last_byte的参数，当指定header时，将使用接收响应header的时间。当last_byte指定时，将使用接收完整响应的时间。指令名是least_time

## 通用哈希

管理员使用给定的文本、请求或运行时的变量(或两者都有)定义散列。NGINX通过生成当前请求的散列并将其放在上游服务器上，从而在各个服务器之间分配负载。当需要更多地控制发送请求到哪个服务器或确定哪个上游服务器最有可能缓存数据时，此方法非常有用。注意，当添加服务器或从池中删除服务器时，散列请求将重新分配。该算法有一个可选参数，即consistent，以最小化重分配的影响。指令名是hash

## 随机调度

该方法用于指示NGINX从组中随机选择一个服务器，并考虑服务器权重。可选的两个[method]参数指示NGINX随机选择两个服务器，然后使用提供的负载均衡方法来平衡这两个服务器。默认情况下，如果传入过来没有method，则使用least_conn方法。用于随机负载均衡的指令名称为random

## IP 哈希

此方法仅适用于HTTP。IP哈希使用客户端IP地址作为哈希值。与在通用散列中使用远程变量略有不同，该算法使用IPv4地址或整个IPv6地址的前三个字节。此方法确保只要该服务器可用，客户机就被代理到同一上游服务器，这在关注会话状态而不是由应用程序的共享内存处理时非常有用。该方法在分布散列时也考虑了权重参数。指令名是ip_hash

最小连接数的示例如下：

```bash
cat > /etc/nginx/conf.d/httpha.conf <<EOF
upstream load {
 least_conn;
 server 192.168.30.201:80 weight=1;
 server 192.168.30.202:80 weight=2;
 server 192.168.30.203:80 backup;
}
server {
 listen 80;
 server_name httpha.xiaohui.cn;
 location / {
 proxy_pass http://load;
 }
}
EOF
```

# 被动健康检查

开源版本的nginx只支持被动健康检查

```bash
cat > /etc/nginx/conf.d/httpha.conf <<EOF
upstream load {
 least_conn;
 server 192.168.30.201:80 weight=1 max_fails=3 fail_timeout=3s;
 server 192.168.30.202:80 weight=2 max_fails=3 fail_timeout=3s;
 server 192.168.30.203:80 backup max_fails=3 fail_timeout=3s;
}
server {
 listen 80;
 server_name httpha.xiaohui.cn;
 location / {
 proxy_pass http://load;
 }
}
EOF
```

HTTP、TCP和UDP负载均衡通过使用相同的服务器参数来配置。被动监控监视客户端请求通过NGINX时失败或超时的连接。这里提到的参数允许调整它们的行为。max_fails的默认值为1,fail_timeout的默认值为10s。健康监控对于所有类型的负载均衡都很重要，不仅从用户体验的角度来看，而且从业务连续性的角度来看也是如此。NGINX被动地监控上游HTTP, TCP和UDP服务器，以确保它们是健康的和运行的。
