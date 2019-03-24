---
title: minAreaRect函数返回值说明
date: 2019-02-08 13:08:33
categories:
- OpenCV
tags:
- OpenCV
copyright: true
---

最近在看代码的时候发现 OpenCV 里面 minAreaRect() 这个函数返回的旋转矩形的 angle, height, width 有点让人困惑，不清楚为什么它们的值是那么多，网上找的解释竟然解释的还不一样，于是就自己写代码验证了一下，我也不知道对不对，欢迎评论指正

<!-- more -->

### 先放几张测试图片
![1.jpg](http://i350.photobucket.com/albums/q439/wood_/1_zpszzciuets.jpg)

![2.jpg](http://i350.photobucket.com/albums/q439/wood_/2_zpsmwe577ot.jpg)

![3.jpg](http://i350.photobucket.com/albums/q439/wood_/3_zpsmic8iuin.jpg)

![4.jpg](http://i350.photobucket.com/albums/q439/wood_/4_zpsiwk22nzz.jpg?t=1538449392)

### Result
其实从图片上看，结果已经很明确了，minAreaRect() 返回的旋转矩形有如下特征

* angle = 过旋转矩形中心的直线(绿线)逆时针旋转，当它第一次与旋转矩形的边垂直时，该直线旋转过的角度的相反数即为 angle 的值

* width = 与旋转直线(绿线)第一次垂直的那两条边的长度，即为 width 的值

* height = 不是 width 的那两条边的长度，即为 height 的值咯

### Code
``` cpp
#include <opencv2/opencv.hpp>
#include <vector>

using namespace cv;

int main()
{
    Mat img(500, 500, CV_8UC3);
    RNG &rng = theRNG();
    while (1) {
        int count = rng.uniform(1, 101);
        std::vector<Point2f> points;

        for (int i = 0; i < count; ++i) {
            Point2f pt;
            pt.x = rng.uniform(img.size().width / 4, img.size().width * 3 / 4);
            pt.y = rng.uniform(img.size().height / 4, img.size().height * 3 / 4);
            points.push_back(pt);
        }

        RotatedRect rect = minAreaRect(Mat(points));
        Point2f rectPoints[4];
        rect.points(rectPoints);

        img = Scalar::all(0);

        for (int i = 0; i < count; ++i) {
            circle(img, points[i], 3, Scalar(0, 0, 255), FILLED, LINE_AA);
        }

        for (int i = 0; i < 4; ++i) {
            line(img, rectPoints[i], rectPoints[(1 + i) % 4], Scalar(255, 0, 0), 3, LINE_AA);
        }

        line(img, Point2f(rect.center.x, 0), Point2f(rect.center.x, img.size().height), Scalar(0, 255, 0), 2, LINE_AA);

        char text[50];
        sprintf(text, "angle: %.2f, width: %.2f, height: %.2f", rect.angle, rect.size.width, rect.size.height);
        putText(img, text, Point(2, 15), CV_FONT_NORMAL, 0.5, Scalar(255, 255, 255));

        imshow("img", img);

        if (waitKey(0) == 27) {
            break;
        }
    }
    return 0;
}
```