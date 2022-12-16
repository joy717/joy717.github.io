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
或者：
```
func thisIsAFunc() {
  var t T
  switch realType := any(t).(type){
  case int:
    fmt.Println(realType)
  }
...
}
```
4. slice header
slice实际上是这么个struct：
```
type notInHeapSlice struct {
	array *notInHeap //指向底层array的某个元素的指针
	len   int // slice长度
	cap   int // slice上限
}
```
当将一个slice作为参数传递的时候，传递的是sliceHeader这么个struct的copy，

这意味着，如果只是修改slice已有元素的话，能够将这些“改变”反馈给调用者。

但**如果对slice增加元素等操作，比如使用append()，这有可能导致slice的改变无法传递到调用者。如果明确要增加元素等操作，需要将slice的指针作为参数传递。**

> append() 当发现超过capacity的时候，会创建一个新的底层array，将原有的array元素copy到新的array上去。这样会导致sliceHeader指向的`array`改变。
> 进而导致预期slice的修改无法生效的情况。




