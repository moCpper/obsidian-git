# 视频封装概述
****
- 像素格式转换 =》编码 =》 封装
- 为什么要封装？
	- 指定编码格式、帧率、时长等参数
	- 视频帧索引存储，方便视频进度跳转

# Mp4文件格式分析
****
**MP4** 文件由 box 组成，每个 box 分为 Header 和 Data。其中 Header 部分包含了 box 的类型和大小，Data 包含了子 box 或者数据，box 可以嵌套子 box。
具体可见[MP4文件格式入门](https://www.cnblogs.com/chyingp/p/mp4-file-format.html)
## fytp
File Type Box，一般在文件的开始位置，描述的文件的版本、兼容协议等。

## Movie Box(moov)
Movie Box，包含本文件中所有媒体数据的宏观描述信息以及每路媒体轨道的具体信息。有且仅有一个。一般位于 ftyp 之后，也有的视频放在文件末尾。注意，当改变 moov 位置时，内部一些值需要重新计算。

## mdat
Media Data Box，存放具体的媒体数据,一般有多个；


# 流程以及接口
****
**解封装流程**
`avformat_open_input`=》`avformat_find_stream_info`=》`av_read_frame`
`打开视频文件，解析文件头 =》无格式头文件格式解析 =》读取一帧数据`

***avformat_open_input***
用于打开一个媒体文件，并分配一个 `AVFormatContext` 结构体来存储媒体文件的格式信息。这个函数是处理多媒体文件的第一步，它允许你读取文件头，获取流信息和其他元数据。

>**AVFormatContext**负责管理整个媒体文件的打开和关闭，存储媒体文件的格式、编码信息、时间基准、时长、帧率和码率等信息。它还负责对媒体文件进行解封装操作，以便于提取出媒体文件中的音频、视频等媒体数据，为后续的解码、编码、读写等操作提供必要的信息和载体


***avformat_find_stream_info***
用于扫描媒体文件以获取流信息，包括**解码器参数、时间戳、帧率**等。这个函数在 `avformat_open_input` 成功打开媒体文件后调用，以填充 `AVFormatContext` 结构体中的 `streams` 数组。
>由于`avformat_find_stream_info` 函数会尝试读取足够的数据包以确定每个流的参数，例如音频的采样率、视频的分辨率等。这个过程可能需要一些时间，特别是对于大型文件或网络流。
```cpp
//打开媒体文件
const char* url = "D:\\cppsoft\\ffmpeg\\code\\src\\first_ffmpeg\\v1080.mp4";
//解封装输入上下文
AVFormatContext* ic = nullptr;
auto re = avformat_open_input(&ic,url,
	NULL,					//封装器格式 null 自动探测 根据后缀名或则文件头
	NULL
	);

CERR(re);

//获取媒体信息 无头部格式
re = avformat_find_stream_info(ic,NULL);
CERR(re);

//打印封装信息
av_dump_format(ic, 0, url
,0			//0表示上下文是输入  1输出
);

avformat_close_input(&ic);
```
![[Pasted image 20240819142334.png]]
根据日志可得.mp4的具体格式

***av_read_frame***
`int av_read_frame(AVFormatContext *s, AVPacket *pkt)`
用于从已经打开的多媒体文件中读取单个数据包.
- `s`: 一个指向已经打开的 `AVFormatContext` 结构体的指针。
- `pkt`: 一个指向 `AVPacket` 结构体的指针，用于存储读取的数据包信息。
**注意** : `av_read_frame`不同于`avcodec_receive_packet`或`avcodec_receive_frame`，**每次调用需要手动`av_packet_unref`释放AVPacket中的data成员**.
例如:
```cpp
AVPacket pkt;
for (;;) {
	re = av_read_frame(ic, &pkt);
	CERR(re);
	av_packet_unref(&pkt);
}
```

利用**AVFormatContext**成员以区分音频和视频包
```cpp
AVStream* as = nullptr; //音频流
AVStream* vs = nullptr; //视频流
for (int i = 0; i < ic->nb_streams; i++) {
	//音频
	if (ic->streams[i]->codecpar->codec_type == AVMEDIA_TYPE_AUDIO) {
		as = ic->streams[i];
		std::cout << " =======音频=======" << std::endl;
		std::cout << "sample_rate : " << as->codecpar->sample_rate << std::endl;
	}
	else if (ic->streams[i]->codecpar->codec_type == AVMEDIA_TYPE_VIDEO) {
		vs = ic->streams[i];
		std::cout << " =======视频=======" << std::endl;
		std::cout << "width  :" << vs->codecpar->width << std::endl;
		std::cout << "height  :" << vs->codecpar->height << std::endl;
	}
}

AVPacket pkt;
for (;;) {
	re = av_read_frame(ic, &pkt);
	CERR(re);
	if(vs && pkt.stream_index == vs->index){
		std::cout << "视频：";
		/*
		*         进行视频解码渲染操作
		*/
	}
	if (as && pkt.stream_index == as->index) {
		std::cout << "音频：";
	}
	std::cout << pkt.pts << ":" << pkt.dts << ":" << pkt.size << std::endl;
	av_packet_unref(&pkt);
}
```
![[Pasted image 20240819160738.png]]
>`codecpar`的类型为`AVCodecParameters`用于保存音视频流基本参数信息的结构体
>定义如下：
```cpp
typedef struct AVCodecParameters {
    enum AVMediaType codec_type;       
    enum AVCodecID   codec_id;
    uint32_t         codec_tag;
    uint8_t *extradata;
    int      extradata_size;
    int format;
    int64_t bit_rate;
    // ... 其他成员
} AVCodecParameters;
```
可用`avcodec_parameters_to_context`将解封装的视频编码参数，传递给解码对象。
>`avcodec_parameters_to_context` 函数用于将 `AVCodecParameters` 中的编码参数cp到 `AVCodecContext` 中。这通常在打开解码器或编码器之前使用，以确保 `AVCodecContext` 包含了流的编码参数。

# 封装视频
****
- 创建上下文
- 打开输出IO
- 写入文件头
- 写入帧数据（需要考虑写入次序）
- 写入尾部数据（pts索引）


**PTS计算**
****
概念：
- **PTS**：Presentation Time Stamp。PTS主要用于度量解码后的视频帧什么时候被显示出来。
- **DTS**：Decode Time Stamp。DTS主要是标识读入内存中的ｂｉｔ流在什么时候开始送入解码器中进行解码。
也就是pts反映帧什么时候开始显示，dts反映数据流什么时候开始解码。


**av_wite_frame 写入帧**
****
- `int av_write_frame(AVFormatContext* s,AVPacket* pkt)`
	-  直接写入
	-  pkt
		-  不改变pkt引用计数 NULL刷新缓冲
		-  pts使用s->streams中的time_base
		-  stream_index 要对应s->streams
	
- `int av_interleaved_write_frame(AVFormatContext* s,AVPacket* pkt)`
	- 需要在内部缓冲数据包，以确保输出文件中的数据包按照dts增加的顺序
	-  pkt:
		- 获取数据引用,**使用完引用计数减一，调用此函数后不可再访问pkt**
		- **非引用计数则复制 传递NULL写入interleaving queues缓冲**
	-  AVFormatContext.max_interleave_delta

**控制播放进度 av_seek_frame**
****
- `int av_seek_frame(AVFormatContext* s,int sream_index,int64_t timestamp,int flags);`
