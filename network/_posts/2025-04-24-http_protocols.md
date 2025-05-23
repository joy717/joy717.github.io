# http各个版本之间的区别

* HTTP1.0 单个请求，结束就关闭链接
* HTTP1.1 单个请求结束后，链接不关闭，后续请求可以复用这个链接. 但带来队头阻塞问题.
> 管道，可以实现请求端“同时”（这边指的是，响应没回来之前）发起，但响应端，必须依照请求的先后顺序，将响应内容完整传输回来，才能继续下一个请求的响应。
* HTTP2 在http层，加了数据帧，数据帧内包含了这块数据属于哪个stream，这样响应的时候，就可以将完整数据拆分开，不同的响应数据，可以交叉放到链接通道内，传输回client端。
> 但这个只是在http层解决了队头阻塞的问题，在tcp层，并不知道这个数据帧，因此当某个tcp包丢失了，依然会导致整个通道内的数据包被阻塞。（因为tcp要求数据包是有顺序的，必须按顺序接收）
* HTTP3 （quic）利用udp协议，在udp协议里面加了数据帧的概念，这就在传输层，解决队头阻塞的问题，同时，由于udp没有强制要求数据包必须按顺序接收，因此也就不阻塞了。
> 实际上，为了保证数据的可靠性，quic自己实现了tcp协议相关的内容，也就是保证数据是要按顺序接收，可以说是tcp2.0.
> 但也正是tcp2.0，所以在单个请求（资源）内，依然要求包是顺序的（tcp的要求），因此在单个请求里，依然有队头阻塞的情况。只不过不再影响其他的资源了。

有意思的是：

HTTP1.1 版本开始，为了解决队头阻塞，浏览器允许对于单个域名，同时发起6个左右的tcp链接. 极大降低了队头阻塞的问题。

后续的各种版本虽然奔着解决队头阻塞问题，但实际上对于队头阻塞并不能有效缓解，有时候不如加大链接数量。

这是因为丢包的时候，往往是丢了一段（TCP/UDP层），这一段里面极大可能包含着多个请求里面的一些数据块（比如123123123->123xxx123 其中的xxx为丢失部分）. 也就变成是，大部分的资源请求因为前面有块数据丢失，导致对于单个请求来说，依然是队头阻塞的.


---


HTTP（HyperText Transfer Protocol）是互联网上应用最广泛的协议之一，自1991年诞生以来经历了多个版本的迭代。以下是HTTP 1.0、HTTP 1.1、HTTP/2和HTTP/3的主要区别及核心改进：

---

### **1. HTTP 1.0（1996年）**
- **基本特性**：
  - **短连接**：每个请求/响应需要建立一个新的TCP连接，完成后立即关闭，高延迟。
  - **无状态**：每次请求独立，服务器不保留上下文信息。
  - **基础功能**：支持GET、POST、HEAD方法，无压缩、缓存机制简单。
- **问题**：
  - 频繁的TCP连接和关闭导致性能低下（三次握手开销大）。

---

### **2. HTTP 1.1（1999年）**
- **核心改进**：
  - **持久连接（Keep-Alive）**：默认复用TCP连接，减少重复握手开销。
  - **管道化（Pipelining）**：允许在同一个连接上发送多个请求（但响应必须按顺序返回，易阻塞）。
  - **分块传输（Chunked Transfer）**：支持流式传输，无需预先知道内容长度。
  - **缓存控制**：引入`Cache-Control`、`ETag`等头部，优化缓存策略。
  - **Host头**：支持虚拟主机（一个IP托管多个域名）。
- **遗留问题**：
  - 队头阻塞（Head-of-Line Blocking）：管道化中若前一个请求延迟，后续请求会被阻塞。
  - 头部冗余：每次请求携带大量重复头部（如Cookie）。

---

### **3. HTTP/2（2015年）**
- **核心改进**：
  - **二进制协议**：取代文本格式，解析更高效。
  - **多路复用（Multiplexing）**：在单一连接上并行传输多个请求/响应，彻底解决队头阻塞。
  - **头部压缩（HPACK）**：减少冗余头部数据。
  - **服务器推送（Server Push）**：服务器可主动推送资源（如CSS/JS）到客户端缓存。
  - **流优先级**：允许设置请求优先级，优化关键资源加载。
- **问题**：
  - 仍依赖TCP协议，可能因TCP层丢包或拥塞导致性能下降。

---

### **4. HTTP/3（2022年正式标准化）**
- **革命性变化**：
  - **基于QUIC协议**：取代TCP，运行在UDP之上，解决TCP的队头阻塞问题。
  - **零RTT连接**：对重复连接可跳过握手，显著降低延迟。
  - **多路复用增强**：即使单个数据包丢失，其他流仍可继续传输。
  - **内置加密**：QUIC默认使用TLS 1.3，安全性更高。
  - **连接迁移**：网络切换（如WiFi转4G）时连接不中断。
- **优势**：
  - 在高丢包、高延迟网络（如移动环境）下性能显著优于HTTP/2。

---

### **版本对比总结**
| 特性                | HTTP 1.0       | HTTP 1.1               | HTTP/2                 | HTTP/3                 |
|---------------------|---------------|-----------------------|------------------------|------------------------|
| **连接方式**         | 短连接         | 持久连接               | 多路复用               | QUIC多路复用           |
| **协议层**           | TCP           | TCP                   | TCP                    | UDP（QUIC）            |
| **队头阻塞**         | 严重           | 管道化部分解决(请求端解决，响应端未解决)         | 应用层解决，TCP层仍有  | 彻底解决               |
| **头部压缩**         | 无             | 无                    | HPACK                  | QPACK                  |
| **服务器推送**       | 不支持         | 不支持                | 支持                   | 支持                   |
| **加密**             | 可选           | 可选                  | 主流强制HTTPS          | 默认加密（TLS 1.3）    |
| **典型延迟**         | 高             | 中等                  | 低                     | 极低（尤其移动网络）   |

---

### **应用场景建议**
- **老旧系统**：仍可能使用HTTP 1.1。
- **现代Web**：HTTP/2适合大多数场景（需HTTPS）。
- **高要求环境**：HTTP/3（如视频流、实时通信）优先，但需客户端和服务端支持（如Cloudflare、Google已部署）。

HTTP/3代表了未来方向，但全面普及仍需时间（需操作系统、浏览器、服务器生态支持）。

**这篇解释更详细，强烈建议多次通读**

https://github.com/rmarx/holblocking-blogpost/blob/master/README_CN.md


# 概念

* **多路复用** 在一个通道内，可以交叉传输不同的数据。比如文件1、文件2同时传输的时候，多路复用为：在一个通道内，12121212 或者 112211221122 这种方式 来传输数据
