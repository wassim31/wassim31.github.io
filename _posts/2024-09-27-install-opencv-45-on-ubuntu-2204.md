---
layout: post
title: "Install OpenCV 4.5 on Ubuntu 22.04"
date: 2024-09-27 05:50:45 +0000
description: "OpenCV installation can be tricky on Linux. A correct process for building OpenCV 4.5 from source, for both Python and C++."
canonical_devto: https://dev.to/wassim31/install-opencv-45-on-ubuntu-2204-306i
---

OpenCV (Open Source Computer Vision Library) is a library of programming functions mainly for real-time computer vision.

But its installation can be very tricky in an environment like Linux, so let's follow a correct installation process.

## Download the sources

The most recent release at the time of writing is [4.5.1](https://github.com/opencv/opencv/releases/tag/4.5.1).

Create a tmp folder for all the archives:

```bash
mkdir ~/opencv4.5-tmp && cd ~/opencv4.5-tmp
```

Download the OpenCV sources and the opencv-contrib sources:

```bash
wget https://github.com/opencv/opencv/archive/4.5.1.zip -O opencv.zip
wget https://github.com/opencv/opencv_contrib/archive/4.5.1.zip -O opencv_contrib.zip
```

Unzip both archives:

```bash
unzip opencv.zip
unzip opencv_contrib.zip
```

Move the files to simpler directory names:

```bash
mv opencv-4.5.1/ opencv
mv opencv_contrib-4.5.1/ opencv_contrib
```

## Build and install

Make a build directory:

```bash
cd opencv && mkdir build && cd build
```

Copy and run the following command (install cmake first if it is not available on your system):

```bash
cmake -D CMAKE_BUILD_TYPE=DEBUG \
      -D CMAKE_INSTALL_PREFIX=~/opencv4.5-custom \
      -D OPENCV_EXTRA_MODULES_PATH=~/opencv4.5-tmp/opencv_contrib/modules \
      -D OPENCV_GENERATE_PKGCONFIG=ON \
      -D BUILD_EXAMPLES=ON ..
```

Make the project:

```bash
make -j4
```

Install OpenCV:

```bash
sudo make install
```

Ensure it is updated in the library storage:

```bash
sudo ldconfig
```

## Configure a C++ project to work with OpenCV

Open your editor of choice (vim in my case), create a folder `~/projects/HelloOpenCV`, and put your code in `main.cpp`.

First, let's try to compile the application with g++ as usual:

```bash
g++ -Wall -o main main.cpp
```

We see that it cannot find our library headers.

For that we need to provide the path to the headers and the linker flags. The best way to find them all in one place is the pkg-config utility. Remember we provided an additional argument to our cmake generation? So let's execute the following:

```bash
export PKG_CONFIG_PATH=$PKG_CONFIG_PATH:/home/parallels/opencv4.5-custom/lib/pkgconfig
pkg-config --cflags --libs opencv4
```

Now add all the flags to the compilation command:

```bash
g++ -Wall -o main main.cpp $(pkg-config --cflags --libs opencv4)
```

Or a more explicit version for our particular sample:

```bash
g++ -Wall -o main main.cpp \
   -I/home/parallels/opencv4.5-custom/include/opencv4 \
   -L/home/parallels/opencv4.5-custom/lib \
   -lopencv_highgui -lopencv_videoio -lopencv_imgcodecs -lopencv_core
```
