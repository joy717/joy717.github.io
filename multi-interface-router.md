# 多网卡路由设置(source-based routing, policy-based routing)

## 背景
假设我们当前有1张网卡，eth0(192.168.0.10/24, gw为192.168.0.254)，
然后，我们想要添加一张新的网卡：eth1（10.8.0.10/24，gw为10.8.0.254）

我们希望，所有经过eth1的网络流量都应该从eth1或者对应gw返回.

## 解决方案
添加一张新的路由表`isp2`：
```
echo 200 isp2 >> /etc/iproute2/rt_tables
```
将所有请求10.8.0.10的流量都导向`isp2`进行路由.(添加ip rule规则)
```
ip rule add from 10.8.0.10 table isp2
```
给这张路由表添加默认路由：
```
ip route add default via 10.8.0.254 dev eth1 table isp2
```
给这张路由表添加10.8.0.0/24网段的路由：
```
ip route add 10.8.0.0/24 dev eth1 proto static scope link src 10.8.0.10 table isp2
```

至此，访问10.8.0.10的包，会经由eth1，走路由表isp2进行路由（因为isp2表会比main表的优先级高），
1. 如果来源地址是10.8.0.0/24网段的，则直接从eth1（10.8.0.10)出去
2. 如果来源地址是非10.8.0.0/24网段，则走默认路由网关10.8.0.254
