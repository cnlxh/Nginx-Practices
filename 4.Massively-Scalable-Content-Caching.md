环境介绍：

| 操作系统       | IP地址           | 主机名              | Nginx版本 | 维护人 | 联系QQ      | 联系微信     |
| ---------- | -------------- | ---------------- | ------- | --- | --------- | -------- |
| CentOS 8.5 | 192.168.30.200 | host1.xiaohui.cn | 1.22.0  | 李晓辉 | 939958092 | Lxh_Chat |
| CentOS 8.5 | 192.168.30.201 | host2.xiaohui.cn | 1.22.0  |     |           |          |
| CentOS 8.5 | 192.168.30.202 | host3.xiaohui.cn | 1.22.0  |     |           |          |

# 准备hosts文件

```bash
cat >> /etc/hosts <<EOF
192.168.30.200 cache.xiaohui.cn cache
192.168.30.200 host1.xiaohui.cn host1
192.168.30.201 host2.xiaohui.cn host2
192.168.30.202 host3.xiaohui.cn host3
EOF
```

# 基本概念

客户端的每个请求，都需要服务器进行处理，一旦客户端规模很大，这势必对服务器会造成较大的带宽、存储、计算等多方面压力，此时就可以考虑将用户所需要的内容放到nginx服务器上进行缓存，然后把缓存有用户内容的nginx服务器放置到全国或全球各地，以供用户就近访问，这样就可以对上游服务器压力做分流，这就是内容分发网络CDN。

# 搭建缓存服务器

我们在这里指定了缓存位置为/data/cache，并在level中指定了目录为二级目录，第一级是一位，第二级是两位，nginx使用哈希键作为文件名，共享内存名为lixiaohui且大小为20m，最大缓存50GB，自用户上次访问时间算起，7*24=168h为不活动时间，即可清除缓存，当有人访问本网站(cache.xiaohui.cn)时，内容将从host2.xiaohui.cn中获取，针对所有(any)http代码响应都进行缓存

```bash
cat > /etc/nginx/conf.d/cache.conf  <<EOF
proxy_cache_path /data/cache levels=1:2 keys_zone=lixiaohui:20m max_size=50g inactive=168h;
server {
 listen 80 default_server;
 server_name cache.xiaohui.cn;
 location / {
   proxy_cache lixiaohui;
   proxy_cache_valid any 168h;
   proxy_pass http://host2.xiaohui.cn;
 }
}
EOF
```

准备缓存目录

```bash
mkdir /data/cache -p
semanage fcontext -a -t httpd_sys_rw_content_t '/data(/.*)'
semanage permissive -a httpd_t
restorecon -RvF /data/
```

```bash
systemctl enable nginx --now
```

# 缓存锁定机制

当大量的客户端同时向缓存服务器请求同一个内容，而这个内容不存在于缓存服务器上时，就都会向真实服务器提交请求，这样压力过高，nginx提供了proxy_cache_lock参数用于解决此问题，只允许第一个请求访问真实服务器并缓存到本地，其他的请求等待并从缓存中获得此内容。

本例中，如果第一个前往真实服务器的请求没有在3秒内完成任务，将会放行下一个请求进入真实服务器，如果6秒内还没有完成任务，就会继续放行下一个，但不再做缓存处理。

```bash
cat > /etc/nginx/conf.d/cache.conf  <<EOF
proxy_cache_path /data/cache levels=1:2 keys_zone=lixiaohui:20m max_size=50g inactive=168h;
server {
 listen 80 default_server;
 server_name cache.xiaohui.cn;
 location / {
   proxy_cache lixiaohui;
   proxy_cache_valid any 168h;
   proxy_pass http://host2.xiaohui.cn;
   proxy_cache_lock on;
   proxy_cache_lock_age 3s;
   proxy_cache_lock_timeout 6s;
 }
}
EOF
```

# 配置缓存哈希键名

proxy_cache_key

根据请求主机、 URI 和 cookie 作为缓存键名，这样就可以缓存包含动态页面在内的内容

```bash
cat > /etc/nginx/conf.d/cache.conf  <<'EOF'
proxy_cache_path /data/cache levels=1:2 keys_zone=lixiaohui:20m max_size=50g inactive=168h;
server {
 listen 80 default_server;
 server_name cache.xiaohui.cn;
 location / {
   proxy_cache lixiaohui;
   proxy_cache_valid any 168h;
   proxy_pass http://host2.xiaohui.cn;
   proxy_cache_lock on;
   proxy_cache_lock_age 3s;
   proxy_cache_lock_timeout 6s;
   proxy_cache_key "$host$request_uri $cookie_user";
 }
}
EOF
```

# 跳过缓存

proxy_cache_bypass

当出现故障需要排错时，总是从缓存取得内容可能不够方便，或者其他目的，不能或不愿使用缓存时，在请求中添加header即可

当proxy_cache_bypass这个header的值是非0或非空时，将不使用缓存，直接将请求发给真实服务器，

```bash
cat > /etc/nginx/conf.d/cache.conf  <<'EOF'
proxy_cache_path /data/cache levels=1:2 keys_zone=lixiaohui:20m max_size=50g inactive=168h;
server {
 listen 80 default_server;
 server_name cache.xiaohui.cn;
 location / {
   proxy_cache lixiaohui;
   proxy_cache_valid any 168h;
   proxy_pass http://host2.xiaohui.cn;
   proxy_cache_lock on;
   proxy_cache_lock_age 3s;
   proxy_cache_lock_timeout 6s;
   proxy_cache_key "$host$request_uri $cookie_user";
   proxy_cache_bypass $http_cache_bypass;
 }
}
EOF
```

# 提升缓存性能

使用客户端侧的cache-control header可以提升缓存性能，下例中，其内容将会缓存1年，并在header中添加cache-control等于public的消息头，这允许任何缓存服务器缓存其内容，如果是private，则只允许客户端缓存其内容。

```bash
cat > /etc/nginx/conf.d/cache.conf  <<'EOF'
proxy_cache_path /data/cache levels=1:2 keys_zone=lixiaohui:20m max_size=50g inactive=168h;
server {
 listen 80 default_server;
 server_name cache.xiaohui.cn;
 location / {
   expires 1y;
   add_header Cache-Control "public";
   proxy_cache lixiaohui;
   proxy_cache_valid any 168h;
   proxy_pass http://host2.xiaohui.cn;
}
}
EOF
```

# 允许清除缓存

默认nginx并不支持清除缓存，需要重新编译nginx以支持此功能

```bash
cat > /etc/yum.repos.d/nginx-prep.repo <<EOF
[baseostream]
baseurl=https://mirrors.tuna.tsinghua.edu.cn/centos/8-stream/BaseOS/x86_64/os/
enabled=1
gpgcheck=0
name=baseo-stream

[appstreamstream]
baseurl=https://mirrors.tuna.tsinghua.edu.cn/centos/8-stream/AppStream/x86_64/os/
enabled=1
gpgcheck=0
name=appstream-stream

[epel-repo]
baseurl=https://mirrors.tuna.tsinghua.edu.cn/epel/8/Everything/x86_64/
enabled=1
gpgcheck=0
name=epel-repo
EOF
```

```bash
yum -y install gcc redhat-rpm-config.noarch pcre-devel openssl openssl-devel \
libxml2 libxml2-devel libxslt-devel gd-devel perl-devel perl-ExtUtils-Embed make
```

```bash
wget http://nginx.org/download/nginx-1.22.0.tar.gz
wget http://labs.frickle.com/files/ngx_cache_purge-2.3.tar.gz
tar xf nginx-1.22.0.tar.gz
tar xf ngx_cache_purge-2.3.tar.gz
cd nginx-1.22.0
nginx -V
# 然后复制configure后面的参数，替换XXXX
./configure --XXXX --add-module=/root/ngx_cache_purge-2.3
make
systemctl stop nginx
cp objs/nginx /usr/sbin/nginx
systemctl restart nginx
```

这里添加了二级目录purge，并允许192.168.30.0/24来清除缓存

假设资源地址为http://cache.xiaohui.cn/1.jpg，那清除缓存的路径是http://cache.xiaohui.cn/purge/1.jpg

```bash
cat > /etc/nginx/conf.d/cache.conf  <<'EOF'
proxy_cache_path /data/cache levels=1:2 keys_zone=lixiaohui:20m max_size=50g inactive=168h;
server {
 listen 80 default_server;
 server_name cache.xiaohui.cn;
location ~ /purge(/.*) {
    allow 192.168.30.0/24;
    deny all;
    proxy_cache_purge lixiaohui $host$1$is_args$args;
}
 location / {
   expires 1y;
   add_header Cache-Control "public";
   proxy_cache lixiaohui;
   proxy_cache_valid any 168h;
   proxy_cache_key "$host$request_uri";
   proxy_pass http://host2.xiaohui.cn;
   add_header Nging-Cache "$upstream_cache_status";
}
}
EOF
```

# 缓存切片

当网站上有一个很大的文件时，客户端如果直接请求这个很大的文件，对客户端以及服务器来讲，都是很大的压力，缓存切片做的事情就是把这个大文件切片成固定大小去后端服务器请求，然后以这个大小返回给客户端，客户端只需要按照这个大小去服务器申请即可，这样就从请求一个超大文件，变成了请求几个小文件

```bash
cat > /etc/nginx/conf.d/cache.conf  <<'EOF'
proxy_cache_path /data/cache levels=1:2 keys_zone=lixiaohui:20m max_size=50g inactive=168h;
server {
 listen 80 default_server;
 server_name cache.xiaohui.cn;
location ~ /purge(/.*) {
    allow 192.168.30.0/24;
    deny all;
    proxy_cache_purge lixiaohui $host$1$is_args$args;
}
 location / {
   slice 1m;
   proxy_set_header Range $slice_range;
   proxy_http_version 1.1;
   expires 1y;
   add_header Cache-Control "public";
   proxy_cache lixiaohui;
   proxy_cache_valid any 168h;
   proxy_cache_key "$host$request_uri$slice_range";
   proxy_pass http://host2.xiaohui.cn;
   add_header Nging-Cache "$upstream_cache_status";
}
}
EOF
```

测试缓存分片效果

可以看到 http响应码是206 Partial Content，Content-Length:  1048576，Content-Range: bytes 0-1048575/104857600，我们请求的长度是0-1048575，本文件最长为104857600

```bash
[root@host1 ~]# curl -I -r 0-1048575 http://cache.xiaohui.cn/aa.txt
HTTP/1.1 206 Partial Content
Server: nginx/1.22.0
Date: Sat, 10 Sep 2022 07:10:27 GMT
Content-Type: text/plain
Content-Length: 1048576
Connection: keep-alive
Last-Modified: Sat, 10 Sep 2022 07:06:39 GMT
ETag: "631c377f-6400000"
Expires: Sun, 10 Sep 2023 07:10:27 GMT
Cache-Control: max-age=31536000
Cache-Control: public
Nging-Cache: MISS
Content-Range: bytes 0-1048575/104857600
```

可以看到本次访问是第一次访问，并未命中缓存，我们去host2上看一下日志，发现代码为206，返回给缓存服务器的长度为1048576

```bash
[root@host2 ~]# tail /var/log/nginx/access.log 
192.168.30.200 - - [10/Sep/2022:03:08:21 -0400] "GET /aa.txt HTTP/1.1" 206 1048576 "-" "curl/7.61.1" "-"
192.168.30.200 - - [10/Sep/2022:03:10:27 -0400] "GET /aa.txt HTTP/1.1" 206 1048576 "-" "curl/7.61.1" "-"
```
