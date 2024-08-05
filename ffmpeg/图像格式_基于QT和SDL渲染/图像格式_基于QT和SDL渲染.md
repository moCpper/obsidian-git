## RGB

- RGB的定义
  -   RGB888 是一种颜色表示方法，使用 24 位（8 位红色、8 位绿色和 8 位蓝色）来表示颜色。其他RGB变体如RGB565则依此类推，使用 16 位表示颜色（5 位红色，6 位绿色，5 位蓝色）
  -   ARGB888- 是一种扩展的颜色表示方法，使用 32 位（8 位 alpha 通道、8 位红色、8 位绿色和 8 位蓝色）来表示颜色。

| 属性  | RGB888        | ARGB888            |
| --- | ------------- | ------------------ |
| 位数  | 24            | 32                 |
| 通道数 | 3通道(R G B)    | 4通道(A R G B)       |
| 透明度 | 不支持           | 支持                 |
| 用途  | 通常用于不需要透明度的图像 | 适用于需要透明度的图像（如图层叠加） |

```cpp
void TestRGB::paintEvent(QPaintEvent* ev){
    QImage im(w, h, QImage::Format_RGB888);         //RGB888表示每24bit组成一个像素
    auto d = im.bits();                             //QImage::bits() 返回第一个像素地址
    int r = 255;
    for (int j = 0; j < h; j++) {
        int b = j * w * 3;                          //获取每行首像素的偏移
        r--;
        for (int i = 0; i < w * 3; i += 3) {
            d[b + i] = r;
            d[b + i + 1] = 0;
            d[b + i + 2] = 0;
        }   
    }

    QPainter p;
    p.begin(this);
    p.drawImage(0, 0, im);                          //0，0坐标从屏幕左上角开始计算
    p.end();
}

```

---
QImage和QPainter为QT绘图类。

-QImage提供图像的像素级的操作,其中QImage::bits()return图片首个'像素'的地址。
-QPainter则负责在绘图设备上进行绘图操作，用以将QImage修改的图片在窗口上进行绘制。
>
*override窗口的paintEvent以达到在自定义控件或窗口上进行绘图操作,其在调用show()时会被自动调用。*

---
绘制结果如图：
![[Pasted image 20240805152130.png]]


##SDL
基于SDL的图像渲染
```cpp
#include <iostream>
extern "C"{
#include<SDL.h>
}
#pragma comment(lib,"SDL2.lib")
#undef main
int SDL_main(int argc,char* argv[]){
    int w = 800;
    int h = 600;
    //1初始化SDL video库
    if (SDL_Init(SDL_INIT_VIDEO)) {
        std::cout << SDL_GetError();
        return -1;
    }

    //2生成SDL窗口
    auto screen = SDL_CreateWindow("test sdl ffmpeg", 
        SDL_WINDOWPOS_CENTERED,                 //窗口位置
        SDL_WINDOWPOS_CENTERED,
        w,h,
        SDL_WINDOW_OPENGL|SDL_WINDOW_RESIZABLE);
    if (!screen) {
        std::cout << SDL_GetError() << std::endl;
        return -1;
    }

    //3生成渲染器
    auto render = SDL_CreateRenderer(screen, -1,SDL_RENDERER_ACCELERATED);
    if (!render) {
        std::cout << SDL_GetError() << std::endl;;
        return -1;
    }

    //4生成材质
    auto texture = SDL_CreateTexture(render, SDL_PIXELFORMAT_ARGB8888,
        SDL_TEXTUREACCESS_STREAMING, w, h);                                         //可加锁
    if (!texture) {
        std::cout << SDL_GetError() << std::endl;
        return -1;
    }
    unsigned char _r = 255;
    //存放图像的数据
    unsigned char* r = new unsigned char[w * h * 4];
    for (;;) {
        _r--;
        SDL_Event ev;
        SDL_WaitEventTimeout(&ev,10);
        if (ev.type == SDL_QUIT) {
            SDL_DestroyWindow(screen);
            break;
        }

        for (int j = 0; j < h; j++) {
            int b = j * w * 4;
            for (int i = 0; i < w * 4; i += 4) {
                r[b + i] = 0;                   //B
                r[b + i + 1] = 0;               //G
                r[b + i + 2] = _r;             //R
                r[b + i + 3] = 0;               //A
            }
        }

        //5内存数据写入材质
        SDL_UpdateTexture(texture, NULL, r, w * 4);

        //6清理屏幕
        SDL_RenderClear(render);
        SDL_Rect sdl_rect;
        sdl_rect.x = 0;
        sdl_rect.y = 0;
        sdl_rect.w = w;
        sdl_rect.h = h;

        //7复制材质到渲染器
        SDL_RenderCopy(render, texture,
            NULL,                               //原图位置和尺寸
            &sdl_rect);                         //目标位置和尺寸

        //8渲染
        SDL_RenderPresent(render);
    }
    delete[] r;
    ::getchar();
    return 0;  
}
```


**SDL_CreateWindow()**
-  用于创建一个SDL窗口，提供给你绘制图形和处理用户输入的区域。
```cpp
auto screen = SDL_CreateWindow("test sdl ffmpeg", 
        SDL_WINDOWPOS_CENTERED,                 //窗口位置
        SDL_WINDOWPOS_CENTERED,
        w,h,
        SDL_WINDOW_OPENGL|SDL_WINDOW_RESIZABLE);
```
其传入的参数分别为 ： 
   -  窗口标题
   -  窗口水平位置
   -  窗口垂直位置
      -  **`SDL_WINDOWPOS_CENTERED`**: 将窗口的位置设置为屏幕中心。这是一个常用的标志，表示窗口在屏幕上的水平和垂直位置都居中。
   -  窗口宽度和高度
   - 以及窗口标志
      - **`SDL_WINDOW_OPENGL`**: 指示窗口将使用 OpenGL 上下文。这通常用于需要进行 OpenGL 渲染的应用程序。
      - **`SDL_WINDOW_RESIZABLE`**: 允许用户调整窗口的大小。这使得窗口可以被拖动和改变尺寸，提供更大的灵活性。


 **` SDL_CreateRenderer()`**
- 其函数返回一个渲染器，这个渲染器可以用来在窗口上绘制图形，是渲染图像和图形的主要工具，在使用 SDL 进行图形绘制时是不可或缺的。
- 
 
 