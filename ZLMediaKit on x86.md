# ZLMediaKit on x86

## 打印调试信息

通过 `./MediaServer -h` 可获取启动帮助。使用 strace 追踪后，发现值得注意的有以下参数（[原输出文件](Log1.txt)）

1. 启动时需要动态库：
   
   `openat(AT_FDCWD, "/lib/x86_64-linux-gnu/libssl.so.3", O_RDONLY|O_CLOEXEC) = 3` 
   
   `openat(AT_FDCWD, "/lib/x86_64-linux-gnu/libcrypto.so.3", O_RDONLY|O_CLOEXEC) = 3`
   
   `openat(AT_FDCWD, "/lib/x86_64-linux-gnu/libstdc++.so.6", O_RDONLY|O_CLOEXEC) = 3`
   
   `openat(AT_FDCWD, "/lib/x86_64-linux-gnu/libm.so.6", O_RDONLY|O_CLOEXEC) = 3`
   
   `openat(AT_FDCWD, "/lib/x86_64-linux-gnu/libgcc_s.so.1", O_RDONLY|O_CLOEXEC) = 3`
   
   `openat(AT_FDCWD, "/lib/x86_64-linux-gnu/libc.so.6", O_RDONLY|O_CLOEXEC) = 3`
   
   除了 libc.so 外，Starry 应该目前都没有，可以都暴力复制一份到文件镜像里。

2. `arch_prctl(ARCH_SET_FS, 0x7feda6bbf780) = 0` 在 Arceos 的 `crates/percpu/src/imp.rs:97`有相关描述，但没有落实到 syscall 层面，只是 unikernel 用。

3. `futex(0x7feda711c77c, FUTEX_WAKE_PRIVATE, 2147483647) = 0`。`FUTEX_WAKE_PRIVATE` 直接给它当成 `FUTEX_WAKE` 处理，在 `futex` 里加一句就行

4. `pipe2([3, 4], O_CLOEXEC)                = 0`。目前 Starry 只有第一个参数（俩fd），没有[pipe(2) - Linux manual page](https://man7.org/linux/man-pages/man2/pipe2.2.html)所规定的第二个参数 CLOEXEC，需要加上。这个参数在 `modules/axfs/src/api/port.rs` 里就有定义，`syscall_fcntl64` 里也有应用，依葫芦画瓢就行。

5. `clone3({flags=CLONE_VM|CLONE_VFORK, exit_signal=SIGCHLD, stack=0x7feda7606000, stack_size=0x9000}, 88) = 11122` 需要支持 CLONE_VFORK 参数。具体来说，这个 clone 结束之后应直接运行子进程并阻塞父进程，直到子进程执行 `exec` 或者 `exit` ，父进程才可继续执行。可能需要在 `struct Process`里加一个状态位，表示该进程是否因为 vfork 被阻塞，然后要执行的时候检查一下，进程进 sys_exec 和 sys_exit 的时候把父进程的这个状态位消一下。

6. `wait4(11122, [{WIFEXITED(s) && WEXITSTATUS(s) == 1}], 0, NULL) = 11122`。目前内核的 `syscall_wait4` 的 Option 只处理了 WNOHANG，没处理以上两个。看看是否需要添加，也有可能不用管也行。

7. `readlink("/proc/self/exe", "/home/ZLMediaKit/"..., 8193) = 63` 需要在 Procfs 里加一个目录 `/self/exe`，参见 `modules/axfs/src/mounts.rs: procfs` 等信息。这条调用等于是读取“当前运行的程序在哪个目录下”，需要在 `struct Process`里加入“创建进程时可执行文件在哪个目录，是什么名字”这条信息，也即把 `syscall_exec` 传入的信息保存到创建的进程中。这样才可以在访问 `readlink("/proc/self/exe"` 的时候拿到这个信息。

8. `newfstatat(AT_FDCWD, "/etc/localtime", {st_mode=S_IFREG|0644, st_size=561, ...}, 0) = 0` 

9. `prctl(PR_GET_NAME, "MediaServer")       = 0`，只需要考虑这里用到的 `PR_GET_NAME`这个参数，获取进程名，很好加的

10. `newfstatat(AT_FDCWD, "/etc/localtime", {st_mode=S_IFREG|0644, st_size=561, ...}, 0) = 0` 我在另一篇写过[在内核中添加特殊文件的大致流程](https://github.com/Arceos-monolithic/tips/blob/main/%E4%B8%BA%20Starry%20%E8%B0%83%E8%AF%95%E5%B9%B6%E6%B7%BB%E5%8A%A0%E5%8A%9F%E8%83%BD%EF%BC%9A%E4%BB%A5%20hwclock%20%E4%B8%BA%E4%BE%8B.md)

11. `openat(AT_FDCWD, "/sys/devices/system/cpu/online", O_RDONLY|O_CLOEXEC) = 3` 这个也很简单，和上面的类似。注意这之后 ZLMediaKit 会读这个文件获取系统支持的核心数，从而决定之后运行时开几个线程。

## 功能支持的时间

简单的可能需要5分钟，长的可能需要1~2小时，发现有额外需要添加的功能再讨论。**不要领一个任务做一整天**。

## 怎么支持上面的功能

1. 往 Starry 上加功能的流程是什么？看[指导书第三章](https://scpointer.github.io/rcore2oscomp/docs/lab3/intro.html)

2. 怎么知道上面说的那些参数、函数都在哪？看[第二章](https://scpointer.github.io/rcore2oscomp/docs/lab2/pos.html)

3. 怎么知道自己写的是对的？可以跑 zlmediakit 做测试。但如果只测单个 syscall，可以根据第二章指引，去[这里](https://github.com/LearningOS/2023a-stage3-proj2/tree/lab2)下载测例框架，自己写&编译一个只执行某个syscall的应用，然后让 Starry 运行。这个过程第一次可能慢一点(10分钟)，后续每次操作都不应超过2分钟。

4. 合并？自己开分支写，完成测试后往主线上合并。怎么合可以线下商量着做
