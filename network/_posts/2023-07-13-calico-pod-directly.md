# 背景

允许集群外部直连calico的pod id

## 注意点

* 需要k8s节点开放179端口(bgp端口)
* pod ip段需要可集群外访问，pod段的掩码必须比节点ip的掩码小（例：如果节点ip为172.31.13.27/24，那么pod ip段的掩码必须为24以下）

# 总体设计

配置pod的ip段为可直接访问的ip，利用calico的bgp模式，通过路由，达到直连的目的。

网络拓扑：将集群的节点1、节点2作为rr节点，rr节点与交换机/路由上的BGP作为邻居节点（EBGP）。集群内部的节点为一个自治的IBGP。

rr节点为bgp自治区域内的中心节点，该区域内的节点路由信息由该rr节点负责。一般为了高可用，会冗余一个rr节点。

<img width="811" alt="image" src="https://github.com/joy717/joy717.github.io/assets/310284/71d344f2-0dc3-4c3f-8f14-8592699fcac7">


本文涉及的OS均为centos7，calico版本为v3.23.3，k8s版本为v1.24.6

## 涉及的ip

| IP              |    描述     |
|-----------------|:---------:|
| 172.31.13.116   | bgp交换机ip  |
| 172.31.13.27    | 节点1 k8s-1 |
| 172.31.13.28    | 节点1 k8s-2 |
| 172.31.13.29    | 节点1 k8s-3 |
| 172.27.208.0/22 |  pod ip段  |


# 交换机配置

## 添加bgp邻居

```
# 根据不同的交换机/路由器，命令有所不同，以下为例子
neighbor 172.31.13.27 remote-as 64512
neighbor 172.31.13.28 remote-as 64512
```

> 172.31.13.27，172.31.13.28分别为集群rr节点ip（一般设置节点1，节点2为rr节点）

> 64512为对应rr节点上的AS Number


# k8s集群设置

## 修改calico的设置
```
# 修改calico的calico_backend， 设置为bird
kubectl edit cm -n kube-system calico-config

# 修改calico的ippool，设置ipipMode=Never，vxlanMode=Never，natOutgoing=false。
# natOutgoing设置为false，可以让pod到外面的请求，保留来源ip
kubectl edit ippools.crd.projectcalico.org default-pool

# 重启calico-node的pod
kubectl delete pods -n kube-system -l k8s-app=calico-node
```

## 修改bgp设置

```
calicoctl apply -f - << EOF
apiVersion: projectcalico.org/v3
kind: BGPConfiguration
metadata:
  name: default
spec:
  logSeverityScreen: Info
  nodeToNodeMeshEnabled: false #为了禁用默认的full-mesh
  asNumber: 64512 # calico集群，默认as number
EOF
```

## 创建BGP的peer

### 设置calico node的rr属性

```
calicoctl patch node k8s-1 -p '{"spec": {"bgp": {"routeReflectorClusterID": "这个可以是节点1的ip"}}}'
calicoctl patch node k8s-2 -p '{"spec": {"bgp": {"routeReflectorClusterID": "这个可以是节点2的ip"}}}'
```

### 设置k8s节点的label，方便calico peer使用

```
kubectl label node k8s-1 route-reflector=true
kubectl label node k8s-2 route-reflector=true
```

### 创建rr与交换机关联的BGPPeer

```
calicoctl apply -f - << EOF
apiVersion: projectcalico.org/v3
kind: BGPPeer
metadata:
  name: rr-peer
spec:
  nodeSelector: route-reflector == 'true'
  peerIP: 172.31.13.116 # 交换机/路由器上的BGP 地址
  asNumber: 7675 # 交换机/路由器上的BGP的AS
EOF
```

这边代表的意思是，如果node有个label是`route-reflector == 'true'`，那么将该节点与172.31.13.116交换机建立连接

### 创建普通节点与rr关联的BGPPeer

```
calicoctl apply -f - << EOF
kind: BGPPeer
apiVersion: projectcalico.org/v3
metadata:
  name: peer-with-route-reflectors
spec:
  nodeSelector: all()
  peerSelector: route-reflector == 'true'
EOF
```

这边代表的意思是，所有的节点`all()`，都与有`route-reflector == 'true'` label 的peer建立连接（实际上就是与rr节点建立连接）

## 验证
如果配置正确的话，在每个节点上，可以通过 `calicoctl node status` 查看状态

```
[root@k8s-1]# calicoctl node status
Calico process is running.

IPv4 BGP status
+---------------+---------------+-------+----------+-------------+
| PEER ADDRESS  |   PEER TYPE   | STATE |  SINCE   |    INFO     |
+---------------+---------------+-------+----------+-------------+
| 172.31.13.116 | node specific | up    | 07:52:37 | Established |
| 172.31.13.28  | node specific | up    | 07:54:35 | Established |
| 172.31.13.29  | node specific | up    | 07:54:35 | Established |
+---------------+---------------+-------+----------+-------------+

IPv6 BGP status
No IPv6 peers found.
```

> 如果在node上使用此命令，发现交换机的peer异常，检查一下是否有在交换机上，将该node设置为邻居

### 其他

此时直接访问pod ip的话，应该可以直接访问到了。

如果交换机无权限，可以建一台新的vm，通过安装配置`quagga`，来假造一个“BGP路由器”，(只做校验的话，可去掉密码)

假设集群外机器要访问pod id，此时可以设置静态路由，将pod ip段 都经过“BGP路由器”路由转发。

具体安装配置步骤，参考底下的参考资料。大体步骤为：

* 修改bgpd配置（密码，as）
* 启动zebra、bgpd服务
* 通过vtysh 设置bgp邻居

```
# 安装“BGP路由器” `quagga`的机器，需要关闭防火墙，以及允许内核ipv4转发
echo "net.ipv4.ip_forward=1" >> /etc/sysctl.conf
sysctl -p

# 关闭selinux
sed -i 's/^ *SELINUX=enforcing/SELINUX=disabled/g' /etc/selinux/config
```

```
# 集群外要访问pod的机器，添加静态路由，指向“BGP路由器”
ip route add 172.27.0.208/22 via 172.31.13.116
```


# 参考资料

[centos部署quagga](https://www.cnblogs.com/hahaha111122222/p/17223363.html)

[kubesphere-calico配置bgp](https://www.kubesphere.io/zh/blogs/calico-guide/)
