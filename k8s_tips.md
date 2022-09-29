## 通过json patch的时候
1. 如果key包含`/`, 需要将`/`转义成`~1`
2. 如果values是个json字符串，需要将json字符串里的`"`转义成`\"`

参考：https://www.rfc-editor.org/rfc/rfc6902
