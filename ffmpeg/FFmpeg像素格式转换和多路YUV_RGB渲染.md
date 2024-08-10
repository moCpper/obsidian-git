FFmpeg的libswscale 是一个主要用于处理图片像素数据的类库。可以完成图片像素格式的转换`(如rgb与yuv之间转换)`图片的拉伸，图像的滤波等工作。
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

根据源码的官方注释可知
```cpp
**
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
struct SwsContext *sws_getCachedContext(struct SwsContext *context,
                                        int srcW, int srcH, enum AVPixelFormat srcFormat,
                                        int dstW, int dstH, enum AVPixelFormat dstFormat,
                                        int flags, SwsFilter *srcFilter,
                                        SwsFilter *dstFilter, const double *param);

```
> 如果*context*为 *null* 则调用*sws_getContext()*得到新的*context*，否则检测*context*是否已经存储，如果有则直接*return*，否则*free context* 并获取新的context。