## RTP，RTCP和RTSP，RTMP的概念与关系

用简单一句话总结: **RTSP发起/终结流媒体，RTP传输流媒体数据，RTCP对RTP进行控制，同步。**

所以经常会看到XX音视频数据用RTP/RTCP传输推流，就把RTP和RTCP这两个概念放到一起。这样就容易混淆。

之所以以前对这几个有点分不清，是因为CTC标准里没有对RTCP进行要求，因此在标准RTSP的代码中没有看到相关的部分。而在私有RTSP的代码中，有关控制、同步等，是在RTP Header中做扩展定义实现的。

另外，RFC3550可以看作是RFC1889的升级文档，只看RFC3550即可。

#### RTP

RTP(Real-time Transport Protocol)是用于Internet上针对多媒体数据流的一种传输层协议，RTP经常配合RTCP协议做视频会议这些。它是建立在UDP之上的。

RTP本身没有提供按时发送机制或者其他QoS保证，它依赖于底层服务区实现这个过程，它不保证传送或者防止无序传输，也不确定底层网络的可靠性。

RTP仅仅是不可靠地传送实时属性的流媒体数据，RTCP监控服务质量并传送正在进行会话参与者的相关信息

#### RTCP

RTCP(Real-time Transport Control Protocol)是RTP协议的一个兄弟协议，RTCP为RTP提供信道外的控制，RTCP本身不传输数据，但是和RTP一起合作将多媒体数据打包和传输，RTCP定期在流媒体会话参与者之间传输控制数据，RTCP主要为RTP提供的QoS提供反馈。RTCP收集相关媒体连接的统计信息，比如传输字节数，传输分组数，丢失分组数，单向和双向网络延迟等等，这样来从统计上评估RTP的QoS质量。

从这里你可以看到，整个RTP协议其实由RTP和RTCP组成。

#### SRTP和SRTCP

安全实时传输协议（Secure Real-time Transport Protocol或SRTP）是在实时传输协议（Real-time Transport Protocol或RTP）基础上所定义的一个协议，旨在为单播和多播应用程序中的实时传输协议的数据提供加密、消息认证、完整性保证和重放保护。它是由David Oran（思科）和Rolf Blom（爱立信）开发的，并最早由IETF于2004年3月作为RFC 3711发布。

由于实时传输协议和可以被用来控制实时传输协议的会话的实时传输控制协议（RTP Control Protocol或RTCP）有着紧密的联系，安全实时传输协议同样也有一个伴生协议，它被称为安全实时传输控制协议（Secure RTCP或SRTCP）；安全实时传输控制协议为实时传输控制协议提供类似的与安全有关的特性，就像安全实时传输协议为实时传输协议提供的那些一样。

在使用实时传输协议或实时传输控制协议时，使不使用安全实时传输协议或安全实时传输控制协议是可选的；但即使使用了安全实时传输协议或安全实时传输控制协议，所有它们提供的特性（如加密和认证）也都是可选的，这些特性可以被独立地使用或禁用。唯一的例外是在使用安全实时传输控制协议时，必须要用到其消息认证特性。


#### RTSP

RTSP(Real-time Streaming Protocol) 是用来控制声音或者影像的多媒体串流协议，并允许多个串流需求控制，传输时所用的网络通讯协议并不在其定义范围之内，服务端可以自行选择使用TCP或UDP传送串流内容，请求主要有DESCRIBE, SETUP, PLAY, PAUSE, TEARDOWN, OPTIONS等，显然就是对会话起到控制作用。RTSP的会话过程中，SETUP可以确定RTP/RTCP使用的端口，PLAY/PAUSE/TEARDOWN可以开始或者暂停RTP的发送。


#### RTP和RTSP

在网络上，你会经常看到类似以下的RTSP和RTP url:

- rtsp://192.168.0.184/myvideo.mpg
- rtp://192.168.0.184:10000

第一个连接就是流化myvideo.mpg这个文件，RTSP可以流化几乎任何东西，音视频，甚至文本。RTSP会与RTP做协商，也就是可以简单理解，RTSP是建立在RTP之上的，在一个会话中，RTSP会控制RTP的传输，开始，暂停RTP流等等。

因此，如果你想让你的medie server在URL被请求时就开始流化某些媒体，你可以实现或者使用某种RTP-only的服务器。但是，如果你想有更多的控制权（想控制流媒体的开始暂停等），你必须使用RTSP，也就是你的media server必须支持RTSP协议，你的客户端也要支持RTSP的请求发送，因为它可以传输SDP和其他重要的解码数据。


所以有RTSP的地方，肯定底层就会有RTP。


#### RTSP和RTMP

这两个协议基本可以放在同等层次的概念中，所以它两之间有个比较和选择。

TMP和RTSP视频流协议使用户能够在任何网络浏览器和大多数移动设备上观看内容。

RTMP和RTSP都是流媒体协议，这意味着它们是管理流媒体如何从一个通信系统到另一个通信系统的规则集。

从一个通信系统传输到另一个系统的规则。如果你想发送的数据是一辆车，那么流媒体协议就是汽车从一个地方到另一个地方所走的路。


两个最常见的流媒体协议是RTMP和RTSP，这就是为什么你会经常看到 
RTSP与RTMP的比较。

虽然它们都完成了类似的目标，但当我们比较RTSP和RTMP时，有一些 
重要的区别。

###### RTMP

RTMP(Real-time Messaging Protocol)呢，当时是个标准化的流媒体协议，它能有效地流传低延迟的按需内容。这种数据可以是预先录制的(视频文件)，也可以是现场直播的(摄像头推流)，但RTMP目前最常被用于直播的音视频内容。

虽然大多数实时视频流媒体软件支持RTMP摄取，但大多数在线视频流利用 
HLS流媒体协议。HTTP Live Streaming(HLS)协议是由苹果公司开创的，与几乎所有的移动设备、游戏机兼容。

RTMP将音频和视频文件从RTMP编码器(推流端)传输到视频托管平台(Media Server)。而HLS将文件从托管平台（Media Server）传输到 各个用户的终端设备上。

RTMP的优势:
- 低延迟，低延迟就允许视频流这个连接保持比较稳定，视频观看用户体验这些都会好。即使底层的网络不可靠。一句话，就是观看体验会很好，所以适合直播
- 适应性强。一个可适应的消费端意味着你的观众不会被锁定在一个线性方向。适应性允许他们跳过和倒退部分流，或在直播开始后可以方便地从中加入观看。大多数观众现在希望他们观看的视频有这种能力。
- 灵活性好。RTMP允许你将各种媒体类型整合到一个统一的包中。无缝地融合音频、视频和文本。此外，你可以有多种不同的媒体渠道，如同时传输MP3和AAC音频流或传输MP4、FLV和F4V视频。它允许RTMP音频流(支持声音直播)。

RTMP的劣势:
- html5不支持。相当于与Web端天然不兼容，现在是html5 Web互联网的天下，所以现在可以看到推流端用RTMP推，但是消费流的这端有转换成HLS的转换器，Web就可以比较方便的消费HLS流了。
- 带宽问题。RTMP流可能特别容易受到低视频带宽问题的影响。这可能会导致你的流媒体经常出现令人沮丧的中断，破坏了你的体验观众的体验。
- http不兼容。你不能通过HTTP连接直接流式传输RTMP数据。为了在你的网站上收看RTMP流，你必须连接到一个特殊的服务器，如 Flash媒体服务器，并使用一个第三方内容交付网络（CDN）

从优势和劣势上面来看，你可以看到，RTMP最适合推流端，消费端反而不适合RTMP，而是HLS,或者Http Live Streaming。

###### RTSP

RTSP相比RTMP没有那么广为人知，这个协议被设计用来控制流媒体服务器的，与RTP交互。

当RTSP控制server到client的连接时，使用的是视频点播流；当它控制client到server的连接时，RTSP使用的是录音流。

RTSP通常用于互联网协议（IP）摄像机流，如那些来自CCTV或IP摄像机。

最开始我做的监控就是RTSP，所以蛮适用监控领域的。但是现在很多监控领域也用RTMP推流了。

RTSP的优势:
- 分段式的流。与其强迫你的观众在观看视频之前下载整个视频观赏，RTSP流允许他们在下载完成之前观看你的内容。
- 定制化。因为之前提到过，RTSP没有限定底层的网络传输协议，通过利用其他协议，如传输控制协议（TCP）和用户数据报协议（UDP）。用户数据报协议（UDP），你可以创建你自己的视频流应用。

RTSP的劣势:
- 不是那么流行。这样以来，那么开源的生态相比RTMP就弱势很多。解决方案也少很多。大多数视频播放器，流媒体不会优先支持RTSP。
- Http不兼容。 与RTMP一样，你不能直接通过HTTP来传输RTSP。正因为如此。由于RTSP的设计更多的是用于私人网络上的视频流，如企业内的安全系统，所以没有简单、直接的方法在览器中传输RTSP。



总的来说，我觉得当今趋势就是RTMP的生态还是要好于RTSP的，即使做监控，大部分也都是在RTMP上了。



#### 参考

- [RTP、RTCP、RTSP 概念-阿里云开发者社区 (aliyun.com)](https://developer.aliyun.com/article/48045)
- [(2 封私信 / 80 条消息) RTP/RTSP/RTCP有什么区别？ - 知乎 (zhihu.com)](https://www.zhihu.com/question/20278635)
- [(4条消息) rtp,rtsp,rtcp的区别_zhanghuaichao的博客-CSDN博客_rtp和rtcp区别](https://blog.csdn.net/zhanghuaichao/article/details/103211053)
- [What is the difference between RTP or RTSP in a streaming server? - Stack Overflow](https://stackoverflow.com/questions/4303439/what-is-the-difference-between-rtp-or-rtsp-in-a-streaming-server)
- 

