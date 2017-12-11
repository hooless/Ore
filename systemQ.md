#### 操作系统启动顺序
```
1.启动Bios 自动检测，获取硬件信息，在此后，计算机就知道读取哪个设备了
2.读取 MBR 主引导记录，硬盘0 扇区，512byt ，启动信息、分区信息，复制到 0x7c00 的物理内存地址
3.启动 boot loader (grub），初始化硬件设备、建立内存空间映射图，为调用操作系统内核做准备 
4.加载内核，调用 start_kernel 函数启动一系列初始化，完成内核环境的建立
5.启动 init ，inittab 确定系统允许级别， 初始化系统
6.启动 sysinit ,配置系统基础环境
7.启动内核模块
8.根据 rc.d 来启动服务和服务的初始化
9.rc.local
10. 执行 /bin/login ，进入等待登录状态
```


#### proc 的作用
```
个人理解: 用户空间和内核空间的接口
```

#### 进程间通信
[点击这里](http://blog.csdn.net/gatieme/article/details/50908749)

####  守护进程
[点击这里](http://www.ruanyifeng.com/blog/2016/02/linux-daemon.html)
[点击这里](http://www.cnblogs.com/mickole/p/3188321.html)

#### 僵尸进程
[点击这里](http://www.cnblogs.com/yuxingfirst/p/3165407.html)
[点击这里](http://blog.csdn.net/LEON1741/article/details/78142269)


#### UDP/TCP
- SYN_SEND
```
客户端尝试链接服务，通过 open 方法，也就是 TCP 三次握手中的第一步之后。 客户端状态
sysctl-w  net.ipv4.tcp_syn_retries, 作为客户端可以设置 syn 包的充实次数，默认 5 次(大约 180s ）。“"仅仅重试 2 次，现代网络够了“
```
- SYN_RECEIVED
```
服务接受创建请求的 syn 后，也就是 TCP 三次握手中的第 2 步，发送 ACK 数据包之前。 服务端状态
一般 15 个左右正常，如果很大，怀疑遭受 syn_flood 攻击
sysctl-w net.ipv4.tcp_max_syn_backlog=4096, 设置改状态的等待队列数，默认 1024 ，调大后可适当防止 syn_flood 
sysctl-w net.ipv4.tcp_syncookies=1, 打开 syncookies ，在 syn backlog 不足的时候，提供临时机制将 syn 链接唤出
sysctl-w net.ipv4.tcp_synack_retries=1, 作为服务端返回 ACK 的重试次数，默认 5 次(大约 180s)，““”仅仅 重试两次，现代网络够了“
```
- ESTABLISHED
```
客户端接受到服务端的 ACK 包后的状态，服务器在发出 ACK 在一定时间后即为 establish 
sysctl-w net.ipv4.tcp_keepalive_time=120 , 默认为 7200 s(2小时)，系统针对空闲链接会进行心跳检查，如果超过 net.ipv4.tcp_keepalive_probes * net.ipv4.tcp_keepalive_intvl = 默认 11 分钟，终止对应的 tcp 链接，可适当调整心跳检查频率
```
- FIN_WAIT1
```
主动关闭的一方，在发出 FIN 请求后，也就是TCP 四次断开的第一步
```
- CLOSE_WAIT
```
被动关闭的一方，在接收到客户端的 FIN 后，也就是 TCP 四次断开后的第二步
```
- FIN_WAIT2
```
主动关闭的一方，在接收到被动关闭的一方的 ACK 后，也就是 TCP 四次断开的第二步
sysctl-w net.ipv4.tcp_fin_timeout=30，可以设定被动关闭方返回 FIN 后的超时时间，有效回收链接，避免 syn_flood
```
- LAST_ACK
```
被动关闭一方，在发送 ACK 后一段时间后(确保客户端已经收到)，再发起一个 FIN 请求，也就是  TCP 四次握手中的第三次
```
- TIME_WAIT
```
主动关闭的一方，在收到被动关闭的 FIN 包后，发送 ACK 。 也就是 TCP 四次握手的第 4 步
sysctl-w net.ipv4.tcp_tw_recycle, 打开快速回收 TIME_WAIT 
sysctl-w net.ipv4.tcp_tw_reuse=1, 快速回收并重用 TIME_WAIT 的链接，貌似和 tw_recycl 有冲突，不能重用就回收？
net.ipv4.tcp_max_tw_buckets: 处于 time_wait 状态的最多连接数，默认180000
```
- 一些扩展   
[tim_wait 过多](http://blog.csdn.net/yusiguyuan/article/details/21445883)   
[三次握手/四次断开](http://blog.csdn.net/whuslei/article/details/6667471)   
[sml时间](http://blog.51cto.com/wushank/1135060)   


