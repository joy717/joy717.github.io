**Tun/Tap Veth-pair 解释：**

https://segmentfault.com/a/1190000041854027

https://bbs.huaweicloud.com/blogs/152596

> TUN 和 TAP 设备的区别在于，TUN 设备是一个虚拟的端到端 IP 层设备，也就是说用户空间的应用程序通过 TUN 设备只能读写 IP 网络数据包（三层），而 TAP 设备是一个虚拟的链路层设备，通过 TAP 设备能读写链路层数据包（二层）

> Tun/Tap 是一个虚拟的端对端虚拟设备，一端在内核网络协议栈，另外一端在用户空间的应用程序，使用 TUN/TAP 设备我们有机会将协议栈中的部分数据包转发给用户空间的应用程序，让应用程序处理数据包。常用的使用场景包括数据压缩、加密等功能。

![image](https://user-images.githubusercontent.com/310284/182316321-bcf0cc01-d98f-47a0-ab0e-7ca014ffc5ce.png)
