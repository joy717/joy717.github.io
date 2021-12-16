module 代码目录：
$GOPATH/pkg/mod

module 下载缓存目录：
$GOPATH/pkg/mod/cache
list获取的`.info`，`.mod`文件存放的地方。

go.mod 只校验依赖是否正常，不会下载真正的代码.（也就是，只体现在cache目录里）
当代码里面用到依赖包的时候，才会真正把代码下载下来放在mod目录里。
