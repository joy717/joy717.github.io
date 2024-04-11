# Reflection（反射）

官方文档；
https://go.dev/blog/laws-of-reflection

中文翻译：
https://yangzhe.me/2018/12/29/the-law-of-reflection/

## 核心

1. 每个变量都是interface{}，存储记录了2个信息（pair），(“值”，“值对应的类型”)， 比如
```
var i int = 3。 记录的是（3, int）
```
  > One important detail is that the pair inside an interface variable always has the form (value, concrete type) and cannot have the form (value, interface type). Interfaces do not hold interface values.
  > 即如果变量是个接口（interface），记录的类型，是实际的类型，而不是接口的类型
```
fmt.Println(reflect.TypeOf(errors.New("myErr")))
// 结果不是errors.error （接口类型），而是：
// *errors.errorString 
```
3. 反射中的2个主要类型，reflect.Value与reflect.Type与上面的对应，即“值”，“值对应的类型”。
4. 所有反射，都是基于以上几点在玩的。即 “值” “值对应的类型” 之间的各种操作。

## 通过反射修改

当使用reflect.ValueOf(mystr)的时候，实际上将mystr转成interface{}，同时需要留意的是，由于是函数调用，所以mystr实际上被值copy了。

如果想要通过反射修改value的时候，需要类似`reflect.ValueOf(&mystr)`，取ptr来传参。如果不是取ptr的方式，那么当传进去的mystr只是一个值copy的副本，当副本被修改后，而原mystr并没有改变。那么这种修改反而让人疑惑。

```
mystr := "vin"
v := reflect.ValueOf(&mystr)
// 由于传参是prt，因此需要调用Elem()获取指针对应的元素。再修改。
v.Elem().SetString("vincent")
fmt.Println(mystr)
```
