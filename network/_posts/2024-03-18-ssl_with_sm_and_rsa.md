## 背景
有国密需求的场景，可能需要https的ssl证书使用sm2等国密加密算法。
## 问题
基于国密的ssl，只能使用支持国密ssl的浏览器访问，如果使用chrome等不支持国密算法，而是支持RSA、ECC等算法的国际浏览器（客户端），那么网站的https会无法访问。
由于引申出一个需求：一个网站同时支持多种ssl证书对应到一个host。
## 解决方案
参考：https://www.wosign.com/Docdownload/sm2_ssl_installation_guide-linux.pdf

或者：https://cloud.tencent.com/document/product/400/47360
## 原理
本质上，使用改造后的openssl（wotrus），支持ssl有多个证书。

怀疑，类似SNI（https://zh.wikipedia.org/wiki/%E6%9C%8D%E5%8A%A1%E5%99%A8%E5%90%8D%E7%A7%B0%E6%8C%87%E7%A4%BA）

即在握手时，告诉服务端，自己使用什么加密方式的证书、域名等。（只是猜测）

## 其他
阿里有自己开源了一个ssl： [铜锁](https://github.com/Tongsuo-Project/Tongsuo)

暂时可作为openssl的上位替代品


