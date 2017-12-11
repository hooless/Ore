## iptables

### iptables 结构详解
[点击这里](http://blog.51cto.com/hoolee/1387664)

### 例子
- DNAT
```
如何将 192.168.10.2 主机的 80 端口的请求转发到 172.116.10.3 8080 端口？
iptables -t nat -I PREROUTING -d 192.168.10.2 -p tcp -m tcp --dport 80 -j DNAT --to-destination 172.116.10.3:8080
```
- REDIRECT 
```
如何将本级的 80 转发到 8080 ？ 
iptables -t nat -I PREROUTING -d 192.168.10.2 -p tcp --dport 80 -j REDIRECT --to-port 8080
```
- SNAT
```
内网访问 internet ?
iptables -t nat -I PREROUTING  -s 172.16.0.0/24 -o eth1  -j SNAT -to 121.20.32.171
```
- 限速
```
丢掉每个 ip 每秒访问 80 端口超过 30 次的数据包
iptables -t filter -I INPUT -p tcp --dport --tcp-flags FIN,SYN,RST,ACK, -m conlimit --connlimit-above 30 --connlimit-mask 32 -j DROP
```

### 统计运营商的流量
```
新建表:
iptables -N %s_in" % traffic_chain
iptables -N %s_out" % traffic_chain
iptables -N %s_in_%s" % (isp, counter[isp])
iptables -N %s_out_%s" % (isp,counter[isp])

# 个个 ip 的流量倒入
iptables -I %s_in -d %s/32 -j %s_in_%s" %(traffic_chain,ip,isp,counter[isp])
iptables -I %s_out -s %s/32 -j %s_out_%s" %(traffic_chain,ip,isp,counter[isp])

倒入流量:
iptables -I INPUT -j %s_in" % traffic_chain
iptables -I OUTPUT -j %s_out" % traffic_chain
```
