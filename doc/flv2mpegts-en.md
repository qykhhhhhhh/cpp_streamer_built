# flv to mpegts
## 1. Introduction
cpp streamer is an audio-video component that provides a streaming development mode。


Implementation of converting FLV files to MPEG-TS using two components:
* flvdemux component;
* mpegtsmux components;

The implementation is as shown in the following diagram

![cpp_stream exampe1](imgs/flv2mpegts.png)

* First, read the FLV file.
* Utilize the flvdemux component: Import the file binary stream via the source interface, parse it, and then output the video + audio media stream through the sinker interface.
* Use the mpegtsmux component: Import the parsed media stream from the upstream through the source interface, encapsulate it into MPEG-TS format internally within the component, and then output it in MPEG-TS format through the sinker interface.
* Output through the sinker interface of the mpegtsmux component, write to a file, and obtain an MPEG-TS file.

## 2. Code development implementation
code file: src/tools/flv2rtmppublish_streamer.cpp

### 2.1 Introduction to Component Interfaces.
Each media component is accessed using an interface class, as shown below:
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
* StreamerName: Returns a string, the unique name of the component.
* SetLogger: Sets the logging output. If not set, the component does not generate internal logs.
* AddSinker: Adds the next-hop interface for the component's output.
* RemoveSinker: Removes the next-hop interface for the component's output.
* SourceData: Interface for receiving data from the upstream component.
* StartNetwork: If it is a network component, such as RTMP or WebRTC's WHIP, it requires a URL input to start the network protocol operation.
* AddOption: Sets specific options for the component.
* SetReporter: Sets the interface for reporting messages from the component, such as internal error messages or bitrate and frame rate information when transmitting media streams through network components.

# 2.2 Creating Components
code: [flv2rtmppublish_streamer.cpp](../src/tools/flv2rtmppublish_streamer.cpp)
```
class Flv2TsStreamerMgr : public CppStreamerInterface, public StreamerReport
{
    int MakeStreamers() {
        CppStreamerFactory::SetLogger(s_logger);//Setting Logging Output
        CppStreamerFactory::SetLibPath("./output/lib");//Setting the Path for Component Dynamic Libraries
    
        flv_demux_streamer_ = CppStreamerFactory::MakeStreamer("flvdemux");//Setting the Path for Component Dynamic Libraries
        flv_demux_streamer_->SetLogger(logger_);//Setting Module Logging Output
        flv_demux_streamer_->SetReporter(this);//Setting Message Reporting
        
        ts_mux_streamer_ = CppStreamerFactory::MakeStreamer("mpegtsmux");//创建mpegtsmux的组件
        ts_mux_streamer_->SetLogger(logger_);//Setting Module Logging Output
        ts_mux_streamer_->SetReporter(this);//Setting Message Reporting
        
        flv_demux_streamer_->AddSinker(ts_mux_streamer_);//The flvdemux component object is configured to have the downstream component as mpegtsmux.
        ts_mux_streamer_->AddSinker(this);//The mpegtsmux component is configured to have the downstream (writing MPEG-TS file).
        return 0;
    }
}
```

# 2.3 FLV file input
File Reading:
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
The file is input through the sourceData interface of the flvdemux component.：
```
    int InputFlvData(uint8_t* data, size_t data_len) {
        Media_Packet_Ptr pkt_ptr = std::make_shared<Media_Packet>();
        pkt_ptr->buffer_ptr_->AppendData((char*)data, data_len);

        flv_demux_streamer_->SourceData(pkt_ptr);
        return 0;
    }
```

# 2.4 mpegts File Output
File Output
```
class Flv2TsStreamerMgr : public CppStreamerInterface, public StreamerReport
{
    int MakeStreamers() {
        //......
        flv_demux_streamer_->AddSinker(ts_mux_streamer_);//The flvdemux component object is configured to have the downstream component as mpegtsmux.
        //......
        return 0;
    }
public:
    //Receiving data from the sinker output interface of the mpegmux component
    virtual int SourceData(Media_Packet_Ptr pkt_ptr) override {
        FILE* file_p = fopen(filename_.c_str(), "ab+");
        if (file_p) {
            fwrite(pkt_ptr->buffer_ptr_->Data(), 1, pkt_ptr->buffer_ptr_->DataLen(), file_p);
            fclose(file_p);
        }
        return 0;
    }

}
```
