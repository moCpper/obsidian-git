**视频编码概述**
- 像素格式转换 =》 编码 =》封装 
- 为什么要编码?
	 - 像素格式数据过大
- 原始帧AVFrame =》 压缩帧AVPacket
- 帧内压缩、帧间压缩
- 有损压缩、无损压缩

>**帧内压缩**
帧内（Intraframe）压缩也称为空间压缩（Spatial compression）。当压缩一帧图像时，仅考虑本帧的数据而不考虑相邻帧之间的冗余信息，这实际上与静态图像压缩类似。
>**帧间压缩**
帧间压缩(Interframe compression)也称为时间压缩(Temporal_compression)，是基于许多视频或动画的连续前后两帧具有很大的相关性(即连续的视频其相邻帧之间具有冗余信息)的特点来实现的；通过比较时间轴上不同帧之间的数据实施压缩，进一步提高压缩比。

#  AVCodec和AVCodecContext
- AVCodec是FFmpeg库中用于表示编解码器的抽象，用于编码或解码视频或音频流。
- 它包含了编解码器所需的所有信息，如编解码器ID、工作模式、参数等。
- AVCodec用于处理数据的编码和解码过程，例如将原始视频帧转换为压缩的比特流，或反之

>在视频处理流程中，通常首先创建一个AVFrame来存储原始视频帧，然后使用AVCodec对其进行编码，生成压缩后的数据流。解码过程则相反，从数据流中提取数据，使用AVCodec解码到AVFrame，然后再进行显示或其他处理。

AVcodec与AVCodecContext源码解析

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
 
/*
....

*/
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
```cpp
typedef struct AVCodecContext {
    const AVClass *av_class;
    AVMediaType codec_type; //编解码器处理的媒体类型，如AVMEDIA_TYPE_VIDEO或AVMEDIA_TYPE_AUDIO
    const AVCodec *codec; // 指向当前编解码器的指针
    AVCodecID codec_id; // 编解码器的ID，例如CODEC_ID_H264
    int width, height; // 视频的情况下，宽和高
    AVRational time_base; // 时间基，用于计算时间戳
    int bit_rate; // 编码时的目标比特率
    /* 更多编解码相关的参数... */
    AVFrame *frame; // 可选的，用于存储原始数据的帧
    AVFrame *coded_frame; // 编码后的帧，解码时使用
    /* 更多功能性、控制和状态字段... */
} AVCodecContext;

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
- 在使用 FFmpeg 进行编解码时，首先需要通过 `avcodec_find_encoder` 或 `avcodec_find_decoder` 函数找到合适的 `AVCodec` 结构体。然后，使用 `avcodec_alloc_context3` 为特定任务分配 `AVCodecContext`，并将找到的 `AVCodec` 结构体指针赋值给 `AVCodecContext` 的 `codec` 字段。这样，`AVCodecContext` 就与特定的编解码器关联起来。`AVCodec` 提供了编解码器的描述和接口，而 `AVCodecContext` 提供了执行编解码任务所需的具体环境和参数。两者相互配合，共同完成编解码任务。

** `avcodec_open2 `**
打开编码器，准备进行编码。传入编码上下文 `AVCodecContext` 指针、编码器 `AVCodec` 指针和一个指向 `AVDictionary` 的指针（这里传 `NULL`，表示没有额外的选项）。如果成功，返回 0；如果失败，返回错误码。

**`avcodec_free_context`**:
释放之前通过 `avcodec_alloc_context3` 分配的编码上下文。传入指向 `AVCodecContext` 指针的指针。

根据输出的日志可得，所有函数执行成功，而且显示了编码器使用的 CPU 功能和编码配置的一些信息。
![[Pasted image 20240813152250.png]]


# avcodec_send_frame和avcodec_receive_packet

>编码器和解码器都维护了一个缓冲区，在刚开始输入数据时，需要多输入几帧，等缓冲区
>被填充满后，才会在receive端接收到编码或解码后的数据
>send端只把数据放入缓冲区，recive端才是解码并获得数据的函数，如果receive发现已有解码后的数据则直接获取，如果没有则开始解码

** avcodec_send_frame **
向编码器提供原始的、未压缩的音视频帧数据，以便进行压缩编码。

** avcodec_receive_frame **
向独立的编码线程中接收压缩帧。
- 参数：
	-  AVCodecContext指针，编码器上下文。用于表示编码器的配置和状态信息。
	-  AVPacket指针，数据包。用于保存接收到的编码数据。
函数返回一个整数，表示执行结果。如果返回0，代表成功；返回AVERROR(EAGAIN)，代表当前状态没有可输出数据；返回AVERROR_EOF，代表已经到达输入流结尾；返回INVAL，代表编码器没有打开或者打开的是解码器。

** av_packet_alloc **
用于分配一个 `AVPacket` 结构体的内存。`AVPacket` 用于存储编码或解码过程中的数据包。


测试用例: 创建每一帧帧数据并压缩
```cpp
//创建好AVFrame空间 未压缩数据
auto frame = av_frame_alloc();
frame->width = c->width;
frame->height = c->height;
frame->format = c->pix_fmt;
re = av_frame_get_buffer(frame, 0);
if (re != 0) {
	char buf[1024]{ 0 };
	av_strerror(re, buf, sizeof(buf) - 1);
	std::cerr << "av_frame_get_buffer failed !" << buf << std::endl;
	return -1;
}

auto pkt = av_packet_alloc();
//10秒视频 250帧
for (int i = 0; i < 250; i++) {
	//生成AVFrame数据,每帧数据不同
	//Y
	for (int y = 0; y < c->height; y++) {
		for (int x = 0; x < c->width; x++) {
			frame->data[0][y * frame->linesize[0] + x] = x + y + i * 3;
		}
	}
	//UV
	for (int y = 0; y < c->height/2; y++) {
		for (int x = 0; x < c->width/2; x++) {
			frame->data[1][y * frame->linesize[1] + x] = 128 + y + i * 2;
			frame->data[2][y * frame->linesize[2] + x] = 64 + x + i * 5;
		}
	}
	frame->pts = i; //显示时间

	//发送未压缩帧到线程中压缩
	re = avcodec_send_frame(c, frame);
	if (re != 0) {
		break;
	}

	while (re >= 0) {	//返回多帧
		//接收压缩帧，一般前几次调用return 空(缓冲，立刻返回，编码未完成)
		//编码是在独立的线程中
		//每次调用会重写分配pkt中的空间
		re = avcodec_receive_packet(c, pkt);
		if (re == AVERROR(EAGAIN) || re == AVERROR_EOF) { break; }
		if(re < 0){ 
			char buf[1024]{ 0 };
			av_strerror(re, buf, sizeof(buf) - 1);
			std::cerr << "avcodec_receive_packet failed !" << buf << std::endl;
			break;
		}
		std::cout << pkt->size << " " << std::flush;
		av_packet_unref(pkt);
	}
}
```
