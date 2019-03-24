---
title: 树莓派下编译安装OpenCV-3.4.1
date: 2019-02-08 13:00:58
categories:
- OpenCV
tags:
- raspberryPi
- OpenCV
copyright: true
---

安装了OpenCV-3.4.1还有对应的contrib

<!-- more -->

***
1. **首先建议先更换软件源，这里以清华的源为例**

* 使用管理员权限，编辑/etc/apt/sources.list文件
```shell
sudo nano /etc/apt/sources.list
```
* 用#注释掉原文件的内容，用以下内容取代
```shell
deb http://mirrors.tuna.tsinghua.edu.cn/raspbian/raspbian/ stretch main contrib non-free rpi
deb-src http://mirrors.tuna.tsinghua.edu.cn/raspbian/raspbian/ stretch main contrib  non‐free rpi
```
* 使用管理员权限，编辑/etc/apt/sources.list.d/raspi.list
```shell
sudo nano /etc/apt/sources.list.d/raspi.list
```
* 用#注释掉原文件的内容，用以下内容取代
```shell
deb http://mirror.tuna.tsinghua.edu.cn/raspberrypi/ stretch main ui
deb‐src http://mirror.tuna.tsinghua.edu.cn/raspberrypi/ stretch main ui
```
* 更新软件源列表
```shell
sudo apt-get update
```
2. **为了充分使用整个存储空间，输入以下命令，进入到软件配置工具后，选择第七项 "Advanced Options" ，再选择 A1 项 "Expand Filesystem" 即可**
```shell
sudo raspi-config
```
3. **先更新一下**

 * 软件源更新
```shell
sudo apt‐get update
```
 * 升级本地所有安装包
```shell
sudo apt‐get upgrade
```
 * 升级树莓派固件
```shell
sudo rpi‐update
```
4. **安装构建OpenCV的相关工具**
```shell
sudo apt‐get install build‐essential cmake git pkg‐config
```
5. **安装常用图像工具包**
```shell
#安装jpeg格式图像工具包 
sudo apt‐get install libjpeg8‐dev
#安装tif格式图像工具包 
sudo apt‐get install libtiff5‐dev
#安装JPEG‐2000格式图像工具包 
sudo apt‐get install libjasper‐dev
#安装png格式图像工具包 
sudo apt‐get install libpng12‐dev
```
6. **安装视频I/O包（注意最后一个包的数字“4”后面是“L”)**
```shell
sudo apt‐get install libavcodec‐dev libavformat‐dev libswscale‐dev libv4l‐dev
```
7. **安装gtk2.0**
```shell
sudo apt‐get install libgtk2.0‐dev
```
8. **安装优化函数包**
```shell
sudo apt‐get install libatlas‐base‐dev gfortran
```
9. **下载OpenCV源码（放在家目录下）**
```shell
wget ‐O opencv‐3.4.1.zip https://github.com/Itseez/opencv/archive/3.4.1.zip
unzip opencv‐3.4.1.zip
wget ‐O opencv_contrib‐3.4.1.zip  https://github.com/Itseez/opencv_contrib/archive/3.4.1.zip
unzip opencv_contrib‐3.4.1.zip
```
10. **进入源码文件夹，新建一个名为 release 的文件夹用来存放 cmake 编译时产生的临时文件**
```shell
cd opencv-3.4.1
mkdir release
cd release
```
11. **设置 cmake 编译参数，安装目录默认为 /usr/local , 注意参数名，等号和参数值之间不能有空格，但每行末尾 “\” 之前有空格，参数值最后是两个英文的点，如果cmake下载缺失文件失败的话，就将release文件夹删掉，从第十步重新开始**

```shell
sudo cmake ‐D CMAKE_BUILD_TYPE=RELEASE \
‐D CMAKE_INSTALL_PREFIX=/usr/local \
‐D OPENCV_EXTRA_MODULES_PATH=~/opencv_contrib‐3.4.1/modules \
‐D INSTALL_PYTHON_EXAMPLES=ON \
‐D BUILD_EXAMPLES=ON ..
```
12. **开始编译**
```shell
#编译
sudo make
#安装
sudo make install
#更新动态链接库
sudo ldconfig
```
13. **编译安装完了应该就可以了，可以写个程序测试一下**

***
参考链接：

* [树莓派OpenCV3.4.1安装血泪史，分享如何规避各种坑](https://blog.csdn.net/jacka654321/article/details/80728795)
* [(树莓派、Linux通用)OpenCV3源码方式安装（最新3.4.1）](https://blog.csdn.net/leaves_joe/article/details/67656340)
* [为树莓派更换国内镜像源](https://blog.csdn.net/la9998372/article/details/77886806)

