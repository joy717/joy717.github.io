# 相关概念

## pod

k8s集群调度的最小单位，一个pod由一个或者若干个容器组成。一个pod内的容器共享网络。

容器是一个环境，类似“沙盒”的概念，将应用程序隔离在特定的环境下运行。其原理是使用linux kernel的特性，namespace作资源隔离（网络、文件等），cgroup作资源限制（限制该环境下可用的资源大小，比如cpu、内存）。


## service

一个逻辑上的概念，概念上等同于LB（LoadBalancer）。主要为了负载均衡以及流量转发。

访问service，实际访问的是service的后端对应的pod，使用iptables/ipvs将访问vip的流量转发到后端的Pod。

背景：

由于k8s内，pod每次重启之后，ip会变化，因此k8s使用service来固化访问地址。服务之间调用的时候，统一使用service的ip。lb的后端由k8s的组件动态的调整。

> 一般使用的时候，实际使用的是service的域名（domain），由k8s内部的dns服务提供dns查询
>
> domain格式为： SVC_NAME.NS_NAME.svc



# 整体网络

目前主要涉及的网络，分成2个部分。

1、pod与pod之间的网络，由具体的cni实现，目前使用的cni是calico。

2、service与pod部分，由kube-proxy实现，目前我们使用的是kube-proxy的ipvs模式。


## pod到pod

cni是k8s抽象出来的网络接口层，负责pod到pod之间的网络。

cni只定义了一些接口行为，只要实现了这些接口的，都可以作为k8s的网络插件使用。

目前主要有Flannel，Calico，Cilium等实现，我们目前使用的是Calico的vxlan模式。

### 同节点

![](https://github.com/joy717/joy717.github.io/assets/310284/ce672c91-2d84-4b49-b73e-dc6554b09fd7)

以这个Pod为例
```
[root@k8s-1 ~]# kubectl get pods centos-1 -o wide
NAME                          READY   STATUS    RESTARTS   AGE   IP               NODE    NOMINATED NODE   READINESS GATES
centos-1   1/1     Running   0          15h   10.233.114.176   k8s-1   <none>           <none>
```

容器内有一张eth0的网卡，对应host（宿主机，以下简称host）有一张对应的calixxxx的网卡，这两张网卡通过veth-pair连接起来。

```
# 容器内
[root@k8s-1 ~]# kubectl exec centos-1 -- ip link
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
3: eth0@if204: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1430 qdisc noqueue state UP mode DEFAULT group default
    link/ether 92:de:d0:8f:9a:1b brd ff:ff:ff:ff:ff:ff link-netnsid 0
 
# 宿主机
[root@k8s-1 ~]# ip link | grep -A 1 "204:"
204: cali232bd664ad4@if3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1430 qdisc noqueue state UP mode DEFAULT group default
    link/ether ee:ee:ee:ee:ee:ee brd ff:ff:ff:ff:ff:ff link-netnsid 16
```

pod内的默认路由为169.254.1.1的网关，但此网关ip实际不存在。

```

[root@k8s-1 ~]# kubectl exec centos-1 -- ip route
default via 169.254.1.1 dev eth0
169.254.1.1 dev eth0 scope link
```

calico将calixxxx的proxy_arp(`/proc/sys/net/ipv4/conf/calixxxx/proxy_arp`)打开，因此pod内访问169.254.1.1的时候，calixxxx会做ARP响应，将自己的mac地址返回给pod。即calixxxx成为了pod的默认网关。

host上所有的pod都有一张calixxxx的网卡，路由表内，每一张calixxxx的网卡都有一条路由记录（ip route），将pod ip跟对应网卡“绑定”起来。(也正因为这些路由规则，所以实际上calixxxx的mac地址并不重要)

```
[root@k8s-1 ~]# ip route
default via 172.31.13.1 dev br_bond0_45
10.233.77.64/26 via 10.233.77.64 dev vxlan.calico onlink
10.233.81.192/26 via 10.233.81.192 dev vxlan.calico onlink
blackhole 10.233.114.128/26 proto 80
10.233.114.129 dev caliac61247354c scope link
10.233.114.168 dev cali925697ccade scope link
10.233.114.169 dev cali6332045ecc3 scope link
10.233.114.170 dev calie1bbc3ec1eb scope link
10.233.114.171 dev cali03edef3d831 scope link
10.233.114.175 dev cali6e2f0fd53fe scope link
10.233.114.176 dev cali232bd664ad4 scope link # 我们关心的Pod
10.233.114.177 dev caliab1996a0cfe scope link
10.233.114.178 dev cali229ab65b360 scope link
10.233.114.181 dev cali9d3714ebc6f scope link
10.233.114.182 dev cali08f4200b4c6 scope link
```

因此本节点内pod到pod的网络，从pod1内的eth0出来，经过对应的calixxxx的网卡，再通过host的路由表，往对应的caliyyyy的网卡上走，最终进入pod2的eth0。



### 跨节点

![](https://github.com/joy717/joy717.github.io/assets/310284/3aac95e5-9882-48d7-a998-d3c188292130)

每个节点有张网卡vxlan.calico，这个相当于是该节点的calico网关，所有跨节点的pod请求，都会经过这个"网关"（通过查询host的main路由表），之后通过overlay（这边使用vxlan），数据包外面包了一层host的ip，走host的网络，到达节点2，解包，再根据节点2上main路由表目标Pod的路由，进到目标Pod里面。

```
# 从节点1的Pod 10.233.114.178 访问节点2的Pod10.233.81.239
# source节点1
[root@k8s-1 ~]# ip route
default via 172.31.13.1 dev br_bond0_45
10.233.77.64/26 via 10.233.77.64 dev vxlan.calico onlink
10.233.81.192/26 via 10.233.81.192 dev vxlan.calico onlink #匹配这条路由，下一跳从vxlan.calico出去
 
# iptables nat表：
-A cali-POSTROUTING -o vxlan.calico -m comment --comment "cali:e9dnSgSVNmIcpVhP" -m addrtype ! --src-type LOCAL --limit-iface-out -m addrtype --src-type LOCAL -j MASQUERADE
 
# target节点2
# 到了节点2后，解包，之后匹配以下路由，到达对应的Pod
[root@k8s-2 ~]# ip r | grep '10.233.81.239'
10.233.81.239 dev cali738a11cf833 scope link
```


calico有很多iptables规则，具体的规则解析可以参考 https://cloud.tencent.com/developer/article/1482739

## service到pod

service到pod的网络主要由k8s的组件kube-proxy处理。

service本身是一个lb，kube-proxy负责将pod添加到service(lb)的后端，以及在节点上做相应的lb转发。我们使用的ipvs模式。

kube-proxy在节点上会创建iptables+ipvs规则，通过这个方式做lb的vip与lb后端的映射。

在iptables部分，PREROUTING@nat  OUTPUT@nat  将要访问service的包，做标记（主要的chain是KUBE-SERVICES）

```
-A KUBE-MARK-MASQ -j MARK --set-xmark 0x4000/0x4000
```

之后，在POSTROUTING@nat的时候，如果有该标记，则做一次MASQ

```
-A KUBE-POSTROUTING -m comment --comment "kubernetes service traffic requiring SNAT" -j MASQUERADE
```

当节点接收到lb请求的时候，进入INPUT chain，经过ipvs，做dnat处理，之后经过OUTPUT，最后经过POSTROUTING做masq。

再往后面的，即访问pod的ip地址，也就是交给cni处理了。


由于使用的是ipvs模式，会在每个节点上创建一个kube-ipvs0的网卡，上面绑定了所有service的vip(ip addr show kube-ipvs0)，但此网卡ARP是禁用的，即（NOARP，同时网卡也是DOWN的，因为不希望也不应该通过此网卡做ARP响应，因为每个节点上都有此网卡，以及对应的ip地址，如果开启arp响应，则会导致冲突）

```
[root@k8s-1 ~]# ip a show kube-ipvs0
14: kube-ipvs0: <BROADCAST,NOARP> mtu 1500 qdisc noop state DOWN group default 
    link/ether 76:ee:bd:d5:6a:b4 brd ff:ff:ff:ff:ff:ff
    inet 10.233.61.203/32 scope global kube-ipvs0
       valid_lft forever preferred_lft forever
    inet 10.233.26.114/32 scope global kube-ipvs0
       valid_lft forever preferred_lft forever
    inet 10.233.28.61/32 scope global kube-ipvs0
       valid_lft forever preferred_lft forever
...
```

由于k8s的service ip实际并不是一个真正的ip地址，而是k8s节点上的一个vip，因此节点之外，其实是无法直接访问service ip的。

k8s提供NodePort以及LoadBalancer的两种类型的service，提供集群之外的网络访问。



### 集群外访问

#### NodePort类型的service

通过iptables规则，将本地ip+指定端口段的流量，经过service的ipvs转发到对应的pod后端上去。

主要的chain还是KUBE-SERVICES

每个节点上都“监听”同一个端口，而且与ip地址无关。

```
-A KUBE-SERVICES -m addrtype --dst-type LOCAL -j KUBE-NODE-PORT
-A KUBE-NODE-PORT -p tcp -m comment --comment "Kubernetes nodeport TCP port for masquerade purpose" -m set --match-set KUBE-NODE-PORT-TCP dst -j KUBE-MARK-MASQ
```

```
# 通过这个ipset设置“监听”端口
ipset list KUBE-NODE-PORT-TCP
```

此种方式的弊端是，会抢占节点的端口，多个服务无法使用同一个端口。而且端口数量及其有限，因此k8s提供了另外一种暴露服务的方式：LoadBalancer

#### LoadBalancer类型的service

service上绑定一个“公网ip”（也可以是伪公网），以及对应的端口。当访问该“公网ip:端口”时候，通过iptables+ipvs转发到pod后端上去

k8s原生并没有LoadBalancer类型的service实现，即缺少LB ip池子的管理，LB ip的分配，ip的arp响应等功能。因此我们使用了MetalLB，一个基于k8s的BareMetal的lb工具。

MetalLB的主要做的事情是，ip池子管理，ip分配，ip的arp响应等

我们已知，k8s service上的ip，实际上是个节点上可见的虚拟的ip，只要让这个ip在节点之外"可见"，也就解决了集群服务暴露给外部访问的问题。

MetalLB在每个节点上有一个服务，每个服务负责本节点的lb的ip。

当网关arp请求过来，虽然kube-ipvs0上有lb的ip地址，但该网卡是NOARP的，因此没有人告诉网关该ip的mac地址。

MetalLB会监听ARP请求，如果发现是自己负责的ip，则会做相应的ARP响应，剩下的，全部交给k8s的kube-proxy，即iptables/ipvs部分
