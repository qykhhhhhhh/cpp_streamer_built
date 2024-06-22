# Reading flv files for rtmp streaming development example
## 1 Introduction
cpp streamer is an audio and video component that provides streaming development mode.


The implementation of reading flv files for rtmp streaming uses two components:
* flvdemux component
* rtmppublish component

The implementation is as shown below

![cpp_stream exampe1](imgs/flv2rtmppublish.png)

* Read the flv file first
* Use the flvdemux component: the source interface imports the file binary stream, and after parsing, outputs the video + audio media stream through the sinker interface;
* Using the rtmppublish component: After the source interface imports the upstream parsed media stream, the component encapsulates the rtmp network transmission format and then sends it to the rtmp server (such as SRS service) through the network;

## 2. Code development and implementation
The code is implemented in: src/tools/flv2rtmppublish_streamer.cpp

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
Create component code, [detailed code flv2rtmppublish_streamer.cpp](../src/tools/flv2rtmppublish_streamer.cpp)
```
class Flv2RtmpPublishStreamerMgr : public CppStreamerInterface, public StreamerReport
{
     int MakeStreamers() {
         CppStreamerFactory::SetLogger(s_logger);//Set log output
         CppStreamerFactory::SetLibPath("./output/lib");//Set the path of the component dynamic library
    
         flv_demux_streamer_ = CppStreamerFactory::MakeStreamer("flvdemux");//Create flvdemux component
         flv_demux_streamer_->SetLogger(logger_);//Set module log output
         flv_demux_streamer_->SetReporter(this);//Set message report
         flv_demux_streamer_->AddOption("re", "true");//Set flv's demux to parse and output according to the timestamp of the media stream
        
         rtmppublish_streamer_ = CppStreamerFactory::MakeStreamer("rtmppublish");//Create rtmppublish component
         rtmppublish_streamer_->SetLogger(logger_);//Set module log output
         rtmppublish_streamer_->SetReporter(this);//Set message report
         rtmppublish_streamer_->StartNetwork(dst_url_, loop_handle);//Set rtmp destination url and libuv network loop handle

         //Set the next hop of the flvdemux component to the rtmppublish component
         flvdemux_streamer_->AddSinker(rtmppublish_streamer_);
         return 0;
     }
}
```

# 2.3 flv file input
File reading:
```
     uint8_t read_data[2048];
     size_t read_n = 0;
     do {
         read_n = fread(read_data, 1, sizeof(read_data), file_p);
         if (read_n > 0) {
             streamer_mgr_ptr->InputFlvData(read_data, read_n);
         }
     } while (read_n > 0);
```
Files are input through the sourceData interface of the flvdemux component:
```
     int InputFlvData(uint8_t* data, size_t data_len) {
         Media_Packet_Ptr pkt_ptr = std::make_shared<Media_Packet>();
         pkt_ptr->buffer_ptr_->AppendData((char*)data, data_len);

         flv_demux_streamer_->SourceData(pkt_ptr);//Import flv data
         return 0;
     }
```