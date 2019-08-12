---
title: Luban压缩实现分析
date: 2019-08-12 15:06:52
categories: 
- Android
    - 图片
tags:
- Android
- Luban
- 图片
- 压缩
---

# Luban压缩实现分析

## 前言

[Luban](https://github.com/Curzibn/Luban)是一个非常好用的`Android`图片压缩库，据作者所言，作者逆向推算了微信的压缩算法，压缩结果很接近微信朋友圈压缩后的效果。在实际使用中，压缩的结果大大缩小了从相册选择出来的图片，质量也几乎没有太大差别。

## 压缩流程

![Luban压缩流程图](Luban压缩流程图.jpg)

鲁班压缩的核心非常简单，最重要的就两个类`Luban.java`和`Engine.java`，核心算法主要是在`Engine.java`

### Luban.java

`Luban.java`的作用主要是进行一些压缩的配置，如下
1. 设置图片源
2. 设置压缩后图片的存储文件夹
3. 压缩后图片的名字
4. 可压缩图片的最小文件大小
5. 设置压缩选择器，用户可根据图片，决定是否压缩

### Engine.java
`Engine.java`处理真正的压缩
1. 计算压缩比例
2. 压缩图片

## 问题

因为`Android 7.0`限制了获取图片的方式，我们获取到的图片地址由`file://...`变成了`content://...`，而且是不能通过`content://...`得到对应文件的。

在`Luban`决定图片是否应该被压缩时，仍然是将其转换为`File`进行处理，因为时无效`File`，所以这时就出现了`bug`，图片并没有被压缩，得到的也是无效的`File`对象。这时可以设置可压缩图片的最小文件大小为-1，`ignoreBy(-1)`来解决这个问题，这样设置时，会直接去压缩，并不会去判断`File`文件的大小，所以就避免了这个问题

知道这个`bug`也自然可以解决。在`Android`平台上，操作图片、文件最好都是用`Uri`来进行处理，会避免很多的问题，如果在使用一个图片选择库、文件选择库、图片剪切库时，返回的是一个`Uri`，不要害怕。

## 压缩算法

作者公开了压缩算法，其中主要根据图片尺寸来计算压缩后的图片尺寸，同时设置压缩质量为`60%`

作者的压缩算法：
> 1. 判断图片比例值，是否处于以下区间内；
> * [1, 0.5625) 即图片处于 [1:1 ~ 9:16) 比例范围内
> * [0.5625, 0.5) 即图片处于 [9:16 ~ 1:2) 比例范围内
> * [0.5, 0) 即图片处于 [1:2 ~ 1:∞) 比例范围内
> 2. 判断图片最长边是否过边界值；
> * [1, 0.5625) 边界值为：1664 * n（n=1）, 4990 * n（n=2）, 1280 * pow(2, n-1)（n≥3）
> * [0.5625, 0.5) 边界值为：1280 * pow(2, n-1)（n≥1）
> * [0.5, 0) 边界值为：1280 * pow(2, n-1)（n≥1）
> 3. 计算压缩图片实际边长值，以第2步计算结果为准，超过某个边界值则：width / pow(2, n-1)，height/pow(2, n-1)
> 4. 计算压缩图片的实际文件大小，以第2、3步结果为准，图片比例越大则文件越大。
> * size = (newW * newH) / (width * height) * m；
> * [1, 0.5625) 则 width & height 对应 1664，4990，1280 * n（n≥3），m 对应 150，300，300；
> * [0.5625, 0.5) 则 width = 1440，height = 2560, m = 200；
> * [0.5, 0) 则 width = 1280，height = 1280 / scale，m = 500；注：scale为比例值
> 5. 判断第4步的size是否过小
> * [1, 0.5625) 则最小 size 对应 60，60，100
> * [0.5625, 0.5) 则最小 size 都为 100
> * [0.5, 0) 则最小 size 都为 100
> 6. 将前面求到的值压缩图片 width, height, size 传入压缩流程，压缩图片直到满足以上数值

## java实现
既然有了算法，那么就自己实现一遍，因为只是计算尺寸，所以实现就非常简单了

下面是我的java实现，用到了`google`的开源库[thumbnailtor](http://code.google.com/p/thumbnailator)，在服务端，若发现客户端上传的图片过大，可以使用这样的算法进行压缩

```java
import net.coobird.thumbnailator.Thumbnails;

import javax.imageio.ImageIO;
import java.awt.image.BufferedImage;
import java.io.File;
import java.io.IOException;

/**
 * Luban压缩算法Java实现
 * 
 * @author mycroft
 */
public class FileCompressor {

    /**
     * 压缩图片文件
     * 
     * @param sourceFile    需要进行压缩的图片
     * @param destFile      压缩后的图片位置
     */
    public static void compressFile(File sourceFile, File destFile) throws IOException {
        if (null == sourceFile || !sourceFile.exists()) {
            return;
        }

        BufferedImage image = ImageIO.read(sourceFile);
        int width = image.getWidth();
        int height = image.getHeight();
        if (width == 0 || height == 0) {
            return;
        }

        int scale = calculateSize(width, height);

        int destWidth = width / scale;
        int destHeight = height / scale;

        Thumbnails.of(image).size(destWidth, destHeight).outputQuality(0.6f).toFile(destFile);
    }

    /**
     * 根据图片宽高计算压缩尺寸
     * 
     * @param srcWidth      图片宽度
     * @param srcHeight     图片高度
     * @return 压缩比例
     */
    private static int calculateSize(int srcWidth, int srcHeight) {
        srcWidth = srcWidth % 2 == 1 ? srcWidth + 1 : srcWidth;
        srcHeight = srcHeight % 2 == 1 ? srcHeight + 1 : srcHeight;

        int longSide = Math.max(srcWidth, srcHeight);
        int shortSide = Math.min(srcWidth, srcHeight);

        float scale = ((float) shortSide / longSide);
        if (scale <= 1 && scale > 0.5625) {
            if (longSide < 1664) {
                return 1;
            } else if (longSide >= 1664 && longSide < 4990) {
                return 2;
            } else if (longSide > 4990 && longSide < 10240) {
                return 4;
            } else {
                return longSide / 1280 == 0 ? 1 : longSide / 1280;
            }
        } else if (scale <= 0.5625 && scale > 0.5) {
            return longSide / 1280 == 0 ? 1 : longSide / 1280;
        } else {
            return (int) Math.ceil(longSide / (1280.0 / scale));
        }
    }
}
```

## 后话

压缩实现非常简单，但是据作者所言，其在朋友圈发送了100张图片，进行对比逆向推算出来的算法，并且公开了算法，这是非常敬佩的。

同时这个算法实现非常简单，如有需要，可以在其他平台和语言实现，如`windows`, `iOS`, `flutter`