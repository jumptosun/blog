---
layout: post
title:  "libavformat.h 分析整理"
date:   2019-07-19
categories: media
---

# 功能概述 

Libavformat 是一个用来处理各种媒体封装的库。最重要的两个目的:
解封装和封装。 再使用 lavf 之前需要调用 av_register_all()，如
需使用网络功能，还用调用 avformat_network_init().

AVInpuFormat, AVOutputFormat 分别描述输入输出格式。你可以通过
av_iformat_next() / av_oformat_next() 来迭代所有 format.

lavf 中最主要的结构为 AVFormatContext， 在 demux 和 mux 中都有
使用。AVFormatContext 包含了所有文件读写所需用到的信息。因为
AVFormatContext 中大多数成员变量，并不是共用ABI 的一部分，所以
它可以在栈上分配。也可以使用 avformat_alloc_context 来创建.

AVFormatContext 主要包含:
AVFormatContext.iformat 即输入源格式描述结构， AVFormatContext.output
即输出源格式描述结构。 input 可以通过 lavf 来自动检测或手动设定,
而 output 必须手动设定。

AVFormatContext.streams 为 AVStreams 的数组, 它们描述了所有的存储
于文件中的媒体流。

AVFormatContext.pb 为 I/O context. 对于输入，可以通过 lavf 或者
手动打开。 对于输出，只能手动打开, 除非设置了AVFMT_NOFILE 标志。

lavf_options, 可以通过 avoptions 机制来配置 lavf 的 muxer  和 demuxer.
可以通过 av_opt_next 和 av_opt_find 来操作 AVFormatContext 中的 
AVClass.

URL : file:// 用来标识本地文件

demuxers 阅读一个媒体文件并且把它分离为连续的小块数据。AVPacket 包含了
属于同一路媒体流的一个或多个数据 . 在 lavf API 中，
avformat_open_input() 用来打开一个文件, av_read_frame 读取数据包， 
avformat_close_input() 用来关闭.

用例:   
```
  const char    *url = "file:in.mp3";
  AVFormatContext *s = NULL;
  int ret = avformat_open_input(&s, url, NULL, NULL);
  if (ret < 0)
      abort();
```

以上代码分配了 AVFormatContext 并且读取了媒体头部信息, 并且导出。
可能头部信息并不足以分析出媒体流信息， 所以我们强烈建议继续调用 
avformat_find_stream_info() 函数， 多读取几帧数据，来获取全部信息。

一些情况下你可能想要在 avformat_open_input 之前定制化 read/write
行为(例如： 从内存读取). 那么你需要在 avformat_alloc_context 后，
传入自己实现的 AVIOContext.


在调用之前可以指定部分输入媒体的参数，
```
  AVDictionary *options = NULL;
  av_dict_set(&options, "video_size", "640x480", 0);
  av_dict_set(&options, "pixel_format", "rgb24", 0);
 
  if (avformat_open_input(&s, url, NULL, &options) < 0)
      abort();
  av_dict_free(&options);
```
如果参数不匹配，会在 options 中继续存在。

使用 av_read_frame 来读取每一帧数据， AVPacket.stream_index 来表明
它从属于哪一路流。avpacket 可以传送给 av_send_packet/ avcodec_decode_subtitle2 来解码。

AVPacket.pts, AVPacket.dts 和 AVPacket.duration 时间信息，如果直到的话
会被设置. 如果媒体流中没有时间信息，也可以不设置。 时间戳的单位在
AVStream.time_base 中被设置。

如果返回的 AVPacket.buf 不为空，那么 packet 是动态分配的，用户可以
无限期的持有。 如果 buf 为空, 那么 AVPacket 是静态分配，数据仅在下
次 av_read_frame 之前有效。 如果用户需要内存长效有效，那么调用 
av_dup_packet 将会使用 av_malloc 来复制一个 packet.
上述两种情况下，都需要调用 av_packet_unref 来释放 packet.


封装主要使用：
avformat_write_header() for writing the
file header, av_write_frame() / av_interleaved_write_frame() for writing the
packets and av_write_trailer() for finalizing the file.

1. 需要选择合适的 AVFormatContext.oformat, 
2. 设置 AVFormatContext.pb , 使用 avio_open2
    AVCodecParameters.codec_type, AVCodecParameters.codec_id, width / heigh, the pixel or sample format。 time_base
3. 使用 avformat_new_stream() 来分配媒体流, 并且设置 AVStream.codecpar
4. 最好只设置相关的 codepar 的参数，少使用  avcodec_parameters_copy()


# 2. AVInputFormat

```
void av_register_input_format(AVInputFormat *format);
const AVInputFormat *av_demuxer_iterate(void **opaque);
const AVInputFormat *av_demuxer_iterate(void **opaque);
AVInputFormat *av_probe_input_format(AVProbeData *pd, int is_opened);
VInputFormat *av_probe_input_format2(AVProbeData *pd, int is_opened, int *score_max);
AVInputFormat *av_probe_input_format2(AVProbeData *pd, int is_opened, int *score_max);
nt av_probe_input_buffer2(AVIOContext *pb, AVInputFormat **fmt,
                                   const char *url, void *logctx,
                                   unsigned int offset, unsigned int max_probe_size);
int av_probe_input_buffer(AVIOContext *pb, AVInputFormat **fmt,
                                  const char *url, void *logctx,
                                  unsigned int offset, unsigned int max_probe_size);

int avformat_open_input(AVFormatContext **ps, const char *url, AVInputFormat *fmt, AVDictionary **options);
```

# 3. AVOutputFormat

```
int avformat_alloc_output_context2(AVFormatContext **ctx, AVOutputFormat *oformat,
                                   const char *format_name, const char *filename);

AVOutputFormat *av_muxer_iterate(void **opaque);
const AVOutputFormat *av_muxer_iterate(void **opaque);

AVOutputFormat *av_guess_format(const char *short_name,
                                const char *filename,
                                const char *mime_type);

enum AVCodecID av_guess_codec(AVOutputFormat *fmt, const char *short_name,
                              const char *filename, const char *mime_type,
                              enum AVMediaType type);
```
# 4. AVStream

```
struct AVCodecParserContext *av_stream_get_parser(const AVStream *s);
int64_t    av_stream_get_end_pts(const AVStream *st);
int av_stream_add_side_data(AVStream *st, enum AVPacketSideDataType type,
                            uint8_t *data, size_t size);

uint8_t *av_stream_new_side_data(AVStream *stream,
                                 enum AVPacketSideDataType type, int size);

uint8_t *av_stream_get_side_data(const AVStream *stream,
                                 enum AVPacketSideDataType type, int *size);

AVStream* avformat_new_stream(AVFormatContext *s, AVCodec* c)

```

# 5. AVFormatContext

```c
AVFormatContext *avformat_alloc_context(void);
void avformat_free_context(AVFormatContext *s);
```

# 6. 其他

```c
void av_hex_dump(FILE *f, const uint8_t *buf, int size);
void av_hex_dump_log(void *avcl, int level, const uint8_t *buf, int size);
void av_pkt_dump2(FILE *f, const AVPacket *pkt, int dump_payload, const AVStream *st);
void av_pkt_dump_log2(void *avcl, int level, const AVPacket *pkt, int dump_payload,
                               const AVStream *st);
```

# 7. 流程

封装:
1. avformat_alloc_context()    
2. 使用 avio_open2() 打开 I/O 处理    
3. avformat_alloc_output_context2()    
4. 手动设置 stream, codepar 等信息 avformat_new_stream()    
5. avformat_write_header() 写入文件头    
6. av_write_frame() / av_interleaved_write_frame()  写入文件数据    
7. av_write_trailer() 完成    


解封装:
1. avformat_alloc_context()    
2. 使用 avio_open2() 打开 I/O 处理    
3. 使用 avformat_open_input() 打开， 阅读媒体头, 传入 option    
4. 使用 avformat_find_stream_info(), 进一步查找媒体信息    
5. av_read_frame()    
    
3,4 步也可自手动设置传入的参数, AVInputFormat
