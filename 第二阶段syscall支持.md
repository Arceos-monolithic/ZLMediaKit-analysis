# 第二阶段

支持运行 `./MediaServer -d &` 命令以及 `ffmpeg`。这个命令可以将 ZLMediaKit 作为守护进程在后台启动，并连接 `ffmpeg` 真正实现推流功能。

## 待分配的任务

1. 在文件镜像中加入 `ffmpeg`，并支持简单启动。
   - MediaServer 启动后会执行 `execve("/bin/sh", ["sh", "-c", "which ffmpeg"], 0x7ffc153105d0`，也即询问内核 "ffmpeg 在哪"。**如果 Starry 上 busybox 的 which 指令有处理不对的地方，也应尽量完善**

2. 支持 `sys_rseq`：
   - rseq 是一个比较新的 syscall，扩展了 Linux 的锁机制。大体上来说，它可以让用户注册一段“临界区代码”，如果某个线程在其中执行时被调度了，下次再回到这个线程时就要从头开始。中文文档比较少，[这里](https://zhuanlan.zhihu.com/p/103960889)有一个，代码建议直接参考 [Linux 源码的这一段](https://github.com/torvalds/linux/blob/master/kernel/rseq.c)。
   - 这个功能对大家来说比较难，如果后续评估认为它太复杂可以绕过

3. 完善 `sys_dup2` 和 `sys_dup3`：
   - 目前 `ulib/axstarry/syscall_fs/src` 中有 `syscall_dup3`，它实际上实现的内容是 `dup2`。我们需要把它改名成 `dup2` （记得看 syscall 编号），然后添加一个真正的 `dup3`。这个不难，只是多一个参数

4. 完善 `futex`：
   - 在第一阶段任务中，已经添加了一个 `FUTEX_WAKE_PRIVATE` 参数。但实际运行 ZLMediaKit 时需要更多的参数，例如 `FUTEX_WAIT_BITSET_PRIVATE` `FUTEX_CLOCK_REALTIME` `FUTEX_BITSET_MATCH_ANY` 等，因此需要在 Starry 中做出一个相对完善的 `futex` 实现。

5. 文件镜像中添加其他必需文件
   - 第一阶段中已经添加了一堆动态库，例如 `/lib/x86_64-linux-gnu/libssl.so.3`，但实际运行时我们还需要更多，比如：
    - `/usr/lib/ssl/openssl.cnf`
    - `/usr/lib/ssl/cert.pem`

6. 还没分析完，待续