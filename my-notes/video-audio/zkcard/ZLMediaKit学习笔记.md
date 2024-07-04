它既是一个独立的Media Server服务器，又是一个Server和Client的框架，即支持推流，也支持拉流。

### ZLMediaKit的大致原理

![[Pasted image 20220728161159.png]]

[ZLMediaKit高并发实现原理 · ZLMediaKit/ZLMediaKit Wiki (github.com)](https://github.com/zlmediakit/ZLMediaKit/wiki/ZLMediaKit%E9%AB%98%E5%B9%B6%E5%8F%91%E5%AE%9E%E7%8E%B0%E5%8E%9F%E7%90%86)



### Http Restful 接口

[MediaServer支持的HTTP API · ZLMediaKit/ZLMediaKit Wiki (github.com)](https://github.com/zlmediakit/ZLMediaKit/wiki/MediaServer%E6%94%AF%E6%8C%81%E7%9A%84HTTP-API)


如果不是127.0.0.1的地址API请求，那么就需要带上config.ini中的secret参数

![[ed74a6c160c0971f4bb370776611b15.png]]

以上请求是拉取当前Media Server的配置。

### 录制视频测试

- ### `/index/api/getMp4RecordFile`
- `/index/api/startRecord`
- `/index/api/stopRecord`


secret=035c73f7-bb6b-4889-a715-d9eb2d1925cc
vhost=\_\_defaultVhost\_\_
app=rtp
stream=stream id

```
http://172.22.28.107:8080/index/api/startRecord?type=1&secret=035c73f7-bb6b-4889-a715-d9eb2d1925cc&vhost=__defaultVhost__&app=live&stream=test


```

这里与FFmpeg是联系的：

![[FFmpeg学习笔记]]


#### 树莓派RTMP推流到ZLMediaKit上的测试性能

- ZLMediaKit是`Release编译`
- FFmpeg开启树莓派的`硬编码和软编码H264`推送到WSL2内的Media Server上，在windows上打开VLC播放均有延迟，RTMP通过本地局域网路由器 （`硬编码H264有5s延迟， 软编码H264有7秒延迟`）
- 用ffplay播放本地摄像头无延迟（实时性很好，0s延迟）

```bash
ffplay -showmode 0 /dev/video0
```


- FFmepg开启树莓派 `硬编码H264`，推送到在树莓派上的Media Server，用FFplay 访问 127.0.0.1播放RTMP流，也有`延迟 (5s)`
-  FFmepg开启树莓派 `软编码H264`，推送到在树莓派上的Media Server，用FFplay 访问 127.0.0.1播放RTMP流，也有`延迟 (7s)`


FFmpeg在树莓派4B上使用H264硬编码RTMP推流:

```bash
ffmpeg -re -i /dev/video0 -vcodec h264_omx -f flv rtmp://127.0.0.1:1935/live/test
```

使用FFplay在树莓派上播放RTMP流:

```bash
ffplay -showmode 0 rtmp://127.0.0.1:1935/live/test
```

以上情况，可以排除视频流的延迟不是网络延迟原因，是树莓派这边的原因，而且`能确认H264硬编码功能确实开启并且起到了提高性能的作用`了，大概三种情况:

- 树莓派这边的ZLMediaKit的性能问题 
- 还是FFmpeg编码和推流的性能问题（修改命令行的推流参数）
- 树莓派本身的设备硬件H264编码器的性能所限问题（这个无法修改，可能需要换其他硬件设备测试，比如英伟达）


1. 验证问题1：是否是ZLMediaKit的Media Server处理慢？

那么要设计一个RTMP流不经过ZLMedia Server的测试案例:

在树莓派上直接FFmpeg推流到VLC播放端:


```bash
ffmpeg -re  -i /dev/video0   -vcodec h264 -f mpegts udp://127.0.0.1:1234
```

通过UDP直接推H264流到某个端口，然后通过ffplay播放这个端口的流

```bash
ffplay -showmode 0 udp://127.0.0.1:1234
```


使用这个参数测试延迟还是7秒。

换成硬件编码:

```bash
ffmpeg -re  -i /dev/video0   -vcodec h264_omx -f mpegts udp://127.0.0.1:1234
```
但是ffplay播放不了。



设计一个经过ZLMedia server的RTMP:

```bash
ffmpeg -re -i /dev/video0 -vcodec h264 -preset ultrafast -threads 4 -f flv rtmp://127.0.0.1:1935/live/test
```

ffplay播放延迟6秒

```bash
ffmpeg -re -i /dev/video0 -vcodec h264 -preset ultrafast -f flv rtmp://127.0.0.1:1935/live/test
```

ffplay播放延迟7秒

```bash
ffmpeg -re -i /dev/video0 -vcodec h264  -f flv rtmp://127.0.0.1:1935/live/test
```

ffplay播放延迟8秒


```bash
ffmpeg -re -i /dev/video0 -vcodec h264_omx   -f flv rtmp://127.0.0.1:1935/live/test
```

ffplay播放延迟5秒


延迟相关问题:

- [怎么测试ZLMediaKit的延时？ · ZLMediaKit/ZLMediaKit Wiki (github.com)](https://github.com/ZLMediaKit/ZLMediaKit/wiki/%E6%80%8E%E4%B9%88%E6%B5%8B%E8%AF%95ZLMediaKit%E7%9A%84%E5%BB%B6%E6%97%B6%EF%BC%9F)
- [直播延时的本质 · ZLMediaKit/ZLMediaKit Wiki (github.com)](https://github.com/ZLMediaKit/ZLMediaKit/wiki/%E7%9B%B4%E6%92%AD%E5%BB%B6%E6%97%B6%E7%9A%84%E6%9C%AC%E8%B4%A8)
- [延时测试 · ZLMediaKit/ZLMediaKit Wiki (github.com)](https://github.com/ZLMediaKit/ZLMediaKit/wiki/%E5%BB%B6%E6%97%B6%E6%B5%8B%E8%AF%95)


不一定是ZLMediaKit的server问题（也可能是ZLMediakIT的原因），因为推送UDP的时候，没有打包RTMP，直接是mpegts的容器格式。播放端解码次数就减少了。

看了ZLMediaKit的文档，可能找到延迟的原因了，大部分肉眼可观测的延迟可能是播放器的原因，用ffplay要传递特殊的参数,才可以比较实时:

```bash
ffplay -showmode 0 -i rtmp://127.0.0.1:1935/live/test -fflags nobuffer
```

以上测试，几乎无延迟。在本地测试的，不走网络。

走了网络，用以上命令播放，几乎也是无延迟。又用VLC测试了一下，确定是播放器的问题了。

2. 验证问题2: 是否是ffmpeg推流的性能问题

先研究ffmpeg的推流参数，修改，观察现象。

[Encode/H.264 – FFmpeg](https://trac.ffmpeg.org/wiki/Encode/H.264)

如果不用硬件编码H264，用ffmpeg本身的libx264软编码，选择--preset 为ultrafast:

```bash
ffmpeg -re -i /dev/video0  -preset ultrafast  -vcodec libx264  -f mpegts udp://127.0.0.1:1234
```

那么通过ffplay播放，实时性很高，延迟基本在`1秒左右`。

结论，树莓派的硬件编码H264还干不过ffmpeg的软解码，这个软解码器的性能以及实时性基本达到项目的要求。但是软解码器可能CPU发热厉害，设备寿命以及耐久度会减少。