## RTMP是怎样工作的

正如我们知道的那样，RTMP是一个基于TCP的双向数据，音频，视频通讯协议。

RTMP的工作原理是在RTMP客户端和RTMP服务器之间建立和维护一个通信途径，以实现快速可靠的数据传输。

与基于HTTP的传输协议如HLS和DASH的行为类似。RTMP也是如此，它将一个多媒体流分解成通常为64字节的音频和128字节的视频片段(fragments)。片段的大小可以 客户端和服务器之间进行协商。

传统的逻辑上认为，片段的尺寸既不能太小，也不能太大。大的片段尺寸会导致写操作的延迟，而非常小的片段会增加CPU的负载。

通过将数据流分解成片段，RTMP可以将不同数据流的片段交织在一起。
不同的流，并在一个单一的连接上传输它们。这通常被称为 
这通常被称为 "多路复用"，类似于视频广播中的统计多路复用。
然而，在实践中，包含几个片段的数据包是交错的。这使得RTMP流媒体更有效率，并允许RTMP 创建多个虚拟的、可寻址的流媒体通道。在解码器一侧。这些交错的数据包可以被解复用以检索原始的音频和视频数据。

### RTMP的连接建立-握手，连接，流化

RTMP的连接过程可以分为标题以上的三个步骤。

##### 第一步，握手

握手过程相对直接，在建立完成TCP连接后就直接开始。client和server端各自发给对方三个包，记作C0,C1,C2和S0,S1,S2。

![[7277aeea795e80657ee86d945deed35.png]]

根据以上时序图，我们来逐步讲解握手步骤:

1. 首先，client发送C0数据包给server，在这个请求中包含了RTMP协议的版本号
2. client发送C1数据包，数据包里面包含1536个随机字节的数据，这个请求发送是无需等待server是否收到C0数据包的回应的，也就是发完了C0，紧接着就发C1，无需管其他。
3.  server端这时候，肯定需要等待C0的到来，以包含版本号的S0或者1536字节随机数的S1数据包回应给client，这时候server也知道client要求的RTMP的版本了。当然了S0和S1就是相当于C0和C1的拷贝了，而不是重新生成随机数发送。
4. 然后client和server发送C2和S2。基本就是交换之前的1536个随机字节数据了。

##### 第二步，连接

连接步骤就是紧接着握手步骤之后了，其实就是client和server交换[AMF编码]([Action Message Format - Wikipedia](https://en.wikipedia.org/wiki/Action_Message_Format)) 后的消息。

AMF之前就是用来在Adobe Flash的Client和Flash的media server之间发送消息的，开发者可以用它来序列化在ActionScript和XML中的对象图。现在AMF也用在RTMP推流中，client和server之间的消息类型和消息内容的表示。

下面就是client和server交互建立连接的例子，当然AMF是二进制的，下面展示的是文本:

client发送:

```txt
(Invoke) "connect" 
(Transaction ID) 1.0 
(Object1) { app: "sample", flashVer: "MAC 10,2,153,2", swfUrl: **null**, tcUrl: "rtmpt://127.0.0.1/sample ", fpad: **false**, capabilities: 9947.75 , audioCodecs: 3191, videoCodecs: 252, videoFunction: 1 , pageUrl: **null**, objectEncoding: 3.0 }
```

server回复:

```txt
(Invoke) "_result" 
(transaction ID) 1.0 
(Object1) { fmsVer: "FMS/3,5,5,2004", capabilities: 31.0, mode: 1.0 } (Object2) { level: "status", code: "NetConnection.Connect.Success", description: "Connection succeeded", data: (array) { version: "3,5,5,2004" }, clientId: 1728724019, objectEncoding: 3.0 }
```

在连接这个步骤中，client和server也会交换并设置下Peer的带宽还有Window Ack Size这样的应答窗口大小。这一切都完成后，表明连接建立完成，然后server也可以流化视频数据了。音视频编解码还有其他详细参数可能需要参考[RTMP规范]([Adobe RTMP Specification · RTMP (veriskope.com)](https://rtmp.veriskope.com/docs/spec/))了.

##### 流化

在握手和连接都完成后，RTMP的client和server就可以真正开始工作了。为此，RTMP规范定义了下面几个commands:
- createStream
- play
- play2
- deleteStream
- closeStream
- receiveAudio
- receiveVideo
- publish
- seek
- pause

通过以上命令的功能实现，才有可能使用RTMP传输视频流。

### 音视频编码在RTMP中的支持

RTMP流中支持ACC,MP3的音频编码。视频上支持H.264/AVC，VP6编码。

### 支持RTMP的软件

主要就是FFmpeg了，当然还有其他，但是比较小众


### 代替RTMP的协议

RTMP因为跟adobe绑定，所以未来不确定。HLS或者Http Live Streaming是一个热门的替代选择。

## 使用FFmpeg来进行RTMP推流的教程

![[Pasted image 20220726150956.png]]

上图反应了RTMP的server和client的关系，RTMP server这样的流媒体服务器就是分发RTMP流到各个设备上的，然后采集端，client编码RTMP流，把数据推送到server上。

然后后面我们分步骤讨论。

##### 第一步，编码并推流

首先就是把音视频数据编码并将数据通过RTMP协议推送到server端

```bash
ffmpeg -re -i crowdrun.mp4 -vcodec libx264 -acodec aac -f flv rtmp://localhost/show/stream
```

我们来讲解下以上的参数含义:

- -re   这是一个输入参数，表示FFmpeg读取与输入视频的帧数相同的每秒帧数。它最适用于直播或摄像机输入。
- -i    输入视频，文件地址
- -vcodec 指定视频的编码格式，目前是x264编码，也就是H.264编码了，或者MPEG-4 part 10，或者也叫AVC编码
- -acodec 指定音频编码格式，大部分是aac编码
- -f  是一个流的输出参数，指定视频的输出容器格式，这里是flv，因为是RTMP协议，所以使用flv(Flash Video)格式
- 最后的参数就是RTMP server的推流地址了，也就是RTMP地址，这个URL一般来说是流媒体服务器，也就是RTMP server端配置的

##### 第二步，流媒体的服务

RTMP server的角色就是接收client的流数据，扩展提高性能，并服务于Internet上的广大观众（终端用户）。

Nginx就可以作为RTMP server，因为它有RTMP模块。

这里有一些 RTMP server的选项 [List of RTMP software - Wikipedia](https://en.wikipedia.org/wiki/List_of_RTMP_software#RTMP_server_software)


- [ossrs/srs: SRS is a simple, high efficiency and realtime video server, supports RTMP, WebRTC, HLS, HTTP-FLV and SRT. (github.com)](https://github.com/ossrs/srs)

- [ZLMediaKit/ZLMediaKit: WebRTC/RTSP/RTMP/HTTP/HLS/HTTP-FLV/WebSocket-FLV/HTTP-TS/HTTP-fMP4/WebSocket-TS/WebSocket-fMP4/GB28181/SRT server and client framework based on C++11 (github.com)](https://github.com/ZLMediaKit/ZLMediaKit)

##### 第三步，消费

最后一步就是终端用户消费观看RTMP视频流，因为server已经配置了RTMP协议的服务。我们就可以使用刚才的推流地址(rtmp://localhost/show/stream)了来观看，可以使用VLC这个播放器来观看。当然，现在当今网络上很多都是互联网了，所以可能会把RTMP转HLS这样的来播放







