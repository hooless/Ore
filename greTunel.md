###linux 下创建GRE隧道

其他国家的互联网如同一个孤岛。要想访问国外网站异常的缓慢，甚至被和谐了。可以建立一条隧道来避免这种情况，下面说说GRE隧道如何建立。
1. GRE介绍
GRE隧道是一种IP-over-IP的隧道，是通用路由封装协议，可以对某些网路层协议的数据报进行封装，使这些被封装的数据报能够在IPv4/IPv6 网络中传输。
Tunnel 是一个虚拟的点对点的连接，提供了一条通路使封装的数据报文能够在这个通路上传输，并且在一个Tunnel 的两端分别对数据报进行封装及解封装。　一个X协议的报文要想穿越IP网络在Tunnel中传输，必须要经过加封装与解封装两个过程。
要在Linux上创建GRE隧道，需要ip_gre内核模块，它是GRE通过IPv4隧道的驱动程序。
2. 查看是否有加载ip_gre模块
```
# modprobe ip_gre
# lsmod | grep gre
ip_gre                 22432  0
gre                    12989  1 ip_gre
```
3. 创建步骤
环境如下：
host A :  121.207.22.123
host B: 111.2.33.28
在host A上面：

```
# ip tunnel add gre1 mode gre remote 111.2.33.28 local 121.207.22.123 ttl 255
# ip link set gre1 up
# ip addr add 10.10.10.1 peer 10.10.10.2 dev gre1
```
创建一个GRE类型隧道设备gre0, 并设置对端IP为111.2.33.28。隧道数据包将被从121.207.22.123也就是本地IP地址发起，其TTL字段被设置为255。隧道设备分配的IP地址为10.10.10.1，掩码为255.255.255.0。

在host B上面：
```
# ip tunnel add gre1 mode gre remote  121.207.22.123 local 111.2.33.28 ttl 255
#  ip link set gre1 up
#  ip addr add 10.10.10.2 peer 10.10.10.1 dev gre1
此时，host A 和 host B 建立起GRE隧道了。
```
4. 检测连通性
```
# ping 10.10.10.2 (host A)
PING 10.10.10.2 (10.10.10.2) 56(84) bytes of data.
64 bytes from 10.10.10.2: icmp_req=1 ttl=64 time=0.319 ms
64 bytes from 10.10.10.2: icmp_req=2 ttl=64 time=0.296 ms
64 bytes from 10.10.10.2: icmp_req=3 ttl=64 time=0.287 ms
```
5. 撤销GRE隧道
在任一一端操作下面命令
```
# ip link set gre1 down
# ip tunnel del gre1
```
