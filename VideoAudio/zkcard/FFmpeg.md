# FFmpeg

## 方便的小工具

FFmpeg命令行可视化，[YoungSx/ffmpeg-cmd-viewer (github.com)](https://github.com/YoungSx/ffmpeg-cmd-viewer)

![[Pasted image 20220706210734.png]]


#### FFmpeg之FFplay

[(4条消息) 音视频学习之ffplay基础命令整理_yun6853992的博客-CSDN博客_ffplay](https://blog.csdn.net/yun6853992/article/details/121870678)

可以播放本地视频和流媒体

#### FFmpeg推流与播放

FFmpeg推RTP流,用的封装格式是mpeg-ts，跟GB28181几乎一样。以下命令就是无限重复推送这个文件，注意下面的端口号10000，是Media Server上的RTP服务配置

```bash
ffmpeg -stream_loop -1 -re -i ./TextInMotion-VideoSamp  
le-1080p.mp4 -vcodec h264 -acodec aac -f rtp_mpegts rtp://127.0.0.1:10000
```

下面就是用ZLMediaKit作为MediaServer，所以播放RTMP媒体源的ffplay命令就是:

```bash
ffplay -showmode 0 rtmp://127.0.0.1:1935/rtp/1629F472
```

其中rtp就是app的名字，后面的十六进制字符串就是stream id，在ZLMediaKit中也是ssrc。这个要对应上，每一次推流，stream id都不一样。当然，你可以可以用RTSP的地址来收看这个推流。因为推流mpeg-ts或mpeg-ps的RTP流(GB28181)最终会在ZLMediaKit的Media Server输出到复用器，支持RTSP/RTMP/ts/fmp4等协议。

```bash
ffplay -showmode 0 rtsp://127.0.0.1:1554/rtp/1629F472
```

注意收看 RTMP和RTSP的端口号不一样。这个要看Media Server的RTMP和RTSP的服务配置。

[播放url规则 · ZLMediaKit/ZLMediaKit Wiki (github.com)](https://github.com/ZLMediaKit/ZLMediaKit/wiki/%E6%92%AD%E6%94%BEurl%E8%A7%84%E5%88%99)


