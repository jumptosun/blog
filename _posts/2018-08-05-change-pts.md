---
layout: post
title:  "使用 ffmpeg 改变音视频的播放速度"
date:   2018-07-08
categories: media
---

最近下了部英文电影，但是 1080p 的版本是俄语配音， 所以又找了个 720p 英文的版本，
使用 ffmpeg 合并他们 :

``` sh
ffmpeg -i video_1080p -i video_720p \   # 选择输入
           -map 0:v -c:v copy      \    # 选择 1080p 中的视频
           -map 1:a -c:a copy      \    # 选择 720p 中的音频
           video_fin
```

但是发现音视频不同步, 试用 ffprobe 发现, 720p 帧率为 25, 1080p 帧率 23.98，
所以 720p 视频播放时间更短，即合并后的电影音频比视频短。

所以要更改视频或者音频的 pts。 按道理改变视频部分的 pts 不用进行 re-encoding,
而音频的采样率固定， 改变音频 pts 必须 re-encoding, 进行丢帧或者补帧。
而实际中发现不管音视频改变 pts 总是会 re-encoding， 考虑到音频的复杂度小，所以:


``` sh
ffmpeg -i video_fin -filter:a "atempo=0.95904" -c:v copy video_align # 更改音频 pts
ffmpeg -i video_fin -filter:v "setpts=1.042535*PTS" -c:a copy  video_align # 更改视频 pts
```

ffmpeg wiki 地址 http://trac.ffmpeg.org/wiki/How%20to%20speed%20up%20/%20slow%20down%20a%20video

另外注意 23.98 是四舍五入后的帧率，实际是 23.976, 必须精确要不然音视频还是对不上， 解释来自知乎:

>首先从制式说起，PAL和NTSC的不同，NTSC是最早发明的版本，诞生于美国，在美，加，日本，南亚被使用，PAL则诞生于德国，是中国，欧洲使用的制式。传统电视一般都被无脑认为是30fps（Frame per second），意思即每秒30张图片连接而成的动态图像，绰号码率，等等。但实际上电视图像的传播码率不是30fps，而是根据PAL和NTSC的不同分别为25和30。电视传播中每张图片分为奇偶两场，因此如果计算场率的话，PAL和NTSC分别是50和60，但实际传播中，PAL制式下，电影一般使用24FPS来代替PAL中的25FPS，同时表示着片源的恒定码率，但实际上PAL制式的传播却是25FPS的，所以PAL制的DVD会把速度提高24/1(片源是24张图片一秒，但实际上一秒播放了25张），同时音调变高而NTSC后来改进了颜色制式信息，当这种改进被采用时，会让24FPS的片源延展到29.97，如何做到，通过3:2 pull down 算法，在5帧中映射实际的4帧，所以24FPS的片源映射到NTSC中是整30FPS，但是在录制片源的时候，录影带的速度会比Betacam（sony广泛被应用的一种数字格式/产品）慢0.1%，因此用30fps来拍摄，NTSC制式播放时就是29.97FPS，而NTSC电视下的直播码率都是直接设定为29.97,（因为不用换算速度差），23.976FPS也是根据这一原则换算求出的，大家可以算算，30对应29.97，那么24FPS的片源对应的呢？就是23.976FPS。这两种FPS对应不同制式下的不同用户，就是这样
>
>作者：费天翔   
>链接：https://www.zhihu.com/question/20633755/answer/15701422


## 参数 map 的解释

ffmpeg 关于 map 的解释:

> input file 'A.avi'
>       stream 0: video 640x360
>       stream 1: audio 2 channels
>
> input file 'B.mp4'
>       stream 0: video 1920x1080
>       stream 1: audio 2 channels
>       stream 2: subtitles (text)
>       stream 3: audio 5.1 channels
>       stream 4: subtitles (text)
>
> input file 'C.mkv'
>       stream 0: video 1280x720
>       stream 1: audio 2 channels
>       stream 2: subtitles (image)
>       
> ffmpeg -i A.avi -i B.mp4 out1.mkv out2.wav -map 1:a -c:a copy out3.mov

There are three output files specified, and for the first two, no -map options are set, so ffmpeg will select streams for these two files automatically.

out1.mkv is a Matroska container file and accepts video, audio and subtitle streams, so ffmpeg will try to select one of each type.
For video, it will select stream 0 from B.mp4, which has the highest resolution among all the input video streams.
For audio, it will select stream 3 from B.mp4, since it has the greatest number of channels.
For subtitles, it will select stream 2 from B.mp4, which is the first subtitle stream from among A.avi and B.mp4.

out2.wav accepts only audio streams, so only stream 3 from B.mp4 is selected.

For out3.mov, since a -map option is set, no automatic stream selection will occur. The -map 1:a option will select all audio streams from the second input B.mp4. No other streams will be included in this output file.
