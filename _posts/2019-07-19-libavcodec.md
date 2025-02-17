---
layout: post
title:  "libavcodec.h 分析整理"
date:   2019-07-19
categories: media
---

# 1. 功能概述 

avcodec_send_packet()/avcodec_receive_frame()/avcodec_send_frame()/
avcodec_receive_packet() 
提共了一组用于编解码的函数。

编解码流程非常的简洁  
1. 初始化 AVcodecContext  
2. 发送有效的输入  
    2.1. 对于解码, 调用 avcodec_send_packet，输入原始压缩数据   
    2.2. 对于编码, 调用 avcodec_send_frame, 未压缩的原始音视频数据  
    在上面两种情况中, 都建议使用带有引用计数的 avframe, avpacket  
3. 循环接受数据。 定期的调用 avcodec_receive_* 来处理输出数据  
    3.1. 对于解码，调用 avcodec_receive_frame。成功则返回原始音视频数据。  
    3.2. 对于编码，调用 avcodec_receive_packet。成功  
    重复调用函数，直到返回 AVERROR(EAGAIN) 或者错误。  
    返回 EAGAIN 说明需要新的输入。  
    每次输入 frame/packet， 通常会输出一个 frame/packet, 但多个输出或者  
    没有都是合理的。  

使用过程中建议遵守上面的流程，但不是必须的。   
编码中混合新旧 API 会导致不可预知的错误。   
所有的 codec 都支持新的 API, 使用旧 API 可能会报错。   
send/recieve 都返回 EAGAIN， 是不可能的。如果存在将会导致无限循环。   

# 2.  AVCodec

AVCodec 中主要包含一下回调函数。   
可以看出回调函数和 avcodec_send_*/avcodec_send_* 相对应。   
每一种编码器，要实现相应回调。   

```
void (*init_static_data)(struct AVCodec *codec);
int (*init)(AVCodecContext *);
int (*encode_sub)(AVCodecContext *, uint8_t *buf, int buf_size,
                  const struct AVSubtitle *sub);
int (*encode2)(AVCodecContext *avctx, AVPacket *avpkt, const AVFrame *frame,
               int *got_packet_ptr);
int (*decode)(AVCodecContext *, void *outdata, int *outdata_size, AVPacket *avpkt);
int (*close)(AVCodecContext *);
int (*send_frame)(AVCodecContext *avctx, const AVFrame *frame);
int (*receive_packet)(AVCodecContext *avctx, AVPacket *avpkt);
int (*receive_frame)(AVCodecContext *avctx, AVFrame *frame);
void (*flush)(AVCodecContext *);
```

操作AVCodec的函数主要包括以下：

```
const AVCodec *av_codec_iterate(void **opaque);
unsigned avcodec_version(void);
const char *avcodec_configuration(void);
const char *avcodec_license(void);
void avcodec_register(AVCodec *codec); // 4.0 以上就不用再调用了。
AVCodec *avcodec_find_decoder(enum AVCodecID id);
AVCodec *avcodec_find_decoder_by_name(const char *name);
AVCodec *avcodec_find_encoder(enum AVCodecID id);
AVCodec *avcodec_find_encoder_by_name(const char *name);
```
主要是操作 AVCodec 的链表。

# 3. AVCodecContext

主要操作函数
```
AVCodecContext *avcodec_alloc_context3(const AVCodec *codec);
void avcodec_free_context(AVCodecContext **avctx);
int avcodec_get_context_defaults3(AVCodecContext *s, const AVCodec *codec);
attribute_deprecated
int avcodec_copy_context(AVCodecContext *dest, const AVCodecContext *src);
int avcodec_close(AVCodecContext *avctx);

avcodec_open2 使用给定的 AVCodec 来初始化 AVCodecContext. 在此之前需要使用
avcodec_alloc_context3 来分配 AVCodecContext.
```

可以使用 avcodec_find_decoder_by_name(), avcodec_find_encoder_by_name(),
avcodec_find_decoder() 和 avcodec_find_encoder(), 来简化 codec 的寻找。

用例: 
```
 @warning This function is not thread safe!
 
 @note Always call this function before using decoding routines (such as
  @ref avcodec_receive_frame()).
 
  @code
  avcodec_register_all();
  av_dict_set(&opts, "b", "2.5M", 0);
  codec = avcodec_find_decoder(AV_CODEC_ID_H264);
  if (!codec)
      exit(1);
 
  context = avcodec_alloc_context3(codec);
 
  if (avcodec_open2(context, codec, opts) < 0)
      exit(1);
  @endcode
```
int avcodec_open2(AVCodecContext *avctx, const AVCodec *codec, AVDictionary **options);
int avcodec_close()(AVCodecContext *avctx)

# 4. AVPacket

保存压缩数据
操作实际数据函数包括

```
AVPacket *av_packet_alloc(void);
AVPacket *av_packet_clone(const AVPacket *src);
void av_packet_free(AVPacket **pkt);
void av_init_packet(AVPacket *pkt);

/**
 * 给 AVPacket 分配指定大小的内存，并且分配 refcount buf
 */
int av_new_packet(AVPacket *pkt, int size);

void av_shrink_packet(AVPacket *pkt, int size);
int av_grow_packet(AVPacket *pkt, int grow_by);
int av_packet_from_data(AVPacket *pkt, uint8_t *data, int size);
```

以下函数操作 side data
```
uint8_t* av_packet_new_side_data(AVPacket *pkt, enum AVPacketSideDataType type,
                                 int size);
int av_packet_add_side_data(AVPacket *pkt, enum AVPacketSideDataType type,
                            uint8_t *data, size_t size);
int av_packet_shrink_side_data(AVPacket *pkt, enum AVPacketSideDataType type,
                               int size);
uint8_t* av_packet_get_side_data(const AVPacket *pkt, enum AVPacketSideDataType type,
                                 int *size);
const char *av_packet_side_data_name(enum AVPacketSideDataType type);
uint8_t *av_packet_pack_dictionary(AVDictionary *dict, int *size);
int av_packet_unpack_dictionary(const uint8_t *data, int size, AVDictionary **dict);
void av_packet_free_side_data(AVPacket *pkt);
```

其他:
```
void av_packet_unref(AVPacket *pkt);
void av_packet_move_ref(AVPacket *dst, AVPacket *src);
int av_packet_copy_props(AVPacket *dst, const AVPacket *src);
int av_packet_make_refcounted(AVPacket *pkt);
int av_packet_make_writable(AVPacket *pkt);
void av_packet_rescale_ts(AVPacket *pkt, AVRational tb_src, AVRational tb_dst);
```

# 5. AVCodecParserContext, AVCodecParser
没有用到过，仅作记录

```
const AVCodecParser *av_parser_iterate(void **opaque);
void av_register_codec_parser(AVCodecParser *parser);
AVCodecParserContext *av_parser_init(int codec_id);
int av_parser_parse2(AVCodecParserContext *s,
                     AVCodecContext *avctx,
                     uint8_t **poutbuf, int *poutbuf_size,
                     const uint8_t *buf, int buf_size,
                     int64_t pts, int64_t dts,
                     int64_t pos);
int av_parser_change(AVCodecParserContext *s,
                     AVCodecContext *avctx,
                     uint8_t **poutbuf, int *poutbuf_size,
                     const uint8_t *buf, int buf_size, int keyframe);
void av_parser_close(AVCodecParserContext *s);
```

6. 针对像素格式转换的函数
```
unsigned int avcodec_pix_fmt_to_codec_tag(enum AVPixelFormat pix_fmt);
int avcodec_get_pix_fmt_loss(enum AVPixelFormat dst_pix_fmt, enum AVPixelFormat src_pix_fmt,
                             int has_alpha);
enum AVPixelFormat avcodec_find_best_pix_fmt_of_list(const enum AVPixelFormat *pix_fmt_list,
                                            enum AVPixelFormat src_pix_fmt,
                                            int has_alpha, int *loss_ptr);
enum AVPixelFormat avcodec_find_best_pix_fmt_of_2(enum AVPixelFormat dst_pix_fmt1, enum AVPixelFormat dst_pix_fmt2,
                                            enum AVPixelFormat src_pix_fmt, int has_alpha, int *loss_ptr);
enum AVPixelFormat avcodec_find_best_pix_fmt2(enum AVPixelFormat dst_pix_fmt1, enum AVPixelFormat dst_pix_fmt2,
                                            enum AVPixelFormat src_pix_fmt, int has_alpha, int *loss_ptr);
enum AVPixelFormat avcodec_default_get_format(struct AVCodecContext *s, const enum AVPixelFormat * fmt);
```

# 7. AVBSFContext
码流过滤器

```
/**
 * @deprecated the old bitstream filtering API (using AVBitStreamFilterContext)
 * is deprecated. Use av_bsf_get_by_name(), av_bsf_alloc(), and av_bsf_init()
 * from the new bitstream filtering API (using AVBSFContext).
 */
const AVBitStreamFilter *av_bsf_get_by_name(const char *name);
const AVBitStreamFilter *av_bsf_iterate(void **opaque);
int av_bsf_alloc(const AVBitStreamFilter *filter, AVBSFContext **ctx);
int av_bsf_init(AVBSFContext *ctx);
int av_bsf_send_packet(AVBSFContext *ctx, AVPacket *pkt);
int av_bsf_receive_packet(AVBSFContext *ctx, AVPacket *pkt);
void av_bsf_flush(AVBSFContext *ctx);
void av_bsf_free(AVBSFContext **ctx);
const AVClass *av_bsf_get_class(void);
typedef struct AVBSFList AVBSFList;
AVBSFList *av_bsf_list_alloc(void);
void av_bsf_list_free(AVBSFList **lst);
int av_bsf_list_append(AVBSFList *lst, AVBSFContext *bsf);
int av_bsf_list_append2(AVBSFList *lst, const char * bsf_name, AVDictionary **options);
int av_bsf_list_finalize(AVBSFList **lst, AVBSFContext **bsf);
int av_bsf_list_parse_str(const char *str, AVBSFContext **bsf);
int av_bsf_get_null_filter(AVBSFContext **bsf);
```

# 8. 其他

# 9. 流程

打开 context:
1. avcodec_find_decoder(AV_CODEC_ID_H264)/ avcodec_find_encoder
2. avcodec_alloc_context3(codec)
3. avcodec_open2(context, codec, opts)

编码: 
1. avcodec_send_frame
2. avcodec_recieve_packet

解码: 
1. avcodec_send_packet
2. avcodec_recieve_frame
