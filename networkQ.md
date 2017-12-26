### 网络相关问题

#### dmesg 看到 ip_conntrack: table full, dropping packet , 如何解决？
```
ip_conntrack 表的最大值有参数 ip_conntrack_max 控制，查看当前最大值可以用： cat /proc/sys/net/ipv4/ip_conntrack_max 
redhat 默认大小为 65536，这个取值由机器的内存大小决定， 65536为 1G 的大小，如果内存不止 1G ，可以设置为 65536 的倍数。数值不能比当前内存可设置的最大值大，不然不生效
```

####  cc 攻击
- cc 攻击重要用来攻击页面
- 模拟多个用户不停的访问需要大量数据库操作(即需要大量cpu时间的页面)。论坛访问人越多，论坛的页面越多，数据库越复杂，被访问的频率越高，占用的资源越多
- 防御
    - 分析攻击的请求头信息，分析它的特征，然后 nginx 在七层做限制
    - 分析请求 ip ，利用 iptables 来限制 ip 
    - nginx : limit-conn、 limit-req 、black list 
- 优化
    - 将网站做成静态页面
    - 合并网页的请求
    

####  ddos 攻击
- 利用合理的请求来占用服务过多的资源，从而使得合法用户无法得到服务的相应
- 接入 cdn 
- deny 来源 ip 
- 分布式 ddos 的话 ，封掉自己的 ip

#### syn_flood
[点击](http://www.cnblogs.com/hubavyn/p/4477883.html)
- 轻量级防御
```
iptables -N syn-flood
iptables -A INPUT -p tcp -syn -j syn-flood
iptables -I syn-flood -p tcp -m limit -limit 3/s --limit-burst 6 -j RETURN
iptables -A syn-flood -J REJECT
```

#### ping 和 traceroute

#### udp 和 tcp 

####  抓 nat 包

##### 浅谈CLOSE_WAIT
[here](https://huoding.com/2016/01/19/488)
