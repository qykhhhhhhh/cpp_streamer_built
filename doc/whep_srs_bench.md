# srs拉流压测bench
## 1. 简介
cpp streamer是音视频组件，提供串流方式开发模式。

whep(webrtc egrest http protocol)拉流实现，使用组件:
* whep组件

本例实现的webrtc的whep拉流，http的路径和服务端是针对srs的webrtc服务实现，whep协议为[webrtc egrest http protocol](https://www.ietf.org/archive/id/draft-murillo-whep-03.txt)，本例的whep配合srs的http subpath路径。

### 1.1 srs的whep url格式
```
http://10.0.8.5:1985/rtc/v1/whip-play/?app=live&stream=1000
```
如上：

http方法: post

http subpath: /rtc/v1/whip-play

http 参数: app=live&stream=1000, 注意app和stream两个参数都是必须的

### 1.2 执行命令行
```
./whep_srs_bench \
 -i "http://10.0.8.5:1985/rtc/v1/whip-play/?app=live&stream=1000" \
 -n 100
```

注意：
* -i的参数为srs的webrtc的whep拉流地址，地址要加引号，如10.0.8.5是srs的地址，1985是srs的信令http端口号
* -n为并发whep session个数，也就是拉流的个数，推荐小于100，如果需要测试多余100的个数，推荐开启多个whep_srs_bench命令行，以确保准确度
