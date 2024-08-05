### RGB颜色

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
>QImage提供图像的像素级的操作,其中QImage::bits()return图片首个像素的地址
>QPainter则负责在绘图设备上进行绘图操作，用以将QImage修改的图片在窗口上进行绘制。
>overridec'k
---
绘制结果如图：
![[Pasted image 20240805152130.png]]