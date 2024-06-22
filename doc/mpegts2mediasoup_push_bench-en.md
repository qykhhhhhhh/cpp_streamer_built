# Read the mpegts file to do mediasoup broadcaster push bench stress test development example
## 1 Introduction
cpp streamer is an audio and video component that provides streaming development mode.

The implementation of reading mpegts files for mediasoup broadcaster streaming uses two components:
* mpegtsdemux component
* mspush component: mediasoup push component

The stress test starts multiple mspush component instances to push streams concurrently. The concurrency mode adopts the single-threaded asynchronous high-concurrency mode of libuv.

The mediasoup broadcaster of webrtc implemented in this example pushes the stream. The https path and server are implemented for the webrtc service of mediasoup. The signaling uses mediasoup broadcaster.

The implementation is as shown below

![cpp_stream exampe1](imgs/mediasoup_broadcaster_bench.png)

* Read the mpegts file first
* Use the mpegtsdemux component: the source interface imports the file binary stream, and after parsing, outputs the video + audio media stream through the sinker interface;
* Use mspush component: After the source interface imports the upstream parsed media stream, the webrtc network transmission format is encapsulated inside the component, and then sent to the webrtc server through the network. Here is the webrtc service of mediasoup. After negotiation with mediasoup broadcaster signaling, Use the mspush component for push streaming;
* Start multiple mspush component instances and push to webrtc service at the same time;

### 1.1 mediasoup broadcaster url format of srs
```
https://xxxxx.com:4443?roomId=200&userId=1000
```
As above:

http method: post

http parameters: roomId=200&userId=1000, note that both roomId and userId parameters are required

### 1.2 Execute command line
```
./mediasoup_push_bench -i webrtc.ts -o "https://xxxxx.com:4443?roomId=200&userId=1000" -n 100
```

Notice:
* The output address group must be enclosed in quotation marks.
* -n is the number of concurrent mediasoup broadcaster sessions
* mpegts file, the encoding format must be: video h264 baseline; audio opus sampling rate 48000, number of channels is 2;

Recommended ffmpeg command line to generate mpegts source files:
```
ffmpeg -i src.mp4 -c:v libx264 -r 25 -g 100 -profile baseline -c:a libopus -ar 48000 -ac 2 -ab 32k -f mpegts webrtc.ts
```

### 1.3 Defects and modifications of mediasoup server source code
The mediasoup broadcaster interface provides me with https api for signaling exchange.

But there are prerequisites: <b> The roomId where the stream is pushed must exist in advance, otherwise the broadcaster creation will fail</b>

Therefore, you need to modify the source code server.js of mediasoup-demo:
```
     expressApp.param(
         'roomId', (req, res, next, roomId) =>
         {
             queue.push(async () =>
                 {
                     consumerReplicas = 0;
                     req.room = await getOrCreateRoom({ roomId, consumerReplicas });
                     next();
                 }).catch((error) =>
                         {
                             logger.error('room creation or room joining failed:%o', error);
 
                             reject(error);
                         });
         });
```

Original code: Check whether the room with roomId exists in the http api. If it does not exist, refuse to create the broadcaster;

New code: Use the http api to detect whether the room with roomId exists. If it does not exist, create a new room with the roomId;

## 2. Code development and implementation
The code is implemented in: [src/tools/mediasoup_push_bench.cpp](../src/tools/mediasoup_push_bench.cpp)

### 2.1 Introduction to cpp streamer component interface
Each media component is accessed using an interface class, as follows:
```
class CppStreamerInterface
{
public:
     virtual std::string StreamerName() = 0;
     virtual void SetLogger(Logger* logger) = 0;
     virtual int AddSinker(CppStreamerInterface* sinker) = 0;
     virtual int RemoveSinker(const std::string& name) = 0;
     virtual int SourceData(Media_Packet_Ptr pkt_ptr) = 0;
     virtual void StartNetwork(const std::string& url, void* loop_handle) = 0;
     virtual void AddOption(const std::string& key, const std::string& value) = 0;
     virtual void SetReporter(StreamerReport* reporter) = 0;
};
```
* StreamerName: Returns a string, the unique component name
* SetLogger: Set log output. If not set, no log will be generated inside the component;
* AddSinker: Add the next hop interface of component output;
* RemoveSinker: Delete the next hop interface output by the component;
* SourceData: The interface through which the component accepts data from the previous hop;
* StartNetwork: If it is a network component, such as rtmp, webrtc whip, you need to enter the url to start the operation of the network protocol;
* AddOption: Set specific options;
* SetReporter: Set the interface for component reporting messages, such as component internal error information, or network component transmission media stream bitrate, frame rate and other information;

# 2.2 Create components
Create component code, [detailed code mediasoup_push_bench.cpp](../src/tools/mediasoup_push_bench.cpp)
```
class Mpegts2MediaSoupPushs: public StreamerReport, public TimerInterface
{
     int MakeStreamers() {
         CppStreamerFactory::SetLogger(s_logger);//Set log output
         CppStreamerFactory::SetLibPath("./output/lib");//Set the path of the component dynamic library

         tsdemux_streamer_ = CppStreamerFactory::MakeStreamer("mpegtsdemux");//Create mpegtsdemux component
         tsdemux_streamer_->SetLogger(logger_);//Set module log output
         tsdemux_streamer_->AddOption("re", "true");//Set flv's demux to parse and output according to the timestamp of the media stream
         tsdemux_streamer_->SetReporter(this);//Set message report
 
         //Create multiple mediasoup broadcaster stress test instances
         for (size_t i = 0; i < bench_count_; i++) {
             //Create mediasoup push component
             CppStreamerInterface* mediasoup_pusher = CppStreamerFactory::MakeStreamer("mspush");
             mediasoup_pusher->SetLogger(logger_);//Set module log output
             mediasoup_pusher->SetReporter(this);//Set message report
             tsdemux_streamer_->AddSinker(mediasoup_pusher);//Set the next hop of the mpegtsdemux component to the mspush component

             mediasoup_pusher_vec.push_back(mediasoup_pusher);
         }

         return 0;
     }

     //Start the timer and start several mspush instances each time
     void StartWhips() {
         size_t i = 0;
         for (i = whip_index_; i < whip_index_ + WHIPS_INTERVAL;i++) {
             if (i >= bench_count_) {
                 break;
             }
             std::string url = GetUrl(i);
             //Set mediasoup broadcaster destination url and libuv network loop handle, and start mediasoup session
             mediasoup_pusher_vec[i]->StartNetwork(url, loop_);
         }
         whip_index_ = i;
         if (whip_index_ >= bench_count_) {
             post_done_ = true;
         }
     }
}
```

# 2.3 mpegts file input
File reading:
```
         uint8_t read_data[10*188];
         size_t read_n = 0;
         do {
             read_n = fread(read_data, 1, sizeof(read_data), file_p);
             if (read_n > 0) {
                 InputTsData(read_data, read_n);
             }
             if (!whip_ready_) {
                 LogErrorf(logger_, "whip error and