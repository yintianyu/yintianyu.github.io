---
layout: post
title: "移植OpenCV到RISC-V上"
date: 2019-10-13 10:59:13 +0800
author: "尹天宇"
tags:
    - OpenCV
    - Port
---

本文讲述如何将Opencv移植到RISC-V平台上。目前（2019年10月）RISC-V的软件生态已经较为完善了；Linux平台除了Buildroot外，Fedora发行版也已经支持RISC-V。
本文描述将OpenCV通过交叉编译的方式运行在RISC-V架构的Linux系统中。

本文所使用的OpenCV的版本为4.1.1，Linux为Fedora 28，其他支持RISC-V的Linux的发行版以及Buildroot等理论上也可用（需要注意的事项见下文），运行平台为QEMU（FPGA、SiFive Unleashed应该也可以），riscv-gnu-toolchain为2019年10月2日的最新版（经过修改，见下文）。编译环境为Ubuntu 16.04。

## 前提准备

首先要注意的是运行平台Linux的glibc版本，本文使用的Fedora 28中的glibc版本为2.27。
你使用的交叉编译工具的glibc版本必须和目标平台的glibc版本相同，因为glibc非常底层，几乎所有的程序都需要使用它（包括最常用的ls, cat等命令），因此一般不会在发行版不支持等情况下升级（或降级）glibc。

目前riscv-glibc支持的版本有2.26, 2.27和2.29三种，而riscv-gnu-toolchain中包含的glibc在2018年8月以前的版本为2.26，在2018年8月以后的版本为2.29，无官方2.27版本的支持。

为了在clone最新的[riscv-gnu-toolchain](https://github.com/riscv/riscv-gnu-toolchain)代码（及其子模块）后，进入`riscv-glibc`目录，手动输入`git checkout riscv-glibc-2.27`切换到2.27版本。
然后修改`riscv-glibc/sysdeps/unix/sysv/linux/riscv/flush-icache.c`，将其原有的
```c
#include <asm/syscalls.h>
```
修改为
```c
#if __has_include__ (<asm/syscalls.h>)
# include <asm/syscalls.h>
#else
# include <asm/unistd.h>
#endif
```
之后再按照`readme.md`中的步骤生成linux版（非裸机版）的toolchain。



## 编译OpenCV

将OpenCV代码下载到opencv目录中，本文中例子为`opencv/opencv-4.1.1`。
在`opencv`文件夹中建立两个文件夹，分别为`config`和`install`（取其他名字也可以）。

由于OpenCV是使用cmake进行编译配置，为了方便配置，我们安装一下cmake-gui。
```sh
sudo apt install cmake-qt-gui
```

在终端中输入`cmake-gui`启动cmake的图形界面。
在Where is the source code中输入opencv-4.1.1的目录，在Where to build the binaries输入建立的config目录。如下图所示。
<!--插入图片-->
![](/img/porting-opencv-1.png)

点击Configure，打开如下图所示的对话框。选择Specify options for cross-compiling.
<!--插入图片-->
![](/img/porting-opencv-2.png)

在Operating System中输入Linux，在Version中输入目标平台的版本（不是很重要）。
然后编译器C选择安装好的riscv64-unknown-linux-gnu-gcc编译器，C++选择riscv64-unknown-linux-gnu-g++。Target Root选择riscv-gnu-toolchain安装目录，如下图所示。
<!--插入图片-->
![](/img/porting-opencv-3.png)

点击Finish，即可开始配置，自动配置后如下图所示。需要手动去掉`WITH_GTK`和`WITH_TIFF`选项（本文暂不支持OpenCV的图形界面，后续按需可能会探索）。将`CMAKE_INSTALL_PREFIX`的值改为上面创建的`install`目录（或其他你想安装的目录，最好不要与系统本身的库目录在一起，因为这是RISC-V平台的交叉编译）。
<!--插入图片-->
![](/img/porting-opencv-4.png)

之后点击Generate即可生成Makefile，然后可以关闭cmake-gui窗口。

打开终端进入config目录，输入
```sh
make -jx
```
进行编译，其中x为处理器的线程数。

在编译过程中如果遇到了个别头文件找不到的问题，其实这些头文件是存在于/usr/include目录中的（前提是你的确安装了对应的库），我将其在riscv-gnu-toolchain安装目录的include目录中创建软连接，即可解决。如果遇到其他问题可按照一般的编译问题处理方法进行排除。

一段时间后（不需要太久）即可编译成功。

之后`make install`进行安装。

## 编译demo程序
本文写了一小段使用OpenCV进行图像膨胀操作的程序`demo-dilate.cpp`，代码如下。
```cpp
#include <opencv2/core/core.hpp>
#include <opencv2/highgui/highgui.hpp>
#include <opencv2/imgproc/imgproc.hpp>
#include <iostream>
 
using namespace std;
using namespace cv;
 
int main(  )
{
 
    //载入原图 
    Mat image = imread("1.jpg");

    //获取自定义核
    Mat element = getStructuringElement(MORPH_RECT, Size(15, 15));
    Mat out;
    //进行膨胀操作
    dilate(image,out, element);

    imwrite("1-dilate.jpg", out);

    //waitKey(0);

    return 0;
}

```

使用如下命令进行编译链接，注意编译器要使用正确的版本。
```sh
riscv64-unknown-linux-gnu-g++ -o dilate demo-dilate.cpp \
-I/usr/local/include -I/path/install/opencv/include/opencv4 \
-L/path/install/opencv/lib -lopencv_core -lopencv_highgui -lopencv_imgproc\
-lopencv_ml -lpthread -lopencv_calib3d -lopencv_dnn -lopencv_features2d -lopencv_flann\
-lopencv_gapi -lopencv_imgcodecs -lopencv_objdetect -lopencv_photo -lopencv_stiching -lopencv_videoio -lopencv_video
```

生成dilate可执行文件备用。


## 启动QEMU的Fedora系统
如何在RISCV64的QEMU上运行Fedora磁盘镜像本文不再赘述，请读者参考[这个文档](https://wiki.qemu.org/Documentation/Platforms/RISCV#Booting_64-bit_Fedora)。

使用SFTP将编译出来的opencv库放到目标Linux系统的`/lib64`目录中，将可执行文件`dilate`以及执行需要的`1.jpg`放到`/root`中。
此时执行`./dilate`会提示GLIBCXX_3.4.26 not found，这是因为我们只匹配了GLIBC的版本，而对于C++标准库并没有匹配。幸运的是，
对于GLIBC++的升级是比较容易的，并没有GLIBC那么困难（容易出错）。

将宿主机（Ubuntu）上安装toolchain目录中的`sysroot/lib`中的`libstdc++.so`、`libstdc++.so.6`和`libstdc++.so.6.0.27`拷贝到
Fedora的某个目录中，比如`/root/lib`，然后设置环境变量`export LD_LIBRARY_PATH=/root/lib:$LD_LIBRARY_PATH`。

之后即可成功运行`./dilate`。

效果展示如下：\\
原图`1.jpg`：
![](/img/porting-opencv-demo-1.jpg)

膨胀后`1-dilate.jpg`：
![](/img/porting-opencv-demo-1-dilate.jpg)



