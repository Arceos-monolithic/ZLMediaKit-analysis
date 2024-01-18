# 服务端启动时 strace 

## 进程分析

```bash
sudo ./MediaServer -d 
```
启动后，`ps -e` 找到对应进程

```bash
12888 pts/5    00:00:00 MediaServer
12891 pts/5    00:00:00 MediaServer
```

执行 `sudo strace -p 12888` 显示结果

```bash
strace: Process 12888 attached
wait4(12891,
```

看来实际运行的不是它，再执行

```bash
sudo strace -p 12891
futex(0x55f8a660b678, FUTEX_WAIT_BITSET_PRIVATE|FUTEX_CLOCK_REALTIME, 0, NULL, FUTEX_BITSET_MATCH_ANY
```

应该是在这监听。

## 启动分析

使用命令 `sudo strace -f ./MediaServer -d` 追踪 syscall 调用，其中 `-f` 追踪 mediaserver 启动后的多个进程。结果在[这里](server_strace_boot.txt)

注意，这只是启动时的 strace，还没有在另一个终端使用 `ffmpeg` 推流。

推流后产生大量的 `clock_nanosleep` 和 `epoll_wait` 等实行忙等待，之后再做讨论。