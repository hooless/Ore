### Nginx

#### 黑白名单
- allow / deny
```
白名单设置，访问根目录 
location / {
                allow 123.34.22.155;
                allow 33.56.32.1/100;
                deny  all;
}

黑名单设置，访问根目录 
location / {
                deny 123.34.22.155;
}

特定目录访问限制
location /tree/list {
                allow 123.34.22.155;
                deny  all;
}
```
- geo module   
[geo](http://www.ttlsa.com/nginx/using-nginx-geo-method/)

- map module
[map](http://www.ttlsa.com/nginx/using-nginx-map-method/)

- map & geo 
[map+geo](http://www.linuxeye.com/configuration/2823.html)

#### 限速
- limit_conn_zone 
```
设置：
    limit_conn_zone $http_host zone=servicelimit:10m;
    limit_conn_zone $binary_remote_addr zone=iplimit:50m;
    limit_conn_log_level error;
    限制单一IP来源的连接数，同时也会限制单一虚拟服务器的总连接数：
使用：
    limit_conn servicelimit 40000;
    同一server 同一时间只允许有40000 连接

    limit_rate 200k; # 每个链接 200k/s
    对每个连接的速率限制。参数rate的单位是字节/秒，设置为0将关闭限速。 按连接限速而不是按IP限制，因此如果某个客户端同时开启了两个连接，那么客户端的整体速率是这条指令设置值的2倍。
```
- limit_req
[here](http://www.nginx.cn/446.html)

####  需求
- 通过头状态修改头
```
                set $me $scheme;
                if ( $http_x_scheme != "") {
                    set $me $http_x_scheme;
                 }
                proxy_set_header X-Scheme $me;
```

- 通过 map 来定义黑名单
```

map $args_x $black {
#   include /path/of/your/map/file;
      default 0;
      aaa   1; 
      bbb   1;
}

    if ($black == 1 ) {
        return 503 
    }
```

#### 尝试用 refere 防盗链来做拒绝请求

#### 
