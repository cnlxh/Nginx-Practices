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

## 重新编译nginx添加模块

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

## 基于城市限制访问

在主配置文件中的http段上方加载这个模块

```bash
vim /etc/nginx/nginx.conf
...
load_module modules/ngx_http_geoip2_module.so;
http {}
```

```bash
cat > /etc/nginx/conf.d/geo.conf <<'EOF'
geoip2 /usr/local/geoip/GeoLite2-Country.mmdb {
   $geoip2_country_code country iso_code;
}
geoip2 /usr/local/geoip/GeoLite2-City.mmdb {
    $geoip2_city_names city names en;
}
map $geoip2_country_code $allowed_country {
    default yes;
    CN yes;
}
map $geoip2_city_names $allowed_city {
    default yes;
    Shanghai no;
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

在配置文件中，我们定义了两个变量，分别为allowed_country和allowed_city，他们的值等于我们下面写的yes或no，然后通过server段的if语句来判断yes或no前面的城市或国家代码来决定是否返回403

上方配置文件处$geoip2_city_names city names en;这个变量后面的city names en是固定写法，这是从一个树形的结构中找出来的，还有别的写法，具体查询city names en还可以写成什么，可参考如下结构：

我们的city names en，就是从下面的结构中摘录的，例如我们想要获得Cangzhou这个字符串，那就写city names en，就能获得Cangzhou

```bash
[root@host1 ~]# mmdblookup --file /opt/GeoLite2-City.mmdb --ip XXXXXX(这里是IP地址，为隐私打码)

  {
    "city": 
      {
        "geoname_id": 
          1816080 <uint32>
        "names": 
          {
            "de": 
              "Cangzhou" <utf8_string>
            "en": 
              "Cangzhou" <utf8_string>
            "fr": 
              "Cangzhou" <utf8_string>
            "ja": 
              "滄州市" <utf8_string>
            "ru": 
              "Цанчжоу" <utf8_string>
            "zh-CN": 
              "沧州市" <utf8_string>
          }
      }
    "continent": 
      {
        "code": 
          "AS" <utf8_string>
        "geoname_id": 
          6255147 <uint32>
        "names": 
          {
            "de": 
              "Asien" <utf8_string>
            "en": 
              "Asia" <utf8_string>
            "es": 
              "Asia" <utf8_string>
            "fr": 
              "Asie" <utf8_string>
            "ja": 
              "アジア" <utf8_string>
            "pt-BR": 
              "Ásia" <utf8_string>
            "ru": 
              "Азия" <utf8_string>
            "zh-CN": 
              "亚洲" <utf8_string>
          }
      }
    "country": 
      {
        "geoname_id": 
          1814991 <uint32>
        "iso_code": 
          "CN" <utf8_string>
        "names": 
          {
            "de": 
              "China" <utf8_string>
            "en": 
              "China" <utf8_string>
            "es": 
              "China" <utf8_string>
            "fr": 
              "Chine" <utf8_string>
            "ja": 
              "中国" <utf8_string>
            "pt-BR": 
              "China" <utf8_string>
            "ru": 
              "Китай" <utf8_string>
            "zh-CN": 
              "中国" <utf8_string>
          }
      }
    "location": 
      {
        "accuracy_radius": 
          500 <uint16>
        "latitude": 
          38.316900 <double>
        "longitude": 
          116.847800 <double>
        "time_zone": 
          "Asia/Shanghai" <utf8_string>
      }
    "registered_country": 
      {
        "geoname_id": 
          1814991 <uint32>
        "iso_code": 
          "CN" <utf8_string>
        "names": 
          {
            "de": 
              "China" <utf8_string>
            "en": 
              "China" <utf8_string>
            "es": 
              "China" <utf8_string>
            "fr": 
              "Chine" <utf8_string>
            "ja": 
              "中国" <utf8_string>
            "pt-BR": 
              "China" <utf8_string>
            "ru": 
              "Китай" <utf8_string>
            "zh-CN": 
              "中国" <utf8_string>
          }
      }
    "subdivisions": 
      [
        {
          "geoname_id": 
            1808773 <uint32>
          "iso_code": 
            "HE" <utf8_string>
          "names": 
            {
              "en": 
                "Hebei" <utf8_string>
              "fr": 
                "Province de Hebei" <utf8_string>
              "zh-CN": 
                "河北省" <utf8_string>
            }
        }
      ]
  }


```

```bash
systemctl restart nginx
```

从上海访问将会拒绝，显示403

# 限制连接数

1. limit_conn_zone：binary_remote_addr是一个预定义的变量，用来表示客户端IP，我们创建了一个共享内存区域，名为limitbyaddr，共10m大小

2. limit_conn_status：超过限制之后的http返回码

3. 在server段limit_conn 后面使用我们定义的共享内存空间，并限制并发数为1

```bash
cat > /etc/nginx/conf.d/limitconn.conf <<'EOF'
limit_conn_zone $binary_remote_addr zone=limitbyaddr:10m;
limit_conn_status 429;
server {
 listen 80 default_server;
 limit_conn limitbyaddr 1;
 location / {
 root /aa;
 index index.html;
 }
}
EOF

```

需要注意的是，如果你的index.html太小，可能看上去这个限制不会生效，所以不要用echo的方式去生成，可以考虑dd命令

```bash
dd if=/dev/zero of=/aa/index.html bs=1M count=20
```

测试连接数限制

一共访问20次，每次并发2个请求，就会有一个失败，但是每次并发一个请求，就会都成功

```bash
yum install httpd-tools -y
[root@host1 ~]# ab -n 20 -c 2 http://127.0.0.1/
This is ApacheBench, Version 2.3 <$Revision: 1843412 $>
Copyright 1996 Adam Twiss, Zeus Technology Ltd, http://www.zeustech.net/
Licensed to The Apache Software Foundation, http://www.apache.org/

Benchmarking 127.0.0.1 (be patient).....done


Server Software:        nginx/1.22.0
Server Hostname:        127.0.0.1
Server Port:            80

Document Path:          /
Document Length:        169 bytes

Concurrency Level:      2
Time taken for tests:   0.009 seconds
Complete requests:      20
Failed requests:        1
   (Connect: 0, Receive: 0, Length: 1, Exceptions: 0)
Non-2xx responses:      19
Total transferred:      20977975 bytes
HTML transferred:       20974731 bytes
Requests per second:    2115.51 [#/sec] (mean)
Time per request:       0.945 [ms] (mean)
Time per request:       0.473 [ms] (mean, across all concurrent requests)
Transfer rate:          2166945.60 [Kbytes/sec] received

Connection Times (ms)
              min  mean[+/-sd] median   max
Connect:        0    0   0.0      0       0
Processing:     0    1   2.1      0       9
Waiting:        0    0   0.2      0       1
Total:          0    1   2.1      0       9

Percentage of the requests served within a certain time (ms)
  50%      0
  66%      0
  75%      0
  80%      0
  90%      1
  95%      9
  98%      9
  99%      9
 100%      9 (longest request)
```

```bash
[root@host1 ~]# ab -n 20 -c 1 http://127.0.0.1/
This is ApacheBench, Version 2.3 <$Revision: 1843412 $>
Copyright 1996 Adam Twiss, Zeus Technology Ltd, http://www.zeustech.net/
Licensed to The Apache Software Foundation, http://www.apache.org/

Benchmarking 127.0.0.1 (be patient).....done


Server Software:        nginx/1.22.0
Server Hostname:        127.0.0.1
Server Port:            80

Document Path:          /
Document Length:        20971520 bytes

Concurrency Level:      1
Time taken for tests:   0.164 seconds
Complete requests:      20
Failed requests:        0
Total transferred:      419435240 bytes
HTML transferred:       419430400 bytes
Requests per second:    121.63 [#/sec] (mean)
Time per request:       8.222 [ms] (mean)
Time per request:       8.222 [ms] (mean, across all concurrent requests)
Transfer rate:          2490937.17 [Kbytes/sec] received

Connection Times (ms)
              min  mean[+/-sd] median   max
Connect:        0    0   0.0      0       0
Processing:     8    8   0.2      8       9
Waiting:        0    0   0.0      0       0
Total:          8    8   0.2      8       9

Percentage of the requests served within a certain time (ms)
  50%      8
  66%      8
  75%      8
  80%      8
  90%      8
  95%      9
  98%      9
  99%      9
 100%      9 (longest request)
```


