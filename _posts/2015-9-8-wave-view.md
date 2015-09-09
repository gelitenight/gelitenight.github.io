---
layout: post
title: "Android实现波浪效果 - WaveView"
published: true
---

## 效果图
先上效果图

![WaveView截图](../images/2015-9-8-Wave-View/screenshot.gif)

## 实现
### WaveView的属性
- Wate Level(水位): 波浪静止时水面距离底部的高度
- Amplitude(振幅): 波浪垂直振动时偏离水面的最大距离
- Wave Length(波长): 一个完整的波浪的水平长度
- Wave Shift(偏移): 波浪相对于初始位置的水平偏移

### 实现思路
1. 创建一个尺寸恰等于WaveView的Bitmap。
2. 在Bitmap中使用默认的属性动态地绘制出默认的波形，默认波形由透明度和水平位移不同的两个波组成。
3. 把这个Bitmap设置成为Paint的Shader。设置Shader相当于画笔的形状，使用设置了Shader的Paint绘制图形时，实际上是在使用Bitmap填充绘制的区域。
4. 在OnDraw中，根据用户设置的属性，用Matrix对Shader进行拉伸和平移，再用它填充WaveView画出最终的效果。

### 关键代码
{% highlight java linenos=table %}
private void createShader() {
    mDefaultAngularFrequency = 2.0f * Math.PI / DEFAULT_WAVE_LENGTH_RATIO / getWidth();
    mDefaultAmplitude = getHeight() * DEFAULT_AMPLITUDE_RATIO;
    mDefaultWaterLevel = getHeight() * DEFAULT_WATER_LEVEL_RATIO;
}
{% endhighlight %}
``` java
private void createShader() {
    mDefaultAngularFrequency = 2.0f * Math.PI / DEFAULT_WAVE_LENGTH_RATIO / getWidth();
    mDefaultAmplitude = getHeight() * DEFAULT_AMPLITUDE_RATIO;
    mDefaultWaterLevel = getHeight() * DEFAULT_WATER_LEVEL_RATIO;

    Bitmap bitmap = Bitmap.createBitmap(getWidth(), getHeight(), Bitmap.Config.ARGB_8888);
    Canvas canvas = new Canvas(bitmap);

    // Draw default waves into the bitmap
    // y=Asin(ωx+φ)+h
    int waveX1 = 0;
    final int wave2Shift = getWidth() / 4;
    final int endX = getWidth();
    final int endY = getHeight();

    final Paint wavePaint1 = new Paint();
    wavePaint1.setColor(Color.WHITE);
    wavePaint1.setAlpha(40);
    wavePaint1.setAntiAlias(true);

    final Paint wavePaint2 = new Paint();
    wavePaint2.setColor(Color.WHITE);
    wavePaint2.setAlpha(60);
    wavePaint2.setAntiAlias(true);

    while (waveX1 < endX) {
        int waveX2 = (waveX1 + wave2Shift) % endX;
        double wx = waveX1 * mDefaultAngularFrequency;

        int startY = (int) (mDefaultWaterLevel + mDefaultAmplitude * Math.sin(wx));

        // draw bottom wave with the alpha 40
        canvas.drawLine(waveX1, startY, waveX1, endY, wavePaint1);
        // draw top wave with the alpha 60
        canvas.drawLine(waveX2, startY, waveX2, endY, wavePaint2);

        waveX1++;
    }

    // use the bitamp to create the shader
    mWaveShader = new BitmapShader(bitmap, Shader.TileMode.REPEAT, Shader.TileMode.CLAMP);
    mViewPaint.setShader(mWaveShader);
}
```

### 使用方法

## 源代码
[Jekyll Now repository](https://github.com/barryclark/jekyll-now)
