# FFmpeg

## 方便的小工具

FFmpeg命令行可视化，[YoungSx/ffmpeg-cmd-viewer (github.com)](https://github.com/YoungSx/ffmpeg-cmd-viewer)

![[Pasted image 20220706210734.png]]


#### FFmpeg之FFplay

[(4条消息) 音视频学习之ffplay基础命令整理_yun6853992的博客-CSDN博客_ffplay](https://blog.csdn.net/yun6853992/article/details/121870678)

可以播放本地视频和流媒体

#### FFmpeg推流与播放

FFmpeg推RTP流,用的封装格式是mpeg-ts(ffmpeg不支持mpeg-ps)，跟GB28181几乎一样。以下命令就是无限重复推送这个文件，注意下面的端口号10000，是Media Server上的RTP服务配置

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

[ZLMediaKit推流测试 · ZLMediaKit/ZLMediaKit Wiki (github.com)](https://github.com/ZLMediaKit/ZLMediaKit/wiki/ZLMediaKit%E6%8E%A8%E6%B5%81%E6%B5%8B%E8%AF%95)

下面是访问HLS协议的的浏览器URL:

```txt
http://172.22.22.94:8080/rtp/F6EC2C40/hls.m3u8?vhost=__defaultVhost__&secret=035c73f7-bb6b-4889-a715-d9eb2d1925cc
```

如果是在浏览器地址栏输入以上URL，那么不能通过m3u8文件播放视频，默认它会把文件下载到本地，本地用windows Media 11 Player播放器可以打开m3u8索引文件，但是找不到相关的.ts分片视频文件。


[LiveQing](http://cloud.liveqing.com:18000/#/liveplayer)

以上链接可以播放WSL2中的m3u8文件，完美支持HLS。

以下是自己写的一个调用liveQing的静态网页实现的HLS播放

```html
<!DOCTYPE html>

<html>

    <head>

        <script src="https://cdn.jsdelivr.net/npm/hls.js@1"></script>

    </head>

    <body>
        <iframe src="http://cloud.liveqing.com:18000/LivePlayer.html?videoUrl=http%3A%2F%2F172.22.22.94%3A8080%2Frtp%2FF6EC2C40%2Fhls.m3u8%3Fvhost%3D__defaultVhost__%26secret%3D035c73f7-bb6b-4889-a715-d9eb2d1925cc" width="640" height="360" allowfullscreen></iframe>

    </body>

    <script>

      

    </script>

</html>
```

RTMP推流

```bash
ffmpeg -stream_loop -1 -re -i ./TextInMotion-VideoSample-1080p.mp4 -vcodec h264 -acodec aac -f flv rtmp://127.0.0.1:1935/[app]/[stream id]
```

如果是用RTMP推流，用ffplay也可以播放RTMP，但是用HLS就可以播放RTMP的推流。用VLC也可以在宿主机上播放WSL2上media server的RTMP流.

```txt
rtmp://172.22.22.94:1935/live/test?vhost=__defaultVhost__&secret=035c73f7-bb6b-4889-a715-d9eb2d1925cc
```

```bash
ffplay -showmode 0 rtmp://127.0.0.1:1935/live/test
```

注意：如果在WSL2上，ffplay播放不出本地mp4文件或者播放不出rtmp流，URL没有问题的话，那么就重启WSL2。

