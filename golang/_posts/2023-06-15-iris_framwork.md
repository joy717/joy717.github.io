---
title: Iris Framework
---

# 反向代理相关

## 背景

有时候希望经过本身服务转发请求给上游，也就是本身服务作为反向代理服务器。

client（浏览器等）→ 反向代理服务器 → 上游服务

## 问题：client请求的时候，响应为出错，有类似ERR_CONTENT_LENGTH_MISMATCH字眼

### 排查

经过排查，该问题的直接原因是：response的返回值的数据大小与声明的http header `Content-Length`不符。

通过debug排查发现，上游返回的数据大小与`Content-Length`相符，

经过代理服务器返回给client的时候，iris日志里面显示 `abort Handler` 以及 `http: wrote more than the declared Content-Length` 等错误。

也就是代理服务器转发给前端的时候，数据大小超过了`Content-Length`。

为什么数据的大小会改变呢？

通过排查发现，一般iris会设置压缩，也就是gzip，导致传送给client的数据大小与`Content-Length`声明的不符。

由于上游返回给代理服务器的时候，没有使用压缩，同时也设置了`Content-Length`，因此代理服务器转发给client的时候，会直接使用`Content-Length`的header，但实际数据却经过了压缩。

### 解决

去掉压缩。即：

```
// 不使用以下语句
app.Use(iris.Compression)
```

**或者**

代理服务器接收到上游数据后，返回数据给client之前，将`Content-Lenght`修改

```
...
	proxy := httputil.NewSingleHostReverseProxy(target)
	proxy.ModifyResponse = func(response *http.Response) error {
		response.Header.Del("Content-Length")
		return nil
	}
...
```

由于代理服务器发送数据给client的时候，还会设置Content-Length，因此我们直接在接收到上游数据的时候，将`Content-Length`的header去掉。


# iris 路由设置

## 问题

一般情况下，iris mvc设置路由规则以这种方式实现：

```
func (c *myController) BeforeActivation(b mvc.BeforeActivation) {
	b.Handle(http.MethodPost, "/list", "List")
	b.Handle(http.MethodPost, "/fileServer", "FileServer")
}
```

由于上游服务器有自己多种的http method（get/post/put等），也有自己的URL路径，如果将反向代理服务器，作为iris mvc的一个普通路由，会有以下问题：

1. 如何设置多种http method？
2. 如何一个路由同时匹配 /fileServer/xxxxx/yyy，/fileServer/aaaa/bbbb/ccc 等路径？

## 解决

```
func (c *vmRecoveryPointController) BeforeActivation(b mvc.BeforeActivation) {
	b.HandleMany("ANY", "/fileServer/*any", "FileServer")
}
```
其中`"ANY"` 也可以写作`"ALL"` 代表的是，所有的Http Method。 如果有特殊需求，比如只希望匹配GET/POST/PUT， 可以写作`"GET POST PUT"`

`"/fileServer/*any"` 中的`*any`代表匹配任意的路径。 这边的意思是匹配 `/fileServer/`以及任意的子路径，但不匹配 `/fileServer`本身
