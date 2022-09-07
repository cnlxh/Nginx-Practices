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

# HTTP 负载均衡

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

# TCP 负载均衡

默认RPM包没有带stream模块，自己编译一下

```bash
wget https://codeload.github.com/nginx/nginx/tar.gz/refs/tags/release-1.22.0
tar xf nginx-release-1.22.0.tar.gz
cd nginx-release-1.22.0
yum install gcc -y
# 加入最后的--with-stream
./auto/configure --prefix=/etc/nginx --sbin-path=/usr/sbin/nginx --modules-path=/usr/lib64/nginx/modules --conf-path=/etc/nginx/nginx.conf --error-log-path=/var/log/nginx/error.log --http-log-path=/var/log/nginx/access.log --pid-path=/var/run/nginx.pid --lock-path=/var/run/nginx.lock --http-client-body-temp-path=/var/cache/nginx/client_temp --http-proxy-temp-path=/var/cache/nginx/proxy_temp --http-fastcgi-temp-path=/var/cache/nginx/fastcgi_temp --http-uwsgi-temp-path=/var/cache/nginx/uwsgi_temp --http-scgi-temp-path=/var/cache/nginx/scgi_temp --user=nginx --group=nginx --with-compat --with-file-aio --with-threads --with-http_addition_module --with-http_auth_request_module --with-http_dav_module --with-http_flv_module --with-http_gunzip_module --with-http_gzip_static_module --with-http_mp4_module --with-http_random_index_module --with-http_realip_module --with-http_secure_link_module --with-http_slice_module --with-http_ssl_module --with-http_stub_status_module --with-http_sub_module --with-http_v2_module --with-mail --with-mail_ssl_module --with-stream --with-stream_realip_module --with-stream_ssl_module --with-stream_ssl_preread_module --with-cc-opt='-O2 -g -pipe -Wall -Werror=format-security -Wp,-D_FORTIFY_SOURCE=2 -Wp,-D_GLIBCXX_ASSERTIONS -fexceptions -fstack-protector-strong -grecord-gcc-switches -specs=/usr/lib/rpm/redhat/redhat-hardened-cc1 -specs=/usr/lib/rpm/redhat/redhat-annobin-cc1 -m64 -mtune=generic -fasynchronous-unwind-tables -fstack-clash-protection -fcf-protection -fPIC' --with-ld-opt='-Wl,-z,relro -Wl,-z,now -pie' --with-stream
```


