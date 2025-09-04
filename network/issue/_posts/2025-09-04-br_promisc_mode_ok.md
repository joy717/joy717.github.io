# 问题1
遇到一个比较奇怪的网络问题.

有两个k8s节点，lb svc使用metallb，把ip的arp响应放在br_bond0_45这个网桥上，svc的后端pod使用spider pool设置一个macvlan的ip，macvlan的parent也是这个网桥br_bond0_45。

cni使用的calico vxlan模式，跨节点的时候，会使用vxlan的隧道用节点ip转发数据。

当lb在节点1，而pod在节点2的时候，集群外访问lb不通，当lb与pod在同一个节点的时候，网络是通的.

但当开启tcpdump抓包的时候，发现网络通了。后续排查，发现是tcpdump默认开启了网卡的promisc mode（混杂模式）导致的。

> 混杂模式的作用是：默认网卡只会接收发给自己（mac地址匹配）的数据包，其他的包会丢掉。开启了混杂模式之后，不管目标是不是给自己的，都会接受

禁用混杂模式去抓包的情况下，发现sync包正常，sync+ack从pod返回节点1的时候，到了br_bond0_45直接被丢弃了.

网络拓扑

<img width="1766" height="1536" alt="image" src="https://github.com/user-attachments/assets/e818227e-056f-4a3b-a36d-3c198154b80a" />


排查发现，macvlan创建后，会将parent开启混杂模式。因为macvlan需要parent开启混杂模式，保证网络包能从parent转发给macvlan的网卡。

在上面的网络拓扑里，lb在另外一个节点，这个节点上没有其他的macvlan的pod，就会导致br_bond0_45没有开启混杂模式.

第一个sync包，一切正常，lb在节点1上，lb通过lvs的nat模式，将数据包转发给节点2的macvlan的pod

节点2的pod，返回sync+ack包给节点1，到了物理网卡p8p1，再到bond0.45都正常，但到了br_bond0_45的时候，数据包被丢弃了（从抓包的mac地址来看，是没错的，这边不知道为什么被丢弃了）

而当br_bond0_45开启了混杂模式之后，数据包通过了。

但从结果来看，混杂一开，网络就通，而混在就是在限制目的mac地址，不能理解。 问了ai，说是每张网卡有自己的fdb
```
bridge fdb show dev br_bond0_45
```
记录已学习的mac地址。（通常通过arp之类的方式来学习），如果不在mac地址列表里面，就会丢弃。时间有限，这块没搞明白。

# 问题2

网桥下br0下有两个网卡，一个物理网卡ens3，多个虚拟网卡vnic，veth， outer...

vnic为vm的网卡对端，veth为pod的一个服务。

他们都绑定了一个同网段的ip

从vm请求br0上的一个lb ip，该lb的后端为veth的一个pod.

同样的拓扑，在节点1、3上是正常的，在节点2就是不正常。

## 现象

数据从vm出来，经过物理网卡ens3，到br0的时候，直接被丢弃了.

对比节点的内核参数，iptables规则，没能发现有嫌疑的地方。


后续排查，偶然发现抓的包，tcpdump显示的是[P] 而不是[In]，

进一步发现，这个包不被认为是发给自己的。

看了下br0的mac地址，并没能正确的设置成ens3的mac地址，而是设置成了虚拟网卡outer的mac地址。

将br0的mac地址调整成ens3的mac地址后，网络正常.

启动网桥的时候，如果没有指定mac地址，mac地址在启动的过程中，会将mac地址最小的子接口的mac地址挂在自己的身上。

这个行为在我们这个场景下，是不合适的。因为我们会拿网桥br0设置ip地址，作为一个普通网卡使用。
