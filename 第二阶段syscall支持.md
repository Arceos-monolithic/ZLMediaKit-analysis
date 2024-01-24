# 第二阶段

支持运行 `./MediaServer -d &` 命令以及 `ffmpeg`。这个命令可以将 ZLMediaKit 作为守护进程在后台启动，并连接 `ffmpeg` 真正实现推流功能。

## 待分配的任务

### 从启动到等待连接

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
     - ~~`/usr/lib/ssl/cert.pem`~~ 这个不一定需要。如果找不到这个文件，ZLMediaServer 会去找同目录下的 `default.pem`，也就是在本地编译后的 `ZLMediaKit/release/linux/Debug/default.pem` 这个位置。

6. `prctl` 需要支持设置线程名：
   
   - 如 `prctl(PR_SET_NAME, "stamp thread"`，相当于每个线程存一个字符串

7. `ioctl` 支持修改 `FIONBIO`：
   
   - 如 `ioctl(4, FIONBIO, [1])` 表示修改 4 这个 fd 为非阻塞读写，`ioctl(5, FIONBIO, [0])` 表示修改 5 这个 fd 为阻塞读写。它等价于 open 时的 `O_NONBLOCK` 或者 fcntl 的 `F_SETFL`，可以参考这俩去实现。“设置为非阻塞读写”意味着当用户用 read 去读这个文件（实际上是个设备 `/sys/devices/system/cpu/online`）却没读到的时候，应该立即返回 EAGAIN，而不是卡在这里去切换其他进程。

8. `SCHED_FIFO` 调度以及相关 syscall：
   
   - 什么是 `SCHED_FIFO`？
     给没上过 OS 课的同学说一下，在这种调度框架下，每个进程有一个优先级，而每个优先级都对应一个队列。在同队列内，每个进程轮流运行；在不同队列之间，优先级高的必定抢占优先级低的，除非高优先级的进程退出或者主动让出。
     以 Linux 为例，优先级最高是 99 最低是 1。假设四个进程 $a,b,c,d$ 的优先级分别是 $99,99,5,1$。那么 a 和 b 轮流运行。除非有一天 a 和 b 都调用 `sys_sleep`，那此时轮到 c 运行，一个时钟周期后内核又回到 a 和 b。假设此时 a,b 都没醒，那么还是只有 c 可以运行，a,b,d 不能运行。假设此时 a 醒了 b 没醒，那么此时只有 a 可以运行，b,c,d 不能运行。
   - MediaServer 进程执行了什么操作？
     
     ```c
     sched_get_priority_min(SCHED_FIFO) = 1
     sched_get_priority_max(SCHED_FIFO) = 99
     sched_setscheduler(7560, SCHED_FIFO, [99]) = 0
     ```
     
     它首先通过 `sched_get_priority_min` 和 `sched_get_priority_max` 获取了内核允许的最大和最小优先级（一般就设置为和 Linux 一致，从 1 到 99），然后通过 `sched_setscheduler` 设置了自己的优先级。我们需要实现这三个 syscall。

9. 线程绑定到核： `sched_setaffinity` 和 `sched_getaffinity`：
   
   - 这个 syscall 可以指定将某个线程绑定到某些具体 cpu 执行。如 `sched_setaffinity(7560, 128, [0])` 表示将 7560 线程绑定到 0 号 cpu。注意第二个和第三个参数本质上是 bitset，有空可以根据[其他资料](https://blog.popkx.com/linux-multi-cpu-programming-specifying-cpu-sched_setaffinity-and-sched_getaffinity-for-threads/)写个基于 musl-libc 的小测例实验一下。
   - 注意事项：
     - fork 出的进程也遵循父进程的设定。
     - ZLMediaKit 的行为是创建与核数相等的线程，然后每个线程各自绑定一个核。所以尽管 syscall 运行每个线程绑定到一个“集合”，但你**可以实现为线程要么不绑定，要么绑定到某一个唯一确定的核**。
     - `sched_getaffinity` 也要实现。在我这的例子是 `sched_getaffinity(0, 4096, [0, 1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19]) = 32`，不过最终测试的实机上核心数可能要少一些。

10. `socket` 的 IPV6 模式：
    
    - ZLMediaKit 默认使用 IPv6：`socket(AF_INET6, SOCK_STREAM, IPPROTO_TCP) = 64`。在内核里参照目前的网络栈加个 ipv6 支持难度应该不大？实在不行再去改 ZLMediaKit 配置看能否改 ipv4。
    - 还有与之相关的一系列 syscall。具体流程是：
      
      ```c
      socket(AF_INET6, SOCK_STREAM, IPPROTO_TCP) = 64
      setsockopt(64, SOL_SOCKET, SO_REUSEADDR, [1], 4) = 0
      ioctl(64, FIONBIO, [1])           = 0
      fcntl(64, F_GETFD) = 0
      fcntl(64, F_SETFD, FD_CLOEXEC)    = 0
      setsockopt(64, SOL_IPV6, IPV6_V6ONLY, [0], 4) = 0
      bind(64, {sa_family=AF_INET6, sin6_port=htons(554), sin6_flowinfo=htonl(0), inet_pton(AF_INET6, "::", &sin6_addr), sin6_scope_id=0}, 28) = 0
      listen(64, 1024)                  = 0
      getsockname(64,{sa_family=AF_INET6, sin6_port=htons(554), sin6_flowinfo=htonl(0), inet_pton(AF_INET6, "::", &sin6_addr), sin6_scope_id=0}, [128 => 28]) = 0
      getpeername(64, 0x55fa52f82770, [128]) = -1 ENOTCONN (Transport endpoint is not connected)
      ```
      
      之后就是把它扔进 epoll 里不断等待连接了。这里涉及 ipv6 的有 `setsockopt` `bind` 和 `getsockname`。可以找找 libc-test 有没有相关测例可以用。

11. OOM 模式：
    应用会试图打开 `/proc/sys/vm/overcommit_memory` 文件并读取其中的信息。这个文件包含了“内核如何处理内存溢出情况”的信息。它比较复杂且不必要，我们不用实现，读取时直接默认返回一个 `"0"` 即可（注意不是返回空，是一个 ASCII 字符 `0`）
    
    ```c
    openat(AT_FDCWD, "/proc/sys/vm/overcommit_memory", O_RDONLY|O_CLOEXEC)             = 11
    read(110, "0", 1)                 = 1
    ```

### 建立连接

以下是建立连接后发生的 syscall 调用以及需要的支持。想要在机器上看到这一部分的功能，至少需要先完成 `ffmpeg` 的启动和推流。

接收并建立连接，主要涉及 `accept` 和 `recvfrom` `sendmsg` 中的 ipv6 问题：

- 首先是建立连接，设置 fd 相关的参数
  
  ```c
  accept(64, {sa_family=AF_INET6, sin6_port=htons(57660), sin6_flowinfo=htonl(0), inet_pton(AF_INET6, "::ffff:127.0.0.1", &sin6_addr), sin6_scope_id=0}, [128 => 28]) = 109
  ioctl(109, FIONBIO, [1])          = 0
  setsockopt(109, SOL_TCP, TCP_NODELAY, [1], 4) = 0
  setsockopt(109, SOL_SOCKET, SO_SNDBUF, [262144], 4) = 0
  setsockopt(109, SOL_SOCKET, SO_RCVBUF, [262144], 4) = 0
  setsockopt(109, SOL_SOCKET, SO_LINGER, {l_onoff=0, l_linger=0}, 8) = 0
  fcntl(109, F_GETFD)               = 0
  fcntl(109, F_SETFD, FD_CLOEXEC)   = 0
  getsockname(109, {sa_family=AF_INET6, sin6_port=htons(554), sin6_flowinfo=htonl(0), inet_pton(AF_INET6, "::ffff:127.0.0.1", &sin6_addr), sin6_scope_id=0}, [128 => 28]) = 0
  getpeername(109, {sa_family=AF_INET6, sin6_port=htons(57660), sin6_flowinfo=htonl(0), inet_pton(AF_INET6, "::ffff:127.0.0.1", &sin6_addr), sin6_scope_id=0}, [128 => 28]) = 0
  ```
- 然后扔进 epoll
  
  ```c
  epoll_ctl(6, EPOLL_CTL_ADD, 109, {events=EPOLLIN|EPOLLOUT|EPOLLERR|EPOLLHUP|EPOLLEXCLUSIVE|EPOLLET, data={u32=109, u64=109}}) = 0
  epoll_wait(6, [{events=EPOLLIN|EPOLLOUT, data={u32=109, u64=109}}], 1024, 5) = 1
  ```
- 等待响应后，开始接收推流
  
  ```c
  recvfrom(109, "OPTIONS rtsp://127.0.0.1:554/liv"..., 262144, 0, 0x7fd1677faa30, [128 => 0]) = 87
  sendmsg(109, {msg_name=NULL, msg_namelen=0, msg_iov=[{iov_base="RTSP/1.0 200 OK\r\nCSeq: 1\r\nDate: "..., iov_len=279}], msg_iovlen=1, msg_controllen=0, msg_flags=MSG_DONTWAIT|MSG_NOSIGNAL}, MSG_DONTWAIT|MSG_NOSIGNAL)            = 279
  recvfrom(109, "ANNOUNCE rtsp://127.0.0.1:554/li"..., 262144, 0, 0x7fd1677faa30, [128 => 0]) = 626
  ```
  
   一部分 recvfrom 和 accept 的输出是 EAGAIN，表示资源不可用。这是正常的，单纯只是因为其他没建立连接的线程在抢同个文件。顺便一提，前几次 recvfrom 正常收到的信息内容分别是 `OPTIONS` `ANNOUNCE` `SETUP` `RECORD`，随后才开始正常传输，不过这属于 rtsp 协议的内容了，内核不需要关心。
- 最后传输完成后 `shutdown`，这个目前 Starry 应该已经有了

### 检查输出

我们需要一项机制去检查内核是否实现了 ZLMidiaKit 需要的功能，且使其正常工作。显然，`return 0` 不能算是一个程序正常运转的
依据。

ZLMediaKit 会将直播流存成 .ts 文件，保存在自己的一个文件中。例如 `openat(AT_FDCWD, "/home/ZLMediaKit/release/linux/Debug/www/live/test/2024-01-19/16/03-56_0.ts", O_WRONLY|O_CREAT|O_TRUNC, 0666) = 110`。每个 ts 文件长度比较短，而且内容不全，需要修改 ZLMediaKit 让他保留完整文件。

> 在文件夹外还有一个 `openat(AT_FDCWD, "/home/ZLMediaKit/release/linux/Debug/www/live/test/hls.m3u8", O_WRONLY|O_CREAT|O_TRUNC, 0666) = 110`，不过合并 ts 文件的时候不一定需要这个 m3u8

之后，我们还需要把文件从 Starry 上拉到本机上，正常播放，以证明 ZLMediaKit 输出了正确的文件。目前立即想到有两种思路：

- 比较优雅的方式：在 Starry 内部支持某种类似 scp 的指令，在运行时把文件拖出来
- 更简单直接的方式：在播放完成后关机，此时文件会保存在文件系统镜像上。写一个脚本把它挂载到本机。就可以把文件复制出来了

之后可以用 ffmpeg 合并 ts 文件并演示。或者其实直接播放 .ts 也是可以的。