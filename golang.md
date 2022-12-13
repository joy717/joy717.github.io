1. golang里面能看到这样的`~T`，其含义是“底层类型是该类型”
比如 `~int`， 代表的含义是“底层类型是int类型”。比如`type myInt int`，那么`myInt`也是属于`~int`


2. 声明一个struct变量，则只是copy对应的struct。
比如
```
type user struct{
  name string
}
y := user{name: "hello"}
x := y
xaddr := &x
xaddr.name = "hola"
```
此时想要修通过xaddr，去修改里面的内容，实际上只会修改到x的，y里面的无法被修改。若希望修改y里面的内容，需要一开始就取y的地址
比如
```
type user struct{
  name string
}
y := user{name: "hello"}
x := &y
xaddr.name = "hola"
```

3. golang泛型判断类型. (“手动擦除类型”)
```
type T interface{
  int | int32...
}

func thisIsAFunc() {
  var t T
  switch realType := interface{}(t).(type){
  case int:
    fmt.Println(realType)
  }
...
}
```
