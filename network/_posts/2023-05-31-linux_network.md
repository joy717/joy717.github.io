**网络**

![image](https://github.com/user-attachments/assets/260f1352-3c29-458f-aaec-e8ff084233ab)

图片来源：https://miro.medium.com/v2/resize:fit:4800/format:webp/1*eFMXAAUb4__ZKGEBgS1aDg.png



一个比较不错的系列教程

https://bbs.huaweicloud.com/blogs/109721

**Tun/Tap Veth-pair 解释：**

https://segmentfault.com/a/1190000041854027

https://bbs.huaweicloud.com/blogs/152596

> TUN 和 TAP 设备的区别在于，TUN 设备是一个虚拟的端到端 IP 层设备，也就是说用户空间的应用程序通过 TUN 设备只能读写 IP 网络数据包（三层），而 TAP 设备是一个虚拟的链路层设备，通过 TAP 设备能读写链路层数据包（二层）

> Tun/Tap 是一个虚拟的端对端虚拟设备，一端在内核网络协议栈，另外一端在用户空间的应用程序，使用 TUN/TAP 设备我们有机会将协议栈中的部分数据包转发给用户空间的应用程序，让应用程序处理数据包。常用的使用场景包括数据压缩、加密等功能。

![image](https://user-images.githubusercontent.com/310284/182316321-bcf0cc01-d98f-47a0-ab0e-7ca014ffc5ce.png)


**bridge+VLAN 作网络隔离**

eth0网卡添加一个子设备eth0.100，配置VLAN100，将eth0.100加入到br0上，其他桥接在此br0上的设备，就都加入到此VLAN中，

此时，eth0相当于是交换机，会做tag添加/移除处理，根据不同的VLAN ID，发往eth0.x设备。

<img width="1058" alt="image" src="https://user-images.githubusercontent.com/310284/184836591-ea907d4c-289e-40b0-9e2b-f1227e7520e7.png">



可参考此文的VLAN部分

https://opengers.github.io/openstack/openstack-base-virtual-network-devices-bridge-and-vlan/



**linux内核网络协议栈**

https://zhuanlan.zhihu.com/p/379915285

**tcp详解**

seq number记录的是client/server各自的序列号。

ack number记录的是从server/client(对方)，当前已接收的数据大小总和。（接收到SYC/FIN包时，会将seq+1作为自己的ack number发送给对方。）



https://packetlife.net/blog/2010/jun/7/understanding-tcp-sequence-acknowledgment-numbers/

https://blog.csdn.net/answer3lin/article/details/84780514

例子详解

https://www.golinuxcloud.com/tcp-sequence-acknowledgement-numbers/


**ip route命令**

```
# ip route 
default via 172.31.12.1 dev br_bond0_44
10.233.117.0/24 via 172.31.12.8 dev tunl0 proto bird onlink 
```
`via`的有意思是路由的下一跳应该去往172.31.12.8这个ip对应的mac地址。

`路由`：会将dest mac地址替换成目的路由的mac地址。走二层网络到路由。
