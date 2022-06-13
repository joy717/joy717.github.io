module 代码目录：
$GOPATH/pkg/mod

module 下载缓存目录：
$GOPATH/pkg/mod/cache
list获取的`.info`，`.mod`文件存放的地方。

go.mod 只校验依赖是否正常，不会下载真正的代码.（也就是，只体现在cache目录里）
当代码里面用到依赖包的时候，才会真正把代码下载下来放在mod目录里。


### 如果仓库地址与项目module不同地址，可使用replace：
go.mod里面添加
```
require github.com/myrepo v0.0.0
replace github.com/myrepo => my.io/other/myrepo master
```

之后
```
go mod tidy
```


### 如果仓库是http而不是https的话，可以设置GOINSECURE以及GOPRIVATE的环境变量，同时使用replace.
```
# 设置允许用http访问git
go env -w GOINSECURE="my.io"
go env -w GOPRIVATE="my.io"
```

go.mod里面添加
```
require github.com/myrepo v0.0.0
# 注意此处新增了.git
replace github.com/myrepo => github.com/other/myrepo.git master
```

之后
```
go mod tidy
```
