## RGB颜色

- RGB的定义
  -   RGB888 是一种颜色表示方法，使用 24 位（8 位红色、8 位绿色和 8 位蓝色）来表示颜色。其他RGB变体如RGB565则依此类推，使用 16 位表示颜色（5 位红色，6 位绿色，5 位蓝色）
  -   ARGB888- 是一种扩展的颜色表示方法，使用 32 位（8 位 alpha 通道、8 位红色、8 位绿色和 8 位蓝色）来表示颜色。

| 属性  | RGB888        | ARGB888            |
| --- | ------------- | ------------------ |
| 位数  | 24            | 32                 |
| 通道数 | 3通道(R G B)    | 4通道(A R G B)       |
| 透明度 | 不支持           | 支持                 |
| 用途  | 通常用于不需要透明度的图像 | 适用于需要透明度的图像（如图层叠加） |

```
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

-QImage提供图像的像素级的操作,其中QImage::bits()return图片首个像素的地址。
-QPainter则负责在绘图设备上进行绘图操作，用以将QImage修改的图片在窗口上进行绘制。
>
*override窗口的paintEvent以达到在自定义控件或窗口上进行绘图操作,其在调用show()时会被自动调用。*

---
绘制结果如图：
![[Pasted image 20240805152130.png]]


##

