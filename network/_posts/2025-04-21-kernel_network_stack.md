# 什么是内核网络栈

即常见的OSI的7层网络栈. 

网络包（packet）经过这些网络栈，进入/离开 linux.


有一堆的协议实现程序，放在内核里面。比如TCP、UDP等协议。

当一个程序发起一个tcp包，会通过socket API与内核里面的TCP栈交互，内核里的TCP栈就会将传递过来的数据，封装成TCP包，增加tcp的header，分包等动作，之后转交给其他的网络层，比如进一步处理成底层的包.

同时实现了TCP协议里面的“可靠性”相关的功能，比如按顺序传包（收包），


# 参考

网络包（packet）在linux内核网络栈中的工作流

https://www.net.in.tum.de/fileadmin/TUM/NET/NET-2024-04-1/NET-2024-04-1_16.pdf
