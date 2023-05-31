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

### tag为v2以及以上的情况
如果有v2以上的tag，需要go.mod里面把module修改成v2版本。
比如
```
module github.com/joy717/blog
```
修改为
```
module github.com/joy717/blog/v2
```
这种方式相当于用了一个新的go module。（如果是同时引入v1跟v2之间的struct，那v1.User与v2.User之间需要有个强制转换的动作）
引用此项目的项目B，也需要同步更改go.mod，同时还需要将代码里面的import都做修改。
这样的话，比较麻烦，可以选择在项目B的go.mod里面，添加一个replace。
比如
```
require github.com/joy717/blog v1.0.0
```
变成
```
require github.com/joy717/blog v1.0.0
replace github.com/joy717/blog v1.0.0 => github.com/joy717/blog/v2 v2.0.0
```
但这样的方式，如果第三方的依赖库也有用到同一个引用，就会将第三方的依赖版本replace掉，与第三方库期望不符。
所以最好的方式还是得把go.mod以及import所有的都修改成v2版本。

