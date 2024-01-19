# ZLMediaKit on x86

本项目的目的是为 Starry 添加新的 syscall 以及功能，以便运行应用 ZLMediaKit。

目前暂定分为以下几步：

- 第一阶段：支持运行 `./MediaServer -h` 命令。这个命令可启动 ZLMediaKit，获取启动帮助后退出程序。
- 第二阶段：支持运行 `./MediaServer -d &` 命令以及 `ffmpeg`。这个命令可以将 ZLMediaKit 作为守护进程在后台启动，并连接 `ffmpeg` 真正实现推流功能。
- 第三阶段：打通 `MediaServer` 需要的网络栈，支持向外输出流；完善 Starry 的 GUI 功能，支持在 Starry 上直接获取视频输出。