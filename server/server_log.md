# 启动服务端

正常来说，可以用 `sudo ./MediaServer -d &` 将 ZLMediaKit 启动在后台，不过这样当前终端会跳 server log 比较混乱，所以我们直接 `sudo ./MediaServer -d` 启动到前台，将当前终端仅作为输出，然后另开终端操作即可。

## 本地 mp4 播放测试

#### 测试方法

启动 mediaserver 后，另一个终端用 ffmpeg 推流。

按理说应该用 ffplay 或者 vlc 之类的工具拉流，但因为我的 WSL2 网络问题没成功。不过这不影响 mediaserver 的使用

#### 测试命令

终端1输入：

```bash
sudo ./MediaServer -d
```

然后在终端2输入：

```bash
ffmpeg -re -i "01.mp4" -vcodec h264 -acodec aac -f rtsp -rtsp_transport tcp rtsp://127.0.0.1/live/test
```

等待视频播放完成，最后回到终端1查看输出如下。日志文件另存一份在[这里](server_log.txt)。

```bash
本地 mp4 播放测试：

❯ sudo ./MediaServer -d
2024-01-18 14:38:02.723 I [MediaServer] [3875-MediaServer] Factory.cpp:41 registerPlugin | Load codec: H264
2024-01-18 14:38:02.723 I [MediaServer] [3875-MediaServer] Factory.cpp:41 registerPlugin | Load codec: H265
2024-01-18 14:38:02.723 I [MediaServer] [3875-MediaServer] Factory.cpp:41 registerPlugin | Load codec: JPEG
2024-01-18 14:38:02.723 I [MediaServer] [3875-MediaServer] Factory.cpp:41 registerPlugin | Load codec: mpeg4-generic
2024-01-18 14:38:02.723 I [MediaServer] [3875-MediaServer] Factory.cpp:41 registerPlugin | Load codec: opus
2024-01-18 14:38:02.723 I [MediaServer] [3875-MediaServer] Factory.cpp:41 registerPlugin | Load codec: PCMA
2024-01-18 14:38:02.723 I [MediaServer] [3875-MediaServer] Factory.cpp:41 registerPlugin | Load codec: PCMU
2024-01-18 14:38:02.723 I [MediaServer] [3875-MediaServer] Factory.cpp:41 registerPlugin | Load codec: L16
2024-01-18 14:38:02.723 D [MediaServer] [3875-MediaServer] System.cpp:133 startDaemon | 启动子进程:3878
2024-01-18 14:38:02.723 I [MediaServer] [3878-MediaServer] System.cpp:177 systemSetup | core文件大小设置为:18446744073709551615
2024-01-18 14:38:02.723 I [MediaServer] [3878-MediaServer] System.cpp:186 systemSetup | 文件最大描述符个数设置为:1048576
2024-01-18 14:38:02.723 I [MediaServer] [3878-MediaServer] main.cpp:256 start_main | ZLMediaKit(git hash:6514be7/2024-01-15T20:34:17+08:00,branch:master,build time:2024-01-17T14:33:26)
2024-01-18 14:38:02.726 D [MediaServer] [3878-MediaServer] SSLBox.cpp:174 setContext | Add certificate of: default.zlmediakit.com
2024-01-18 14:38:02.727 D [MediaServer] [3878-stamp thread] util.cpp:366 operator() | Stamp thread started
2024-01-18 14:38:02.728 I [MediaServer] [3878-MediaServer] EventPoller.cpp:500 EventPollerPool | EventPoller created size: 20
2024-01-18 14:38:02.728 I [MediaServer] [3878-MediaServer] main.cpp:350 start_main | 已启动http api 接口
2024-01-18 14:38:02.729 I [MediaServer] [3878-MediaServer] main.cpp:352 start_main | 已启动http hook 接口
2024-01-18 14:38:02.779 I [MediaServer] [3878-MediaServer] TcpServer.cpp:218 start_l | TCP server listening on [::]: 554
2024-01-18 14:38:02.779 I [MediaServer] [3878-MediaServer] TcpServer.cpp:218 start_l | TCP server listening on [::]: 1935
2024-01-18 14:38:02.780 I [MediaServer] [3878-MediaServer] TcpServer.cpp:218 start_l | TCP server listening on [::]: 80
2024-01-18 14:38:02.780 I [MediaServer] [3878-MediaServer] TcpServer.cpp:218 start_l | TCP server listening on [::]: 443
2024-01-18 14:38:02.781 I [MediaServer] [3878-MediaServer] TcpServer.cpp:218 start_l | TCP server listening on [::]: 10000
2024-01-18 14:38:02.782 I [MediaServer] [3878-MediaServer] UdpServer.cpp:120 start_l | UDP server bind to [::]: 10000
2024-01-18 14:38:02.782 I [MediaServer] [3878-MediaServer] UdpServer.cpp:120 start_l | UDP server bind to [::]: 9000
2024-01-18 14:43:02.729 W [MediaServer] [3878-event poller 0] EventPoller.cpp:209 async_l | take time: 41ms, thread may be overloaded
2024-01-18 14:44:10.796 I [MediaServer] [3878-event poller 11] MediaSource.cpp:517 emitEvent | 媒体注册:rtsp://__defaultVhost__/live/test
2024-01-18 14:44:11.554 D [MediaServer] [3878-event poller 11] MediaSink.cpp:162 emitAllTrackReady | All track ready use 2975ms
2024-01-18 14:44:11.554 I [MediaServer] [3878-event poller 11] MediaSource.cpp:517 emitEvent | 媒体注册:fmp4://__defaultVhost__/live/test
2024-01-18 14:44:11.554 I [MediaServer] [3878-event poller 11] MultiMediaSourceMuxer.cpp:551 onAllTrackReady | stream: rtsp://127.0.0.1:554/live/test , codec info: mpeg4-generic[48000/2/16] H264[1280/720/30]
2024-01-18 14:44:11.554 I [MediaServer] [3878-event poller 11] MediaSource.cpp:517 emitEvent | 媒体注册:rtmp://__defaultVhost__/live/test
2024-01-18 14:44:11.555 I [MediaServer] [3878-event poller 11] MediaSource.cpp:517 emitEvent | 媒体注册:ts://__defaultVhost__/live/test
2024-01-18 14:44:19.169 I [MediaServer] [3878-event poller 11] MediaSource.cpp:517 emitEvent | 媒体注册:hls://__defaultVhost__/live/test
2024-01-18 14:48:04.962 I [MediaServer] [3878-event poller 11] MediaSource.cpp:517 emitEvent | 媒体注销:hls://__defaultVhost__/live/test
2024-01-18 14:48:04.962 I [MediaServer] [3878-event poller 11] MediaSource.cpp:517 emitEvent | 媒体注销:ts://__defaultVhost__/live/test
2024-01-18 14:48:04.963 I [MediaServer] [3878-event poller 11] MediaSource.cpp:517 emitEvent | 媒体注销:rtmp://__defaultVhost__/live/test
2024-01-18 14:48:04.963 I [MediaServer] [3878-event poller 11] MediaSource.cpp:517 emitEvent | 媒体注销:fmp4://__defaultVhost__/live/test
2024-01-18 14:48:04.963 I [MediaServer] [3878-event poller 11] MediaSource.cpp:517 emitEvent | 媒体注销:rtsp://__defaultVhost__/live/test
2024-01-18 14:48:04.973 W [MediaServer] [3878-event poller 11] RtspSession.cpp:62 onError | 1-170(127.0.0.1:40806) RTSP 推流器(__defaultVhost__/live/test)断开:recv teardown request,耗时(s):236
```