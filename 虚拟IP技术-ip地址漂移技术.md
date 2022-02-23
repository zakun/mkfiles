## 虚拟IP技术-ip地址漂移技术
### 虚拟IP
  在 TCP/IP 的架构下，所有想上网的电脑，不论是用何种方式连上网路，都必须要有一个唯一的 IP-address。事实上IP地址是主机硬件地址的一种抽象，简单的说，MAC地址是物理地址，IP地址是逻辑地址。
  
  虚拟IP，就是一个未分配给真实主机的IP，也就是说对外提供服务器的主机除了有一个真实IP外还有一个虚IP，使用这两个IP中的任意一个都可以连接到这台主机。
  
  虚拟IP一般用作达到HA(High Availability)的目的,比如让所有项目中数据库链接一项配置的都是这个虚IP，当主服务器发生故障无法对外提供服务时，动态将这个虚IP切换到备用服务器。
### 虚拟IP原理
  ARP是地址解析协议，它的作用很简单，将一个IP地址转换为MAC地址，然后给传输层使用。
  每台主机中都有一个ARP高速缓存，存储同一个网络内的IP地址与MAC地址的对应关 系，以太网中的主机发送数据时会先从这个缓存中查询目标IP对应的MAC地址，会向这个MAC地址发送数据。操作系统会自动维护这个缓存。
  
  在Linux下可以使用arp命令操作ARP高速缓存。
  
  比如存在物理机A(IP是192.168.192.54 )和物理机器B(IP是192.168.192.40)，A作为对外服务的主服务器(比如数据库主库)，B作为备份机器，两台服务器之间的通信是通过 Heartbeat，即主服务器会定时的给备份服务器发送数据包，告知主服务器服务正常，当备份服务器在规定时间内没有收到主服务器的 Heartbeat，就会认为主服务器宕机，则备份服务器就会升级为主服务器。假设物理机A的ARP缓存如下：
  
  ![1](https://img2018.cnblogs.com/blog/885859/201908/885859-20190825225619249-1735742106.png)
  
  另外物理机器B(IP是192.168.192.40)的ARP缓存如下：
  
  ![2](https://img2018.cnblogs.com/blog/885859/201908/885859-20190825225631629-422632752.png)
 
  当机器B通过BeatHeart得知机器A对外服务质量低于预期的时候(比如发生故障，服务无响应)，会将自己的ARP缓存发送出去，让路由器修改 路由表，告知虚拟地址应该指向我(物理机器B,192.168.192.40),这时候，外界再次访问虚拟IP的时候，机器B会变成主服务器，而A降级为 备份服务器。这就完成了主从机器的自动切换，这一切对外界是透明的。
### 用LVS来分析过程
  虚IP。何为虚IP那，就是一个未分配给真实主机的IP，也就是说对外提供数据库服务器的主机除了有一个真实IP外还有一个虚IP，使用这两个IP中的 任意一个都可以连接到这台主机，所有项目中数据库链接一项配置的都是这个虚IP，当服务器发生故障无法对外提供服务时，动态将这个虚IP切换到备用主机。
  
  开始我也不明白这是怎么实现的，以为是软件动态改IP地址，其实不是这样，其实现原理主要是靠TCP/IP的ARP协议。因为ip地址只是一个逻辑 地址，在以太网中MAC地址才是真正用来进行数据传输的物理地址，每台主机中都有一个ARP高速缓存，存储同一个网络内的IP地址
  
  与MAC地址的对应关 系，以太网中的主机发送数据时会先从这个缓存中查询目标IP对应的MAC地址，会向这个MAC地址发送数据。操作系统会自动维护这个缓存。这就是整个实现 的关键。
  
  下边就是我电脑上的arp缓存的内容。
```
  (192.168.1.219) at 00:21:5A:DB:68:E8 [ether] on bond0
  (192.168.1.217) at 00:21:5A:DB:68:E8 [ether] on bond0
  (192.168.1.218) at 00:21:5A:DB:7F:C2 [ether] on bond0
```
  192.168.1.217、192.168.1.218是两台真实的电脑，
  192.168.1.217 （`主`）为对外提供数据库服务的主机。
  192.168.1.218 （`备`）为热备的机器。
  192.168.1.219为`虚IP`。
  
  大家注意红字部分，219、217的MAC地址是相同的。
  
  再看看那217宕机后的arp缓存
```
  (192.168.1.219) at 00:21:5A:DB:7F:C2 [ether] on bond0
  (192.168.1.217) at 00:21:5A:DB:68:E8 [ether] on bond0
  (192.168.1.218) at 00:21:5A:DB:7F:C2 [ether] on bond0 
```
  这就是奥妙所在。当218 发现217宕机后会向网络发送一个ARP数据包，告诉所有主机192.168.1.219这个IP对应的MAC地址是00:21:5A:DB:7F:C2，这样所有发送到219的数据包都会发送到mac地址为00:21:5A:DB:7F:C2的机器，也就是218的机器。
### 实例设置虚ip漂移 
  我们可以通过 `Keepalived` 来实现这个过程。 `Keepalived` 是一个基于 `VRRP` 协议（Virtual Router Redundancy Protocol，即虚拟路由冗余协议）来实现的`LVS（负载均衡器`）服务高可用方案，可以利用其来避免单点故障。
  
  一个 LVS 服务会有2台服务器运行 `Keepalived`，一台为主服务器（`MASTER`），另一台为备份服务器（`BACKUP`），但是对外表现为一个虚拟IP，主服务器会发送特定的消息给备份服务器，当备份服务器收不到这个消息的时候，即主服务器宕机的时候，备份服务器就会接管虚拟IP，这时就需要根据 `VRRP` 的优先级来选举一个 `backup` 当 `master`，保证路由器的`高可用`，继续提供服务，从而保证了高可用性。
#### 先来准备两台机器，IP地址如下：
```
lc1: 172.24.8.101
lc7: 172.24.8.107
```
  我们现在要实现添加一个虚IP：`172.24.8.150`，当 lc1 机器正常时，`172.24.8.150` 指向 lc1，当 lc1 出现故障时指向 lc7。
  
  此时通过 ping 可以看到 `172.24.8.150` 是无法 ping 通的。
  
  在这两台机器上分别安装 `Keepalived`
```
$ sudo yum install -y keepalived
```
#### 配置 Keepalived
- lc1 的配置
```
$ cat keepalived.conf
  vrrp_instance VI_1 {
  state MASTER
  interface enp7s0f0
  virtual_router_id 51
  priority 101
  advert_int 1
  authentication {
  auth_type PASS
  auth_pass 123456
  }
  virtual_ipaddress {
  172.24.8.150
  }
}
```
- lc7 的配置　　
```
vrrp_instance VI_1 {
  state MASTER
  interface enp7s0f0
  virtual_router_id 51
  priority 100
  advert_int 1
  authentication {
  auth_type PASS
  auth_pass 123456
  }
  virtual_ipaddress {
  172.24.8.150
  }
}
```
- 启动 lc1 和 lc7 上的 Keepalived 服务
```
sudo systemctl restart keepalived.service
```
- 将 Keepalived 加入开机启动项
```
sudo systemctl enable keepalived.service
```
- 测试
  
  通过 ping 172.24.8.150 发现已经可以通了。
  　　
- 查看 lc1 的 IP信息
```
$ ip addr show enp7s0f0
2: enp7s0f0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP qlen 1000
  link/ether 6c:92:bf:0d:09:47 brd ff:ff:ff:ff:ff:ff
  inet 172.24.8.101/24 brd 172.24.8.255 scope global enp7s0f0
  valid_lft forever preferred_lft forever
  inet 172.24.8.150/32 scope global enp7s0f0
  valid_lft forever preferred_lft forever
  inet6 fe80::6e92:bfff:fe0d:947/64 scope link
  valid_lft forever preferred_lft forever
```
  其中可以看到 `inet 172.24.8.150/32 scope global enp7s0f0`，说明现在 lc1 是作为虚拟IP的 `master`来运行的。
  　　
- 查看 lc7 的 IP信息
```
$ ip addr show enp7s0f0
2: enp7s0f0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP qlen 1000
  link/ether 6c:92:bf:0d:21:49 brd ff:ff:ff:ff:ff:ff
  inet 172.24.8.107/24 brd 172.24.8.255 scope global enp7s0f0
  valid_lft forever preferred_lft forever
  inet6 fe80::6e92:bfff:fe0d:2149/64 scope link
  valid_lft forever preferred_lft forever
```
  此时 lc7 中没有虚拟IP 的信息。
  　　
- 验证 Failover

  我们手动停止 lc1 上的 Keepalived 服务：

```
sudo systemctl stop keepalived.service
```
  此时 lc1 的 IP信息为：

```
$ ip addr show enp7s0f0
2: enp7s0f0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP qlen 1000
  link/ether 6c:92:bf:0d:09:47 brd ff:ff:ff:ff:ff:ff
  inet 172.24.8.101/24 brd 172.24.8.255 scope global enp7s0f0
  valid_lft forever preferred_lft forever
  inet6 fe80::6e92:bfff:fe0d:947/64 scope link
  valid_lft forever preferred_lft forever
```
  可以看到 lc1 已经不在有 虚拟IP 的信息了。
  　　
- 查看 lc7 的 IP信息：
```
ip addr show enp7s0f0
2: enp7s0f0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP qlen 1000
  link/ether 6c:92:bf:0d:21:49 brd ff:ff:ff:ff:ff:ff
  inet 172.24.8.107/24 brd 172.24.8.255 scope global enp7s0f0
  valid_lft forever preferred_lft forever
  inet 172.24.8.150/32 scope global enp7s0f0
  valid_lft forever preferred_lft forever
  inet6 fe80::6e92:bfff:fe0d:2149/64 scope link
  valid_lft forever preferred_lft forever
```
  可以看到 `lc7` 的 IP信息中 已经有虚拟IP `172.24.8.150` 的信息了。

  此时如果再把 `lc1` 上的 `Keepalived` 启动，可以看到 虚拟IP 又重新绑定到了 `lc1` 上。
 