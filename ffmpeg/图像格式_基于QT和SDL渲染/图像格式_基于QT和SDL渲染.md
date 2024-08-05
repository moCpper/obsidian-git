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
            d[b + i] = 255;
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

QImage和QPainter为QT绘图类，
