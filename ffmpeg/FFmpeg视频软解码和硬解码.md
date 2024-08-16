常识：
- **软解码**：**使用CPU进行解码**，不依赖于硬件加速。灵活性高，可以解码几乎所有FFmpeg支持的格式。
- **硬解码**：**使用非CPU进行解码**，如显卡GPU、专用的DSP、FPGA、ASIC芯片等，减轻CPU负担。但**可能不支持某些特定的解码格式**。

# avcodec_send_packet
- 发送到解码线程
-  **avpkt**为NULL通过**`avcodec_receive_frame`**获取缓冲中的帧
-  **avpklt**数据传入后会**copy**一份，调用完可清理avpkt
-  **avpkt**中可以包含多帧数据，都会被解码。

# avcodec_receive_frame
- **` int avcodec_receive_frame(AVCodecContext* avctx,AVFrame* frame)
- frame:
	 - 每次调用函数都会调用**`av_frame_unref(frame)`**,导致data空间重新分配
	 - 如果要重复利用需要调用**`av_frame_make_writable(AVFrame* frame)`**
- return
	- 0表示返回成功，成功后缓冲中可能还有帧数据，可以再次屌用**`avcodec_receive_frame`**获取

# H264帧分割
**`av_parser_parse2`**
**解析**视频和音频流,要作用是分析输入的多媒体数据包，提取出帧信息.
`av_parser_parse2`尝试在输入数据中找到帧的边界。并将其分割到 `pkt->data` 中。`pkt->size` 表示解析出的帧的大小。
具体使用如：
```cpp
//1 分割H264 存入AVPacket
//ffmpeg -i v1080.mp4 -s 400x300 test.h264
std::string filename = "D:\\cppsoft\\ffmpeg\\code\\src\\first_ffmpeg\\test.h264";
std::ifstream ifs;
ifs.open(filename, std::ios::binary);
if (!ifs) { std::cerr << ".h264 open failed ! " << std::endl; return -1; }
unsigned char inbuf[4096]{ 0 };
AVCodecID codec_id = AV_CODEC_ID_H264;

//1 找到解码器
auto codec = avcodec_find_decoder(codec_id);

//2 创建上下文
auto c = avcodec_alloc_context3(codec);

//3 打开上下文
avcodec_open2(c, NULL, NULL);

//分割上下文
auto parser = av_parser_init(codec_id);
auto pkt = av_packet_alloc();
while (!ifs.eof()) {
	ifs.read((char*)inbuf,sizeof(inbuf));
	int data_size = ifs.gcount();			//读取的字节数
	if (data_size <= 0) { break; }
	auto data = inbuf;
	while (data_size > 0) {				//一次有多帧数据
		//通过0001 截断输出到AVPacket中 返回帧大小
		int ret = av_parser_parse2(parser, c,
			&pkt->data, &pkt->size,			//输出
			data, data_size,				//输入
			AV_NOPTS_VALUE, AV_NOPTS_VALUE, 0);
		data += ret;
		data_size -= ret;					//已处理
		if (pkt->size) {					//若截取失败，为0
			std::cout << pkt->size << " " << std::flush;
		}
	}
}
av_packet_free(&pkt);
```
![[Pasted image 20240816142306.png]]
根据输出日志可看到读取的每帧数据的大小。
