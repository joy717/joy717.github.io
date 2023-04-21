# 现象
k8s 使用ipvs，跨节点访问svc，频繁出现Connection reset by peer的错误

**解决**
1. 调整内核参数
```
# 临时
sysctl -w net.netfilter.nf_conntrack_tcp_be_liberal=1
# 持久化方案
## 修改 /etc/sysctl.conf, 设置net.netfilter.nf_conntrack_tcp_be_liberal=1，并且执行
sysctl -p
```
2. 解决网络包频繁出现 提高网络性能

## 原因
通过tcpdump抓包，发现很多segment没有被捕获到，同时很多RST包，以及重试的Retransmission的包。

conntrack有个配置nf_conntrack_tcp_be_liberal。默认为0.

此配置下，conntrack对于未能处理的packet（比如out of window）会标记成INVALID，并且不会做NAT处理。（访问svc ip时候，ipvs会做nat处理，转到后面的pod id）,但是会放行对应的包，

目标pod接收到这个包的时候，会发现自己不认识（因为之前都是通过svc nat过来），于是发了一个RST的包回到client端，不断重试，最终被关闭。

> nf_conntrack_tcp_be_liberal - BOOLEAN 0 - disabled (default) not 0 - enabled 
> Be conservative in what you do, be liberal in what you accept from others. If it's non-zero, we mark only out of window RST segments as INVALID.

https://github.com/kubernetes/kubernetes/issues/74839

https://technology.lastminute.com/chasing-k8s-connection-reset-issue/

**https://toutiao.io/posts/bx8f1o4/preview**
