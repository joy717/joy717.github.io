---
layout: post
author: joy717
---

**tcp详解**

seq number记录的是client/server各自的序列号。

ack number记录的是从server/client(对方)，当前已接收的数据大小总和。（接收到SYC/FIN包时，会将seq+1作为自己的ack number发送给对方。）



https://packetlife.net/blog/2010/jun/7/understanding-tcp-sequence-acknowledgment-numbers/

https://blog.csdn.net/answer3lin/article/details/84780514

例子详解

https://www.golinuxcloud.com/tcp-sequence-acknowledgement-numbers/


**tcp 三次握手 四次挥手**

https://blog.csdn.net/qzcsu/article/details/72861891
