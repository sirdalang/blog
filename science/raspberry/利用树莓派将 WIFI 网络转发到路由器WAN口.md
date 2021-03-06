# 利用树莓派将 WIFI 网络转发到路由器WAN口

[toc]

## 1 目的

树莓派的无线网卡和有线网卡作路由，无线网卡与外网（如电信路由器出口）相连，有线网卡与内网相连。

内网通过一个路由器分发。

实现内外网隔离。

## 2 概况

外网网络（局域网）：192.168.0.0/24

树莓派NAT路由：192.168.2.0/24（有线网卡） --> 192.168.0.112（树莓派的无线网络IP）

树莓派无线网卡IP：192.168.0.112

树莓派有线网卡IP：192.168.2.1

内网网络：192.168.2.0/24

内网测试电脑IP：192.168.2.2

路由器外网地址：192.168.2.2 （和电脑测试一样）

路由器NAT网络：192.168.3.0/24 （随意）

## 3 环境

1.树莓派4。

2.树莓派能够连接WIFI，WIFI连接互联网。

3.一根网线和一个路由器。

## 4 主要节点

1.树莓派和电脑互联，以树莓派作为主机，实现局域网互通。

2.配置NAT，使发送到eth0的包能够被转发到wlan0上去，从而实现网络共享。

3.测试的时候用PING，PING通了之后如果不能正常上网则检查dns设置等问题。

### 4.1 局域网互通

1.将树莓派连接至WIFI，保证可以上网。

2.树莓派与电脑直连（先不与路由器WAN口连接，方便测试）。电脑有线网卡设置IP为192.168.2.2/24，网关为192.168.2.1。树莓派有线网卡IP设置为192.168.2.1/24。

3.树莓派的路由表调整。移除eth0的默认路由表，添加对192.168.2.0/24网络的eth0转发规则。如：

route delete default dev eth0
route add -net 192.168.2.0/24 dev eth0
4.此时，局域网配置完成，PING检查是否互通。

### 4.2 启用内核包转发

其实是很重要的一步，因为内核默认是关闭这个功能的。

相关方式可以在网上查，这里提供一条。

echo "1" > /proc/sys/net/ipv4/ip_forward

### 4.3 转发规则

1.在树莓派端配置，命令如下

sudo iptables -t nat -A POSTROUTING -s 192.168.2.0/24 -o wlan0 -j SNAT --to 192.168.0.112
即，将192.168.2.0/24网络上的源数据包转发到wlan0对应192.168.1.112源IP上（描述不准确，就是一条NAT类型的转发规则）。

2.此时，PING 192.168.0.0/24网络的IP地址应通过。配置好主机的DNS服务后，外网网络地址也相应能够解析访问。

### 4.4 连接路由器

1.与连接电脑类似，将树莓派与路由器WAN口连接。路由器上网方式（也即WAN口的网络配置）设置为静态IP，与电脑上设置一致即可。路由器网络设置为192.168.3.0/24即可正常工作（不冲突即可）。

## 5 性能

实测CPU占用只能跑到25%（单核占用），不知道是不是方法问题。下行/上行速度最高约 25~30 Mbps。经过测试发现，速率与无线网络信号强度有关。

## 附1 /etc/rc.local

以下脚本直接放在启动脚本或者手动执行均可。如果路由器、树莓派总共三块网卡的IP均配置正确的话，电脑连接到路由器的WI-FI，就能PING通外网了。

```script
echo "1" > /proc/sys/net/ipv4/ip_forward

iptables -t nat -A POSTROUTING -s 192.168.2.0/24 -o wlan0 -j SNAT --to 192.168.0.112

route delete default dev eth0
route add -net 192.168.2.0/24 dev eth0

ifconfig eth0 192.168.2.1/24
```

## 附2 另一个 /etc/rc.local

找了另一个方案，不依赖于具体的外网地址

```script
echo "1" > /proc/sys/net/ipv4/ip_forward

#iptables -A FORWARD -i wlan0 -j ACCEPT
#iptables -A FORWARD -o wlan0 -j ACCEPT
iptables -t nat -A POSTROUTING -o wlan0 -j MASQUERADE

route delete default dev eth0
route add -net 192.168.2.0/24 dev eth0

ifconfig eth0 192.168.2.1/24
```