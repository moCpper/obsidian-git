**视频编码概述**
- 像素格式转换 =》 编码 =》封装 
- 为什么要编码?
	 - 像素格式数据过大
- 原始帧AVFrame =》 压缩帧AVPacket
- 帧内压缩、帧间压缩
- 有损压缩、无损压缩

>**帧内压缩：**
帧内（Intraframe）压缩也称为空间压缩（Spatial compression）。当压缩一帧图像时，仅考虑本帧的数据而不考虑相邻帧之间的冗余信息，这实际上与静态图像压缩类似。
**帧间压缩**
帧间压缩(Interframe compression)也称为时间压缩(Temporal_compression)，是基于许多视频或动画的连续前后两帧具有很大的相关性(即连续的视频其相邻帧之间具有冗余信息)的特点来实现的；通过比较时间轴上不同帧之间的数据实施压缩，进一步提高压缩比。

 #  AVCodec
- AVCodec是FFmpeg库中用于表示编解码器的抽象，用于编码或解码视频或音频流。
- 它包含了编解码器所需的所有信息，如编解码器ID、工作模式、参数等。
- AVCodec用于处理数据的编码和解码过程，例如将原始视频帧转换为压缩的比特流，或反之

>在视频处理流程中，通常首先创建一个AVFrame来存储原始视频帧，然后使用AVCodec对其进行编码，生成压缩后的数据流。解码过程则相反，从数据流中提取数据，使用AVCodec解码到AVFrame，然后再进行显示或其他处理。

AVcodec源码解析

```cpp
typedef struct AVCodec {
    /**
     * Name of the codec implementation.
     * The name is globally unique among encoders and among decoders (but an
     * encoder and a decoder can share the same name).
     * This is the primary way to find a codec from the user perspective.
     */
    const char *name;
    /**
     * Descriptive name for the codec, meant to be more human readable than name.
     * You should use the NULL_IF_CONFIG_SMALL() macro to define it.
     */
    const char *long_name;
    enum AVMediaType type;
    enum AVCodecID id;
    /**
     * Codec capabilities.
     * see AV_CODEC_CAP_*
     */
    int capabilities;
    const AVRational *supported_framerates; ///< array of supported framerates, or NULL if any, array is terminated by {0,0}
    const enum AVPixelFormat *pix_fmts;     ///< array of supported pixel formats, or NULL if unknown, array is terminated by -1
    const int *supported_samplerates;       ///< array of supported audio samplerates, or NULL if unknown, array is terminated by 0
    const enum AVSampleFormat *sample_fmts; ///< array of supported sample formats, or NULL if unknown, array is terminated by -1
    const uint64_t *channel_layouts;         ///< array of support channel layouts, or NULL if unknown. array is terminated by 0
    uint8_t max_lowres;                     ///< maximum value for lowres supported by the decoder
    const AVClass *priv_class;              ///< AVClass for the private context
    const AVProfile *profiles;              ///< array of recognized profiles, or NULL if unknown, array is terminated by {FF_PROFILE_UNKNOWN}
 
    /**
     * Group name of the codec implementation.
     * This is a short symbolic name of the wrapper backing this codec. A
     * wrapper uses some kind of external implementation for the codec, such
     * as an external library, or a codec implementation provided by the OS or
     * the hardware.
     * If this field is NULL, this is a builtin, libavcodec native codec.
     * If non-NULL, this will be the suffix in AVCodec.name in most cases
     * (usually AVCodec.name will be of the form "<codec_name>_<wrapper_name>").
     */
    const char *wrapper_name;
 
    /*****************************************************************
     * No fields below this line are part of the public API. They
     * may not be used outside of libavcodec and can be changed and
     * removed at will.
     * New public fields should be added right above.
     *****************************************************************
     */
    int priv_data_size;
    struct AVCodec *next;
    /**
     * @name Frame-level threading support functions
     * @{
     */
    /**
     * If defined, called on thread contexts when they are created.
     * If the codec allocates writable tables in init(), re-allocate them here.
     * priv_data will be set to a copy of the original.
     */
    int (*init_thread_copy)(AVCodecContext *);
    /**
     * Copy necessary context variables from a previous thread context to the current one.
     * If not defined, the next thread will start automatically; otherwise, the codec
     * must call ff_thread_finish_setup().
     *
     * dst and src will (rarely) point to the same context, in which case memcpy should be skipped.
     */
    int (*update_thread_context)(AVCodecContext *dst, const AVCodecContext *src);
    /** @} */
 
    /**
     * Private codec-specific defaults.
     */
    const AVCodecDefault *defaults;
 
    /**
     * Initialize codec static data, called from avcodec_register().
     *
     * This is not intended for time consuming operations as it is
     * run for every codec regardless of that codec being used.
     */
    void (*init_static_data)(struct AVCodec *codec);
 
    int (*init)(AVCodecContext *);
    int (*encode_sub)(AVCodecContext *, uint8_t *buf, int buf_size,
                      const struct AVSubtitle *sub);
    /**
     * Encode data to an AVPacket.
     *
     * @param      avctx          codec context
     * @param      avpkt          output AVPacket (may contain a user-provided buffer)
     * @param[in]  frame          AVFrame containing the raw data to be encoded
     * @param[out] got_packet_ptr encoder sets to 0 or 1 to indicate that a
     *                            non-empty packet was returned in avpkt.
     * @return 0 on success, negative error code on failure
     */
    int (*encode2)(AVCodecContext *avctx, AVPacket *avpkt, const AVFrame *frame,
                   int *got_packet_ptr);
    int (*decode)(AVCodecContext *, void *outdata, int *outdata_size, AVPacket *avpkt);
    int (*close)(AVCodecContext *);
    /**
     * Encode API with decoupled packet/frame dataflow. The API is the
     * same as the avcodec_ prefixed APIs (avcodec_send_frame() etc.), except
     * that:
     * - never called if the codec is closed or the wrong type,
     * - if AV_CODEC_CAP_DELAY is not set, drain frames are never sent,
     * - only one drain frame is ever passed down,
     */
    int (*send_frame)(AVCodecContext *avctx, const AVFrame *frame);
    int (*receive_packet)(AVCodecContext *avctx, AVPacket *avpkt);
 
    /**
     * Decode API with decoupled packet/frame dataflow. This function is called
     * to get one output frame. It should call ff_decode_get_packet() to obtain
     * input data.
     */
    int (*receive_frame)(AVCodecContext *avctx, AVFrame *frame);
    /**
     * Flush buffers.
     * Will be called when seeking
     */
    void (*flush)(AVCodecContext *);
    /**
     * Internal codec capabilities.
     * See FF_CODEC_CAP_* in internal.h
     */
    int caps_internal;
 
    /**
     * Decoding only, a comma-separated list of bitstream filters to apply to
     * packets before decoding.
     */
    const char *bsfs;
 
    /**
     * Array of pointers to hardware configurations supported by the codec,
     * or NULL if no hardware supported.  The array is terminated by a NULL
     * pointer.
     *
     * The user can only access this field via avcodec_get_hw_config().
     */
    const struct AVCodecHWConfigInternal **hw_configs;
} AVCodec;
```


解析其中的各种成员变量，可得
```cpp
// 成员变量
const char *name；// 编码器名称，avcodec_find_encoder_by_name使用该变量
const char *long_name; //编码器全称
enum AVMediaType type; // 编码音视频类别
enum AVCodecID id;   // 编码器唯一id，avcodec_find_encoder使用该变量
const AVRational *supported_frames; // 支持的视频帧率
const enum AVPixelFormat *pix_fts;  // 视频像素格式
const int *supported_samplerates;  // 支持的音频采样率
const enum AVSampleFormat *sample_fmts; // 音频采样格式
const uint64_t *channel_layouts ; // 音频通道数
int priv_data_size;
const AVClass *priv_data;
struct AVCodec *next;  // 链表下一个编码器
// 成员函数
int (*send_frame)(AVCodecContext *avctx, const AVFrame *frame);    // avcodec_send_frame调用
int (*receive_packet)(AVCodecContext *avctx, AVPacket *avpkt);     // avcodec_recevie_packet调用
int (*receive_frame)(AVCodecContext *avctx, AVFrame *frame);       // avcodec_send_packet/avcodec_receive_frame调用
```


示例: FFmpeg编码器获取和上下文打开
```cpp
auto main() -> int {
	//1 找到编码器
	auto codec = avcodec_find_encoder(AV_CODEC_ID_H264);
	if (!codec) {
		std::cerr << "codec not find ! " << std::endl;
		return -1;
	}

	//2 编码上下文
	auto c = avcodec_alloc_context3(codec);
	if (!c) {
		std::cerr << "avcodec_alloc_context3 failed ! " << std::endl;
		return -1;
	}

	//3 设定上下文参数
	c->width = 400;							//视频宽高
	c->height = 300;		

	//帧时间戳的时间单位 pts/time_base = 播放时间(秒)
	c->time_base = { 1,25 };				//分数，1/25

	c->pix_fmt = AV_PIX_FMT_YUV420P;		//元数据像素格式，与编码算法相关
	c->thread_count = 16;					//编码线程数

	//4 打开编码上下文
	int re = avcodec_open2(c, codec, NULL);
	if(re != 0){
		char buf[1024]{ 0 };
		av_strerror(re, buf, sizeof(buf) - 1);
		std::cerr << "avcodec_open2 failed !" << buf << std::endl;
		return -1;
	}

	std::cout << "avcodec_open2 success!" << std::endl;

	//释放编码器上下文
	avcodec_free_context(&c);
	return 0;
}
```

**` avcodec_find_encoder `**
根据提供的编码器 ID（这里是 `AV_CODEC_ID_H264`），查找对应的编码器。如果找到了，返回一个指向 `AVCodec` 的指针；如果没有找到，返回 `NULL`。

**` avcodec_alloc_context3 `**
为编码器分配一个编码上下文 `AVCodecContext`，并初始化它。需要传入一个指向 `AVCodec` 的指针。如果成功，返回 0；如果失败，返回错误码。

** `avcodec_open2 `**
打开编码器，准备进行编码。传入编码上下文 `AVCodecContext` 指针、编码器 `AVCodec` 指针和一个指向 `AVDictionary` 的指针（这里传 `NULL`，表示没有额外的选项）。如果成功，返回 0；如果失败，返回错误码。

**`avcodec_free_context`**:
释放之前通过 `avcodec_alloc_context3` 分配的编码上下文。传入指向 `AVCodecContext` 指针的指针。


根据输出的日志可得，所有函数执行成功，而且显示了编码器使用的 CPU 功能和编码配置的一些信息。
![[Pasted image 20240813152250.png]]
