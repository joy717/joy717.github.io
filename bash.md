
### bash读取文件中的hosts，再用ssh登录到远程机器，命令未能正常执行。
现象为：只有第一个节点的部分命令得到执行
原因是：`while read`从文件中读取的时候，需要指定`< abc.txt`来读取数据流，但这个会被ssh命令给拦截。

**_错误示范_**
```
#!/bin/bash
kubectl get nodes --no-headers -o wide | grep -v master | awk '{print $6}' > hosts.txt

input="hosts.txt"

while IFS= read -r line
do      
  echo $line
  ssh $line mkdir -p "/etc/docker/certs.d/1.2.3.4:10043"
  scp /etc/docker/certs.d/1.2.3.4:10043/ca.crt root@$line:/etc/docker/certs.d/1.2.3.4:10043/ca.crt
done < "$input"
```
**_应该修改为_**
```
#!/bin/bash
kubectl get nodes --no-headers -o wide | grep -v master | awk '{print $6}' > hosts.txt

input="hosts.txt"

while IFS= read -r line
do      
  echo $line
  ssh $line mkdir -p "/etc/docker/certs.d/172.32.1.175:10043" < /dev/null
  scp /etc/docker/certs.d/172.32.1.175:10043/ca.crt root@$line:/etc/docker/certs.d/172.32.1.175:10043/ca.crt < /dev/null
done < "$input"
```
> **_注意:_**  实际上，就是在ssh这些命令之后，加上一个 `< /dev/null` 来结束ssh相关的命令
