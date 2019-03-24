---
title: Ubuntu16.04LTS下编译安装OpenCV
date: 2019-02-08 13:05:45
categories:
- OpenCV
tags:
- Ubuntu
- OpenCV
copyright: true
---

这篇教程是陈明聪学长写的，这里放在我的博客上供以后参考

添加了 OpenCV_Contrib 的编译安装

<!-- more -->

### Pre.前期准备

下载OpenCV 和 OpenCV_Contrib 源码包
进入[OpenCV官方下载页面](https://opencv.org/releases.html)
选择Sources类型下载
OpenCV_Contrib 的源码去 Github 上找找，我也忘了我在哪下的了，下的版本要跟 OpenCV 一样...

### 1.安装依赖包

```shell
sudo apt-get install cmake
sudo apt-get install build-essential libgtk2.0-dev libavcodec-dev libavformat-dev libjpeg.dev libtiff4.dev libswscale-dev libjasper-dev
```

### 2.编译OpenCV

解压之前下载好的源码包
```shell
unzip opencv-3.4.3.zip  #这里以opencv-3.4.3为例
unzip opencv_contrib-3.4.3.zip -d /opencv-3.4.3 #把 opencv_contrib 解压到 opencv 源码目录下
```
进入OpenCV源码目录
```shell
cd opencv-3.4.3
mkdir build
cd build
cmake -D CMAKE_BUILD_TYPE=Release -D CMAKE_INSTALL_PREFIX=/usr/local -D OPENCV_EXTRA_MODULES_PATH=../opencv_contrib-3.4.3/modules ..
sudo make -j4
sudo make install
```
### 3.添加路径
首先将OpenCV的库添加到路径，从而可以让系统找到
```shell
sudo gedit /etc/ld.so.conf.d/opencv.conf
```
执行此命令后打开的可能是一个空白文件，不用管，只用在文件末尾添加
```shell
/usr/local/lib
```
执行如下命令使得刚才的配置路径生效
```shell
sudo ldconfig
```
配置bash
```shell
sudo gedit /etc/bash.bashrc
```
在最末尾添加
```shell
PKG_CONFIG_PATH=$PKG_CONFIG_PATH:/usr/local/lib/pkgconfig
export PKG_CONFIG_PATH
```
保存，执行如下命令使得配置生效
```shell
source /etc/bash.bashrc
```
更新
```shell
sudo updatedb
```
### 4.测试
至此所有的配置都已经完成
下面用一个小程序测试一下
cd到opencv-3.4.3/samples/cpp/example_camke目录下
我们可以看到这个目录下官方已经给出了一个cmake的example我们可以拿来测试下
按顺序执行

```shell
cmake .
make
./opencv_example
```
即可看到打开了摄像头，在左上角有一个hello opencv，即表示配置成功

