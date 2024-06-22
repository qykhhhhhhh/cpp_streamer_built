# mediasoup broadcaster拉流bench压测
## 1. 简介
cpp streamer是音视频组件，提供串流方式开发模式。

mediasoup broadcaster拉流bench，使用组件:
* mspull组件: mediasoup pull拉流组件

压测启动多个mspull组件实例，并发进行拉流，并发模式采用libuv的单线程异步高并发模式。

本例实现的webrtc的mediasoup broadcaster拉流，https的路径和服务端是针对mediasoup的webrtc服务实现，信令采用mediasoup broadcaster。


### 1.1 执行命令行
```
./mediasoup_pull_bench \
 -i "https://xxxxx.com.cn:4443?roomId=100&apid=7689e48c-09ae-48ca-8973-ad5de69de5e8&vpid=aadbbb0b-2e4e-4ed8-8bd6-22e3c50b9fc1" \
 -n 100 \​
 -l 1.log
```

* -i的参数，mediasoup的broadcaster接入模式，xxxx.com:4443为sfu的域名地址，roomId为房间号，apid为推流audio的producerId, vpid为推流video的producerId；
* -n的参数为整数，测试并发的数量，推荐小于100，如果需要测试大于100的，开启多个命令行，以保证拉流的准确度；
* -l的参数为日志文件

