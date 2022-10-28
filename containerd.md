k8s使用containerd作为cri，容器镜像通过dockerfile+docker来build，基础镜像为alpine
问题：

1. 文件明明存在，但提示文件`no found`或者`no such file or directory`, 原因可能是丢失lib库。
   dockerfile里面增加此链接： `RUN mkdir /lib64 && ln -s /lib/libc.musl-x86_64.so.1 /lib64/ld-linux-x86-64.so.2`
2. CMD命令的时候，应该写为 `CMD ["/mybin"]` 而不应该写作`CMD /mybin` 否则，在pod启动时候，会报`/bin/sh no such file or directory`，虽然docker以及ctr直接启动，都能正常启动。
