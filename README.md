[README_中文](./README-CH.md)
# cpp_streamer
cpp streamer is an audio and video component developed based on C++11. Users can connect the components together to implement their own streaming media functionality.

Supports various media formats, streaming live/RTC protocols.
* flv mux/demux
* mpegts mux/demux
* rtmp publish/play
* srs whip
* srs whip bench(srs webrtc bench)
* mediasoup whip(mediaoup webrtc bench)

For network development, it utilizes the high-performance, cross-platform libuv network asynchronous library.

## Usage Instructions.
cpp streamer is an audio-video component that offers streaming development mode.

For example：The implementation of converting FLV files to MPEG-TS is as shown in the following diagram.

![cpp_stream flv2mpegts](doc/imgs/flv2mpegts.png)

* First, read the FLV file;
* Using the flvdemux component: import the file binary stream through the source interface, parse it, and then output the video + audio media stream through the sinker interface;
* Using the mpegtsmux component: import the parsed media stream from the upstream through the source interface, encapsulate it into MPEG-TS format internally in the component, and then output it in MPEG-TS format through the sinker interface；
* Output through the sinker interface of the mpegtsmux component, write to a file, and obtain an MPEG-TS file；

## application examples

* [FLV to MPEG-TS Conversion](doc/flv2mpegts-en.md)
* [FLV to RTMP Streaming](doc/flv2rtmp-en.md)
* [MPEG-TS to WHIP (WebRTC HTTP ingest protocol) conversion, pushing stream to SRS WebRTC service](doc/mpegts2whip_srs-en.md)
* [MPEG-TS to WHIP (WebRTC HTTP ingest protocol) conversion bench testing, stress testing the stream pushing to the SRS WebRTC service.](doc/mpegts2whip_srs_bench-en.md)
* [Stress testing MPEG-TS to mediasoup broadcaster streaming push](doc/mpegts2mediasoup_push_bench-en.md)

