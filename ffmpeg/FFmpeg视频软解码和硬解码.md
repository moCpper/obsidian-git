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