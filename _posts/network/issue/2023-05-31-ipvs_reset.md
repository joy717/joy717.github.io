# 现象
k8s 使用ipvs，跨节点访问svc，频繁出现Connection reset by peer的错误

**解决**
1. 调整内核参数
```
# 临时 设置内核参数net.netfilter.nf_conntrack_tcp_be_liberal=1。只有out of window的RST包才会被标记为INVALID。
sysctl -w net.netfilter.nf_conntrack_tcp_be_liberal=1
# 持久化方案
## 修改 /etc/sysctl.conf, 设置net.netfilter.nf_conntrack_tcp_be_liberal=1，并且执行
sysctl -p
```
2. 解决网络包频繁出现 提高网络性能。比如提升硬件，开启RPS(Receive Packet Steering)等

## 原因
实际原因是网络流量处理不过来。加上我们使用IPVS的方式，触发此问题。



通过tcpdump抓包，发现很多segment没有被捕获到，同时很多RST包，以及重试的Retransmission的包。

conntrack有个配置nf_conntrack_tcp_be_liberal。默认为0.



此配置下，当包处理比较慢的时候（比如流量比较大的时候），包会out of window，conntrack对于这些packet，会标记成INVALID，并且不会做NAT处理，但是会放行对应的包，

目标pod接收到这个包的时候，会发现自己不认识（因为之前都是通过svc nat过来），于是发了一个RST的包回到client端，这个包的五元组（tcp+srcIP+srcPort+destIP+destPort）是合法的，因此关闭链接。


> 访问svc ip时候，ipvs会做nat处理，转成后端的pod id

> nf_conntrack_tcp_be_liberal - BOOLEAN 0 - disabled (default) not 0 - enabled 
> 
> Be conservative in what you do, be liberal in what you accept from others. If it's non-zero, we mark only out of window RST segments as INVALID.

## 其他
如果发现重启后，发现虽然net.netfilter.nf_conntrack_tcp_be_liberal = 1 这个设置在sysctl.conf里面，但是sysctl -a | grep liberal 却是0. 也就是这个参数在重启后，没有生效。

往  /etc/modules-load.d/my_nf_conntrack.conf 设置nf_conntrack为默认module

```
echo "nf_conntrack_ipv4" > /etc/modules-load.d/ze_nf_conntrack.conf
```

原因是加载内核模块顺序的问题。nf_conntrack并非是内置的必须的模块，只有其他systemd的service依赖他的时候，才会去启动。

在nf_conntrack模块启动之前，systemd-sysctl-service被启动，此时sysctl关于nf_conntrack的参数会被忽略掉。因此不生效。

利用systemd-sysctl.service对systemd-modules-load.service的依赖，可以配置systemd-modules-load.service相关参数，保证sysctl启动的时候，相应的内核模块已经启动。

```
# systemd-analyze critical-chain systemd-sysctl.service
The time when unit became active or started is printed after the "@" character.
The time the unit took to start is printed after the "+" character.

systemd-sysctl.service +44ms
└─systemd-modules-load.service @314ms +90ms
  └─systemd-journald.socket @242ms
    └─system.slice @220ms
      └─-.slice @220ms
```

https://www.osso.nl/blog/sysctl-modules-load-order-nf-conntrack/



## 相关链接
https://github.com/kubernetes/kubernetes/issues/74839

https://technology.lastminute.com/chasing-k8s-connection-reset-issue/

**https://toutiao.io/posts/bx8f1o4/preview**
