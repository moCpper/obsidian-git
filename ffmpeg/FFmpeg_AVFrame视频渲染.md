## AVFrame

AVFrame的源码实现：

```cpp
typedef struct AVFrame { 
	uint8_t* data[AV_NUM_DATA_POINTERS]; 
	int linesize[AV_NUM_DATA_POINTERS]; 
	uint8_t **extended_data; 
	int width, height;                      //视频帧宽和高(像素)
	int nb_samples;                         //音频帧中单个声道中包含的采样点数
	int format;                             //帧格式
	int key_frame; 
	enum AVPictureType pict_type; 
	AVRational sample_aspect_ratio; 
	int64_t pts; 
	...... 
} AVFrame;
```
其中linesize成员表示数组中的每个元素代表对应图像平面的每一行的字节数。

-  AVFrame 对象必须调用 **` av_frame_alloc() `**在堆上分配，注意此处指的是 AVFrame 对象本身，AVFrame 对象必须调用 **` av_frame_free() `**进行销毁。
-  AVFrame 通常只需分配一次，然后可以多次重用，每次重用前应调用 **` av_frame_unref() `** 将 frame 复位到原始的干净可用的状态。

** av_frame_get_buffer **

用于分配和初始化(默认值) `AVFrame` 结构体中的数据缓冲区。这个函数在处理视频和音频编解码时非常重要，因为它确保 `AVFrame` 结构体拥有适当的内存来存储媒体数据。
1. **`AVFrame* frame`**
    - 指向 `AVFrame` 结构体的指针。这个结构体用于存储解码后的视频帧或音频样本数据。
    - 在调用 `av_frame_get_buffer()` 之前，你通常需要初始化 `AVFrame` 结构体的其他字段（如 `width`, `height`, `format`, `channels` 等），以便函数能够正确分配内存。
2. **`int align`**
    - 内存对齐要求。这个参数指定了分配缓冲区时所需的对齐方式，以提高性能。
    - `align` 的值应该是 2 的幂。例如，`align` 为 16 表示分配的内存块将是 16 字节对齐的。通常使用 16 是一种常见的选择，但具体值可能依赖于编码器或解码器的要求。

**` 注意, av_frame_get_buffer()在执行成功后return 0 ; `**

** av_frame_free() **
 释放一个 frame。

** int av_frame_ref(AVFrame\* dst, const AVFrame\* src); **
将 src 中帧的各属性拷到 dst 中，并将共享的内存中的引用计数++ `(FFmpeg用纯C实现，不存在拷贝构造，拷贝赋值,析构等等，所以只能采用最原始的函数式的函数调用来模仿拷贝等一系列操作 )`

**  av_frame_clone() **
创建一个新的 frame，新的 frame 和 src 使用同一数据缓冲区，缓冲区管理使用引用计数机制。  
本函数相当于 av_frame_alloc()+av_frame_ref()

