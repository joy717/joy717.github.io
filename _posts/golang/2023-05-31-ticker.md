如果想要实现类似timer的东西，不建议用time.sleep+循环，
因为time.sleep()会一直增加毫秒数，最终导致跳时间。

比如要实现：
每一秒打印当前时间。
如果用time.sleep(1*time.Second) + 一个循环，
最终会发现有些时候是2s打印一次

用time.Ticker则不会出现这样的问题。
