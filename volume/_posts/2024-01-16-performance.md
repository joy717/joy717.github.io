# 性能测试

## fio
```
# 随机4k读
fio -filename=/dev/emcpowerb -direct=1 -iodepth 1 -thread -rw=randread -ioengine=psync -bs=4k -size=20G -numjobs=50 -runtime=180 -group_reporting -name=rand_100read_4k
# 随机4k写
fio -filename=/dev/emcpowerb -direct=1 -iodepth 1 -thread -rw=randwrite -ioengine=psync -bs=4k -size=20G -numjobs=50 -runtime=180 -group_reporting -name=rand_100write_4k

# 顺序4k读
fio -filename=/dev/emcpowerb -direct=1 -iodepth 1 -thread -rw=read -ioengine=psync -bs=4k -size=20G -numjobs=50 -runtime=180 -group_reporting -name=sqe_100read_4k 
# 顺序4k写
fio -filename=/dev/emcpowerb -direct=1 -iodepth 1 -thread -rw=write -ioengine=psync -bs=4k -size=20G -numjobs=50 -runtime=180 -group_reporting -name=sqe_100write_4k 
```
