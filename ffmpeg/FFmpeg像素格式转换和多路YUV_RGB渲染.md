FFmpeg的libswscale 是一个主要用于处理图片像素数据的类库。可以完成图片像素格式的转换`(如rgb与yuv之间转换)`图片的拉伸，图像的滤波等工作。

## sws_getCachedContext
```cpp
SwsContext *sws_getCachedContext(struct SwsContext *context,
                             int srcW, int srcH, enum PixelFormat srcFormat,
                             int dstW, int dstH, enum PixelFormat dstFormat,
                             int flags, 
                             SwsFilter *srcFilter, 
                             SwsFilter *dstFilter,
                             const double *param);

```

1. 第一参数可以传NULL，默认会开辟一块新的空间。
2. srcW,srcH, srcFormat， 原始数据的宽高和原始像素格式(YUV420)，
3. dstW,dstH,dstFormat;   目标宽，目标高，目标的像素格式
4. flag 提供了一系列的算法，快速线性，差值，矩阵，不同的算法性能也不同，快速线性算法性能相对较高。只针对尺寸的变换。对像素格式转换无此问题

根据源码注释可知
```cpp
/**
 * Check if context can be reused, otherwise reallocate a new one.
 *
 * If context is NULL, just calls sws_getContext() to get a new
 * context. Otherwise, checks if the parameters are the ones already
 * saved in context. If that is the case, returns the current
 * context. Otherwise, frees context and gets a new context with
 * the new parameters.
 *
 * Be warned that srcFilter and dstFilter are not checked, they
 * are assumed to remain the same.
 */

```
> 如果*context*为 *null* 则调用*sws_getContext()*得到新的*context*，否则检测*context*是否已经存储，如果有则直接*return*，否则*free context* 并获取新的context。

## sws_scale() 
FFmpeg中的 sws_scale() 函数主要是用来做视频像素格式和分辨率的转换，其优势在于：可以在同一个函数里实现：1.图像色彩空间转换， 2:分辨率缩放，3:前后图像滤波处理。


**以下是一个将`YUV420P`转为`RGBA`的一个demo：**
```cpp
auto main() -> int {
	std::ifstream yuv_file;
	yuv_file.open(FILE_YUV, std::ios::binary);
	if (!yuv_file) {
		std::cout << "yuv_file open error ! " << std::endl;
		return -1;
	}
	std::ofstream rgba_ofs;
	rgba_ofs.open(RGBA_FILE, std::ios::binary);
	if (!rgba_ofs) {
		std::cerr << "RGBA_FILE open error ! " << std::endl;
		return -1;
	}
	int yuv_w = 400;
	int yuv_h = 300;
	int rgb_w = 800;
	int rgb_h = 600;

	//YUV420P 平面存储 yyyy uu vv
	unsigned char* yuv[3] = { 0 };
	int yuv_linesize[3]{ yuv_w,yuv_w/2,yuv_w/2 };
	yuv[0] = new unsigned char[yuv_w * yuv_h];			//Y
	yuv[1] = new unsigned char[yuv_w * yuv_h / 4];		//U
	yuv[2] = new unsigned char[yuv_w * yuv_h / 4];		//V
	
	//RGBA交叉存储 rgba rgba
	unsigned char* rgba = new unsigned char[rgb_w * rgb_h * 4];
	int rgba_linesize = rgb_w * 4;

	SwsContext* yuv2rgb = nullptr;
	for (int i = 0;i < 10;i++) {                 //绘制10帧
		//读取YUV帧
		yuv_file.read((char*)yuv[0],yuv_w*yuv_h);
		yuv_file.read((char*)yuv[1], yuv_w * yuv_h / 4);
		yuv_file.read((char*)yuv[2], yuv_w * yuv_h / 4);
		if (yuv_file.gcount() == 0) { break; }

		//yuv转rgba
		//上下文创建和获取
		yuv2rgb = sws_getCachedContext(
		yuv2rgb,						//转换上下文,nullptr新创建,非nullptr判断与现有参数
											//是否一致，一致则直接返回，否则先清理当前再创建
		yuv_w,yuv_h,					//输入宽高
		AV_PIX_FMT_YUV420P,				//输入像素格式
		rgb_w,rgb_h,					//输出的宽高
		AV_PIX_FMT_RGBA,				//输出的像素格式
		WS_BILINEAR,					//选择支持变化的算法，双线性插值
		0,0,0							//过滤器参数
		);
		if (!yuv2rgb) {
			std::cerr << "sws_getCachedContext failed !" << std::endl;
			return -1;
		}

		unsigned char* data[1];
		data[0] = rgba;
		int lines[1]{ rgba_linesize };
		int re = sws_scale(yuv2rgb,
			yuv,							//输入数据
			yuv_linesize,					//输入数据行字节数
			0,                              //Y轴高度
			yuv_h,							//输入高度
			data,							//输出数据
			lines
			);								//return输出的高度
		std::cout << re << std::endl;
		rgba_ofs.write((char*)rgba,rgb_w*rgb_h*4);
	}
	rgba_ofs.close();
	yuv_file.close();
	delete yuv[0];
	delete yuv[1];
	delete yuv[2];
	return 0;
}
```
将RGBA转YUV420P:
```cpp
std::ifstream inf;
SwsContext* rgb2yuv = nullptr;
inf.open(RGBA_FILE, std::ios::binary);
if (!inf) {
	std::cerr << "RGBA_file open error ! " << std::endl;
	return -1;
}
for (;;) {
	inf.read((char*)rgba, rgb_w * rgb_h * 4);
	if (inf.gcount() == 0) { break; }
	rgb2yuv = sws_getCachedContext(rgb2yuv, rgb_w, rgb_h, AV_PIX_FMT_RGBA,
		yuv_w, yuv_h, AV_PIX_FMT_YUV420P, SWS_BILINEAR, 0, 0, 0);
	if (!rgb2yuv) {
		std::cout << "sws_getCachedContext failed !" << std::endl;
		return -1;
	}
	unsigned char* data[1];
	data[0] = rgba;
	int lines[1]{ rgba_linesize };
	int re = sws_scale(rgb2yuv, data, lines, 0, rgb_h, yuv, yuv_linesize);
	std::cout << re << " ";
}
```

DEMO： 多路YUV_RGB渲染
github:[moCpper/-YUV_RGB-: 测试使用AVFrame实现RGB YUV二路渲染 (github.com)](https://github.com/moCpper/-YUV_RGB-)
