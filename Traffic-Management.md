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

NGINX可以基于许多属性智能地路由流量和控制流，本篇内容大概介绍NGINX：

1. 基于百分比分割客户端请求的能力

2. 利用客户端的地理位置

3. 控制流量速率、连接和带宽限制

也可以混合和匹配这些特性，从而实现无数的可能性

# A/B 测试

A/B测试也叫分离测试，主要是将客户端的请求按百分比分发到不同服务器上

1. 准备后端服务器

2. 准备前端nginx用于分流

3. 测试分流效果

## 准备后端服务器

在host2和host3上安装并启动nginx服务之后，执行以下命令

```bash
cat > /etc/nginx/conf.d/static.conf <<EOF
server {
 listen 80 default_server; 
 location / {
 root /usr/share/nginx/html;
 # alias /usr/share/nginx/html;
 index index.html index.htm;
 }
}
EOF
```

在host2上创建页面

```bash
echo host2 page > /usr/share/nginx/html/index.html
systemctl restart nginx
```

在host3上创建页面

```bash
echo host3 page > /usr/share/nginx/html/index.html
systemctl restart nginx
```

## 准备nginx用于分流

```bash
cat > /etc/nginx/conf.d/split.conf <<'EOF'
split_clients "${remote_addr}" $splittest{
    50%     "first";
    50%     "second";
    *       "";
}
upstream first {
    server 192.168.30.201:80;
}
upstream second {
    server 192.168.30.202:80;
}
server {
    listen 80;
    server_name host1.xiaohui.cn;
    location / {
        proxy_pass http://$splittest$request_uri;
    }
}
EOF
```

给nginx提供一个默认页用于和其他服务器区分

```bash
echo default web site > /usr/share/nginx/html/index.html
systemctl restart nginx
```

## 测试分流效果

```bash
[root@host2 ~]# curl host1.xiaohui.cn
host2 page
[root@host3 ~]# curl host1.xiaohui.cn
host3 page
```

发现不同的主流去访问nginx的时候，已经分流成功了。

# 利用客户端的地理位置响应

这里需要根据客户端请求的IP地址来判断国家和城市，所以需要用到IP地址数据库，此处采用www.maxmind.com提供的免费产品，请注意需要注册才可以下载数据，注册之后，点击下面的连接即可下载，注意需要下载国家和城市两个数据库，下载完之后把两个mmdb后缀的文件上传到服务器的合适位置

https://www.maxmind.com/en/accounts/current/geoip/downloads

我把数据放到了/usr/local/geoip

```bash
mkdir /usr/local/geoip
cp Geo* /usr/local/geoip/
ll /usr/local/geoip/
-rw-r--r--. 1 root root 69619043 Sep  8 03:44 GeoLite2-City.mmdb
-rw-r--r--. 1 root root  5590205 Sep  8 03:44 GeoLite2-Country.mmdb
```

默认情况下，nginx并没有带此任务所需要的模块，需要我们编译

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
libxml2 libxml2-devel libxslt-devel gd-devel perl-devel perl-ExtUtils-Embed \
libmaxminddb-devel GeoIP GeoIP-devel GeoIP-data make
```

```bash
wget http://nginx.org/download/nginx-1.22.0.tar.gz
wget https://github.com/leev/ngx_http_geoip2_module/archive/refs/heads/master.zip
tar xf nginx-1.22.0.tar.gz
unzip -d /usr/lib64/nginx/modules/ master.zip
cd nginx-1.22.0
nginx -V
# 然后复制configure后面的参数，替换XXXX
./configure --XXXX --add-dynamic-module=/usr/lib64/nginx/modules/ngx_http_geoip2_module-master
make -j 10
cp objs/*.so /usr/lib64/nginx/modules/
systemctl stop nginx
cp objs/nginx /usr/sbin
systemctl restart nginx
```

在主配置文件中的http段上方加载这个模块

```bash
vim /etc/nginx/nginx.conf
...
load_module modules/ngx_http_geoip2_module.so;
http {}
```

```bash
cat > /etc/nginx/conf.d/geo.conf <<'EOF'
geoip2 /opt/GeoLite2-Country.mmdb {
   $geoip2_country_code country iso_code;
}
geoip2 /opt/GeoLite2-City.mmdb {
    $geoip2_city_names location time_zone;
}
map $geoip2_country_code $allowed_country {
    default yes;
    CN no;
}
map $geoip2_city_names $allowed_city {
    default yes;
    Asia/Shanghai no;
}
server {
 listen 80;
 server_name httpha.xiaohui.cn;
 location / {
 root /aa;
 index index.html;
 }
if ( $allowed_city = no ) { return 403; }
if ( $allowed_country = no ) { return 403; }

}
EOF


```

```bash
systemctl restart nginx
```

从上海访问将会拒绝，显示403



在Nginx前还有其他代理的话。可以通过Nginx的 X-Forwarded-For header进行递归查找分析源IP

未完待续


