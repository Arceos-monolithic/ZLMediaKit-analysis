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

可以看到，结果和仅通过 `./MediaServer -h` 输出调试信息是类似的，只是多打开了一个 Log 文件（`openat(AT_FDCWD, "/home/ZLMediaKit/release/linux/Debug/log/2024-01-18_00.log", O_WRONLY|O_CREAT|O_APPEND, 0666) = 3`。真正处理推流的是 daemon 启动后创建的另一个进程：

```bash
clone(child_stack=NULL, flags=CLONE_CHILD_CLEARTID|CLONE_CHILD_SETTID|SIGCHLD, child_tidptr=0x7fa79f08ca50) = 17066
write(1, "\33[32m2024-01-18 16:15:03.408 D ["..., 1202024-01-18 16:15:03.408 D [MediaServer] [17063-MediaServer] System.cpp:133 startDaemon | 启动子进程:17066
) = 120
wait4(17066,
```

## 处理推流的实际进程

换成 `sudo strace -f -o log.txt ./MediaServer -d ` 命令再次执行。 `-f` 是为了监听 fork 出的其他进程， `-o` 输出到文件是因为推流后产生大量的 `clock_nanosleep` 和 `epoll_wait` 等 syscall 导致输出过多。

这次运行的输出记录在[这里](server_fork_log.txt)。在日志涉及的进程有

- 7554 是主进程， 准备好环境后 `clone3({flags=CLONE_VM|CLONE_VFORK, exit_signal=SIGCHLD, stack=0x7fd16d722000, stack_size=0x9000}, 88` 产生 7555
- 7555 和 7556 是子任务，仅为了执行 `["sh", "-c", "which ffmpeg"] ` 这一个命令。执行完之后它们就退出了
- 7557 是 7554 中产生的实际用于服务的进程。它通过
  ```bash
  clone3({flags=CLONE_VM|CLONE_FS|CLONE_FILES|CLONE_SIGHAND|CLONE_THREAD|CLONE_SYSVSEM|CLONE_SETTLS|CLONE_PARENT_SETTID|CLONE_CHILD_CLEARTID, child_tid=0x7fd16ccd7910, parent_tid=0x7fd16ccd7910, exit_signal=0, 
  ```
  创建**线程**7558。