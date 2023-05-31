## 通过json patch的时候
1. 如果key包含`/`, 需要将`/`转义成`~1`
2. 如果values是个json字符串，需要将json字符串里的`"`转义成`\"`
3. 如果返回错误`the server rejected our request due to an error in our request`，需要检查一下，父节点是否存在。比如 add "/metadata/annotations/foo"，时，需要确认"/metadata/annotations"是否存在，如果不存在，需要添加
```
[
  {"op":"add","path":"/metadata/annotations","value":{}},
  {"op":"add","path":"/metadata/annotations/foo","value": "bar"}
]
```

参考：https://www.rfc-editor.org/rfc/rfc6902
