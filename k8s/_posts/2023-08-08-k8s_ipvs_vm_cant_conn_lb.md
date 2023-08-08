---
title: ipvs模式下，节点上的vm无法访问lb
category: network,k8s
---

# 现象
创建的vm，无法访问 ingress controller 上的服务。

ingress controller的ip地址在k8s-2，vm的宿主机是k8s-1，即跨节点访问

实例
以预发布环境来说，在虚机（172.31.12.23）里面，无法访问 172.32.4.130:10043 （ingress controller LB的ip）

但可以正常ping 172.32.4.130，且可以访问 172.32.4.130:7070

7070服务不是通过ingress controller暴露的，而是直接监听在物理机上的7070端口。

# 直接原因
vm所在的宿主机，能正常收到 172.32.4.130返回的数据包，但与请求时候的数据包不匹配，所以被丢弃了。



具体的抓包数据

```
[root@k8s-1 ~]# tcpdump -i any host 172.32.4.130 and port 10001 -nne
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on any, link-type LINUX_SLL (Linux cooked), capture size 262144 bytes
16:46:35.562471   P fa:e1:e4:b1:bd:00 ethertype IPv4 (0x0800), length 76: 172.31.12.23.42712 > 172.32.4.130.10001: Flags [S], seq 4080658935, win 29200, options [mss 1460,sackOK,TS val 3867159 ecr 0,nop,wscale 7], length 0
16:46:35.562504 Out fa:e1:e4:b1:bd:00 ethertype IPv4 (0x0800), length 76: 172.32.4.114.21472 > 172.32.4.130.10001: Flags [S], seq 4080658935, win 29200, options [mss 1460,sackOK,TS val 3867159 ecr 0,nop,wscale 7], length 0
16:46:35.562505 Out fa:e1:e4:b1:bd:00 ethertype 802.1Q (0x8100), length 80: vlan 44, p 0, ethertype IPv4, 172.32.4.114.21472 > 172.32.4.130.10001: Flags [S], seq 4080658935, win 29200, options [mss 1460,sackOK,TS val 3867159 ecr 0,nop,wscale 7], length 0
16:46:35.562506 Out fa:e1:e4:b1:bd:00 ethertype 802.1Q (0x8100), length 80: vlan 44, p 0, ethertype IPv4, 172.32.4.114.21472 > 172.32.4.130.10001: Flags [S], seq 4080658935, win 29200, options [mss 1460,sackOK,TS val 3867159 ecr 0,nop,wscale 7], length 0

## 这段是响应包
16:46:35.562744  In 76:bc:9a:17:45:06 ethertype 802.1Q (0x8100), length 80: vlan 103, p 0, ethertype IPv4, 172.32.4.130.10001 > 172.32.4.114.21472: Flags [S.], seq 2047737646, ack 4080658936, win 28960, options [mss 1460,sackOK,TS val 2034849122 ecr 3867159,nop,wscale 7], length 0
16:46:35.562744  In 76:bc:9a:17:45:06 ethertype 802.1Q (0x8100), length 80: vlan 103, p 0, ethertype IPv4, 172.32.4.130.10001 > 172.32.4.114.21472: Flags [S.], seq 2047737646, ack 4080658936, win 28960, options [mss 1460,sackOK,TS val 2034849122 ecr 3867159,nop,wscale 7], length 0
16:46:35.562744  In 76:bc:9a:17:45:06 ethertype IPv4 (0x0800), length 76: 172.32.4.130.10001 > 172.32.4.114.21472: Flags [S.], seq 2047737646, ack 4080658936, win 28960, options [mss 1460,sackOK,TS val 2034849122 ecr 3867159,nop,wscale 7], length 0
```

```
[root@k8s-2 ~]# tcpdump -i any host 172.32.4.130 and port 10001 -nne
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on any, link-type LINUX_SLL (Linux cooked), capture size 262144 bytes
16:46:35.562516  In 00:1e:08:0e:d5:85 ethertype 802.1Q (0x8100), length 80: vlan 103, p 0, ethertype IPv4, 172.32.4.114.21472 > 172.32.4.130.10001: Flags [S], seq 4080658935, win 29200, options [mss 1460,sackOK,TS val 3867159 ecr 0,nop,wscale 7], length 0
16:46:35.562516  In 00:1e:08:0e:d5:85 ethertype 802.1Q (0x8100), length 80: vlan 103, p 0, ethertype IPv4, 172.32.4.114.21472 > 172.32.4.130.10001: Flags [S], seq 4080658935, win 29200, options [mss 1460,sackOK,TS val 3867159 ecr 0,nop,wscale 7], length 0
16:46:35.562516  In 00:1e:08:0e:d5:85 ethertype IPv4 (0x0800), length 76: 172.32.4.114.21472 > 172.32.4.130.10001: Flags [S], seq 4080658935, win 29200, options [mss 1460,sackOK,TS val 3867159 ecr 0,nop,wscale 7], length 0
16:46:35.562516  In 00:1e:08:0e:d5:85 ethertype IPv4 (0x0800), length 76: 172.32.4.114.21472 > 172.32.4.130.10001: Flags [S], seq 4080658935, win 29200, options [mss 1460,sackOK,TS val 3867159 ecr 0,nop,wscale 7], length 0

## 这段是响应包
16:46:35.562718 Out 76:bc:9a:17:45:06 ethertype IPv4 (0x0800), length 76: 172.32.4.130.10001 > 172.32.4.114.21472: Flags [S.], seq 2047737646, ack 4080658936, win 28960, options [mss 1460,sackOK,TS val 2034849122 ecr 3867159,nop,wscale 7], length 0
16:46:35.562722 Out 76:bc:9a:17:45:06 ethertype IPv4 (0x0800), length 76: 172.32.4.130.10001 > 172.32.4.114.21472: Flags [S.], seq 2047737646, ack 4080658936, win 28960, options [mss 1460,sackOK,TS val 2034849122 ecr 3867159,nop,wscale 7], length 0
16:46:35.562723 Out 76:bc:9a:17:45:06 ethertype 802.1Q (0x8100), length 80: vlan 103, p 0, ethertype IPv4, 172.32.4.130.10001 > 172.32.4.114.21472: Flags [S.], seq 2047737646, ack 4080658936, win 28960, options [mss 1460,sackOK,TS val 2034849122 ecr 3867159,nop,wscale 7], length 0
16:46:35.562724 Out 76:bc:9a:17:45:06 ethertype 802.1Q (0x8100), length 80: vlan 103, p 0, ethertype IPv4, 172.32.4.130.10001 > 172.32.4.114.21472: Flags [S.], seq 2047737646, ack 4080658936, win 28960, options [mss 1460,sackOK,TS val 2034849122 ecr 3867159,nop,wscale 7], length 0
```

```
00:1e:08:0e:d5:85 这个是路由的mac地址

fa:e1:e4:b1:bd:00 这个是vm的mac地址

76:bc:9a:17:45:06 这个是172.32.4.130所在节点k8s-2的网卡的mac地址
```


从tcpdump里面可以看到，网络包到达k8s-2的时候，是通过00:1e:08:0e:d5:85（即通过路由）进入的k8s-2，

但网络包返回k8s-1的时候，mac地址变成了76:bc:9a:17:45:06（即k8s-2）。这样不匹配，就被丢弃了。



# 底层原因
k8s在iptables中，给所有的service（k8s的service）设置规则，会做MASQ。

从vm出来的网络包，由于目的地是k8s的LB类型service，因此被做了MASQ，此时src ip为vm所在宿主机的ip，mac地址为vm的mac地址

当响应包回来的时候，由于目的地是vm所在宿主机的ip，因此mac地址用的是vm所在宿主机的mac地址。

由于请求包与响应包的mac地址对应不上，因此网络包被丢弃了。


```
-A POSTROUTING -m comment --comment "cali:O3lYWMrLQYEMJtB5" -j cali-POSTROUTING
-A POSTROUTING -m comment --comment "kubernetes postrouting rules" -j KUBE-POSTROUTING
-A POSTROUTING -s 172.17.0.0/16 ! -o docker0 -j MASQUERADE
-A KUBE-FIREWALL -j KUBE-MARK-DROP
-A KUBE-LOAD-BALANCER -j KUBE-MARK-MASQ
-A KUBE-MARK-MASQ -j MARK --set-xmark 0x4000/0x4000
```

```
# 此处会对访问nodePort类型的service的网络包，做masq
-A KUBE-NODE-PORT -p tcp -m comment --comment "Kubernetes nodeport TCP port for masquerade purpose" -m set --match-set KUBE-NODE-PORT-TCP dst -j KUBE-MARK-MASQ

-A KUBE-POSTROUTING -m comment --comment "Kubernetes endpoints dst ip:port, source ip for solving hairpin purpose" -m set --match-set KUBE-LOOP-BACK dst,dst,src -j MASQUERADE
-A KUBE-POSTROUTING -m mark ! --mark 0x4000/0x4000 -j RETURN
-A KUBE-POSTROUTING -j MARK --set-xmark 0x4000/0x0
-A KUBE-POSTROUTING -m comment --comment "kubernetes service traffic requiring SNAT" -j MASQUERADE
```

```
# 此处会对访问LB类型的service的网络包，做masq
-A KUBE-SERVICES -m comment --comment "Kubernetes service lb portal" -m set --match-set KUBE-LOAD-BALANCER dst,dst -j KUBE-LOAD-BALANCER

-A KUBE-SERVICES ! -s 10.233.64.0/18 -m comment --comment "Kubernetes service cluster ip + port for masquerade purpose" -m set --match-set KUBE-CLUSTER-IP dst,dst -j KUBE-MARK-MASQ
-A KUBE-SERVICES -m addrtype --dst-type LOCAL -j KUBE-NODE-PORT
-A KUBE-SERVICES -m set --match-set KUBE-CLUSTER-IP dst,dst -j ACCEPT
-A KUBE-SERVICES -m set --match-set KUBE-LOAD-BALANCER dst,dst -j ACCEPT
```



# 解决方案
从vm出来的网络包，访问k8s的LB/NodePort类型serivce，不应该被MASQ。

因此解决方案就是在vm所在的节点，给iptables增加一条规则，直接ACCEPT，而不进入k8s的KUBE-POSTROUTING链。

同时，为了防止iptables规则被清，增加一个crontab任务，作为守护进程，定时检测。

由于vm实际上在任意节点，所以所有节点都应该设置。

规则为：

iptables -t nat -I POSTROUTING -s 172.31.12.0/24 -d 172.31.12.0/24 -m comment --comment "edge fix node/lb" -j ACCEPT
