****
常识：
- **软解码**：**使用CPU进行解码**，不依赖于硬件加速。灵活性高，可以解码几乎所有FFmpeg支持的格式。
- **硬解码**：**使用非CPU进行解码**，如显卡GPU、专用的DSP、FPGA、ASIC芯片等，减轻CPU负担。但**可能不支持某些特定的解码格式**。硬解码可使用的格式如：
	 1. **CUDA**：如果使用 NVIDIA 的 CUDA 技术进行硬解码，那么解码过程通常发生在 GPU 的显存中。
    
	1. **VAAPI**：VAAPI（Video Acceleration API）是一种在 Linux 系统上使用的硬件加速技术，它允许 GPU 处理视频解码任务，解码过程发生在 GPU 的显存中。
    
	1. **DXVA2**：DXVA2（DirectX Video Acceleration 2.0）是 Microsoft 提供的一种视频加速技术，用于 Windows 系统。它也允许 GPU 进行视频解码，同样，解码过程发生在 GPU 的显存中。
    
	1. **QSV**：Intel Quick Sync Video 是 Intel 提供的一种视频加速技术，用于 Intel 的集成 GPU。硬解码发生在 GPU 的显存中。
    
	1. **Vulkan Video**：Vulkan 是一种跨平台的图形和计算 API，它也支持视频解码加速，解码过程同样发生在 GPU 的显存中。
    
	1. **Metal**：对于 Apple 的硬件，Metal API 可以用于 GPU 加速的视频解码。
    
硬解码的优势在于它可以显著提高解码效率，减少 CPU 的负担，特别是在处理高分辨率或高比特率的视频流时。
****
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
****
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

将**packet**发送的**解码线程**并获取：
```cpp
if (pkt->size) {					//若截取失败，为0
	//std::cout << pkt->size << " " << std::flush;
	//发送packet到解码线程
	ret = avcodec_send_packet(c, pkt);
	if (ret < 0) { break; }
	//获取多帧解码数据
	while (ret >= 0) {
		//每次会调用av_frame_unref
		ret = avcodec_receive_frame(c, frame);
		if (ret < 0) { break; }
		std::cout << frame->format << " " << std::flush;
	}
```
与编码相同，可能存在**缓冲区未读完**的情况，需要另做处理：
```cpp
	//取出缓存数据
	int ret = avcodec_send_packet(c, NULL);
	while (ret >= 0) {
		ret = avcodec_receive_frame(c, frame);
		if (ret < 0) { break; }
		std::cout << frame->format << "-" << std::flush;
	}
```
下面是一个完整的demo：将解码出的数据利用**SDL2**进行渲染
```cpp
std::string filename = "D:\\cppsoft\\ffmpeg\\code\\src\\first_ffmpeg\\test.h264";
std::ifstream ifs;
ifs.open(filename, std::ios::binary);
if (!ifs) { std::cerr << ".h264 open failed ! " << std::endl; return -1; }
unsigned char inbuf[4096]{ 0 };
AVCodecID codec_id = AV_CODEC_ID_H264;
auto view = XVideoView::Create();
//1 找到解码器
auto codec = avcodec_find_decoder(codec_id);

//2 创建上下文
auto c = avcodec_alloc_context3(codec);

//3 打开上下文
avcodec_open2(c, NULL, NULL);

//分割上下文
auto parser = av_parser_init(codec_id);
auto pkt = av_packet_alloc();
auto frame = av_frame_alloc();
int count = 0;										//计算帧率
auto beg = NowMs();
bool is_init_win = false;
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
			//std::cout << pkt->size << " " << std::flush;
			//发送packet到解码线程
			ret = avcodec_send_packet(c, pkt);
			if (ret < 0) { break; }
			//获取多帧解码数据
			while (ret >= 0) {
				//每次会调用av_frame_unref
				ret = avcodec_receive_frame(c, frame);
				if (ret < 0) { break; }
				std::cout << frame->format << " " << std::flush;
				/*
				* 第一帧初始化窗口
				*/
				if (!is_init_win) {
					is_init_win = true;
					view->Init(frame->width, frame->height, (XVideoView::Format)frame->format);
				}
				view->DrawFrame(frame);

				count++;
				auto cur = NowMs();
				if (cur - beg >= 100) {					// 1/10秒钟计算一次fps
					std::cout << "\nfps:"<<count*10 << std::endl;
					beg = cur;
					count = 0;
				}
			}
		}
	}
}

//取出缓存数据
int ret = avcodec_send_packet(c, NULL);
while (ret >= 0) {
	ret = avcodec_receive_frame(c, frame);
	if (ret < 0) { break; }
	std::cout << frame->format << "-" << std::flush;
}

av_parser_close(parser);
avcodec_free_context(&c);
av_frame_free(&frame);
av_packet_free(&pkt);
```

# 完成硬件GPU加速解码DXVA2
****
## 解码硬件加速 DXVA2

**`const char *av_hwdevice_get_type_name(enum AVHWDeviceType type);`**
此函数通常用于确定一个硬件设备（如 **GPU**）的类型，以便了解它支持哪些**硬件加速**特性。

```cpp
/*
* 打印所有支持的硬件加速方式
*/
for (int i = 0;; i++) {
	auto config = avcodec_get_hw_config(codec, i);
	if (!config) { break; }
	if (config->device_type) {
		std::cout << av_hwdevice_get_type_name(config->device_type) << std::endl;
	}
}
```
![[Pasted image 20240816182651.png]]
由此可得此设备支持的**硬件加速**的方式

开启硬件加速：
```cpp
//硬件加速格式
auto hw_type = AV_HWDEVICE_TYPE_DXVA2;
//初始化硬件加速上下文
AVBufferRef* hw_ctx = nullptr;
av_hwdevice_ctx_create(&hw_ctx,hw_type,NULL,NULL,0);

c->hw_device_ctx = av_buffer_ref(hw_ctx);
```
![[Pasted image 20240816185235.png]]
可见，**硬解码下CPU占用率几乎没有**。
>`硬解码下pix_fmt区别于YUV420P，是因为U和V存于同于个data[1]中，需要另作渲染。
>`硬解码过程发生在GPU显存中`

- 在开启1线程进行**软解码**时，fps以及CPU占用率
![[Pasted image 20240816185534.png]]
![[Pasted image 20240816185800.png]]

- 在开启1线程进行**硬解码**时，fps以及CPU占用率:
![[Pasted image 20240816185645.png]]
`pix_fmt为53说明AV_PIX_FMT_DXVA2_VLD`
![[Pasted image 20240816185904.png]]

**因为本机显卡为NVIDIA 4060laptop,在此设备下硬解码效率高于软解码**

>`在开启**硬件加速**时，调试可发现**frame**中的**data[0]到data[2]**都为NULL，这是因为data[3]中存放的是显存的地址。因此在利用SDL进行渲染时，需要将其copy到内存中。`

![[Pasted image 20240818085949.png]]

```cpp
				auto pframe = frame;		//为了同时支持硬解码和软解码
				if (c->hw_device_ctx) {		//硬解码
					//硬解码转换GPU =》CPU   显存=》内存
					//AV_PIX_FMT_NV12     1 plane for Y and 1 plane for the UV components
					av_hwframe_transfer_data(hw_frame,frame,0);
					pframe = hw_frame;
				}

				//AV_PIX_FMT_DXVA2_VLD
				std::cout << pframe->format << " " << std::flush;
```
`av_hwframe_transfer_data`用于将显存中的数据copy到内存中。
根据调试可得,转换后的pix_fmt区别于YUV420P，**hw_frame**的**data[1]**同时存储了**U和V**。
且hw_frame的format为**AV_PIX_FMT_NV12**，根据官方的注释`1 plane for Y and 1 plane for the UV components`可知他区别与传统的YUV420P。
