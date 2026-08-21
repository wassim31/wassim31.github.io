---
layout: post
title: "Install OpenCV 4.5 on Ubuntu 22.04"
date: 2024-09-27 05:50:45 +0000
description: "OpenCV (Open Source Computer Vision Library) is a library of programming functions mainly for..."
canonical_devto: https://dev.to/wassim31/install-opencv-45-on-ubuntu-2204-306i
---

OpenCV (Open Source Computer Vision Library) is a library of programming functions mainly for real-time computer vision.

**But it's installation can be very tricky in an environment like Linux, so let's follow a correct installation process**


### Build OpenCV for Python and C++ from sources


1. Let's open the most recent release of opencv to the date of this video capturing:
   https://github.com/opencv/opencv/releases/tag/4.5.1

2. Create a tmp folder for all archives:
   `mkdir ~/opencv4.5-tmp && cd ~/opencv4.5-tmp`

3. We need to download opencv sources:
   `wget https://github.com/opencv/opencv/archive/4.5.1.zip -O opencv.zip`

4. We need to download opencv-contrib sources:
   `wget https://github.com/opencv/opencv_contrib/archive/4.5.1.zip -O opencv_contrib.zip`

5. Unzip the opencv files:
   `unzip opencv.zip`

6. Unzip the opencv-contrib files:
   `unzip opencv_contrib.zip`

7. Move the files to simple directories:
   `mv opencv-4.5.1/ opencv`

8. Move opencv-contrib files to simple directories:
   `mv opencv_contrib-4.5.1/ opencv_contrib`

9. Make build directory:
   `cd opencv && mkdir build && cd build`

10. Copy and run the following command. Install cmake if it is not available on the system.

   ```bash
   cmake -D CMAKE_BUILD_TYPE=DEBUG \
         -D CMAKE_INSTALL_PREFIX=~/opencv4.5-custom \
         -D OPENCV_EXTRA_MODULES_PATH=~/opencv4.5-tmp/opencv_contrib/modules \
         -D OPENCV_GENERATE_PKGCONFIG=ON -DOPENCV_GENERATE_PKGCONFIG=ON \
         -D BUILD_EXAMPLES=ON ..
   ```

1. Make the project:
   ```
   make -j4
   ```
1. Install opencv:
   ```
   sudo make install
   ```
1. Ensure that it is updated in the library storage:
   ```
   sudo ldconfig
   ```

   ## Configure C++ project to work with opencv

1. Open your editor of choice (vim in my case)
1. Create a folder "~/projects/HelloOpenCV"
1. Paste the `main.cpp` code to the `main.cpp`
1. Firstly, let's try to compile our application with g++ as usual:
   ```
   g++ -Wall -o main main.cpp
   ```
   We see that it cannot find our library headers
1. For that we need to provide path to the headers and linker flags. The best way to find them
   all in one place is to use the utility module pkg-config. Remember we provided an additional
   argument to our cmake generation? So, let's execute the following:
   ```
   export PKG_CONFIG_PATH=$PKG_CONFIG_PATH:/home/parallels/opencv4.5-custom/lib/pkgconfig
   pkg-config --cflags --libs opencv4
   ```
1. Add all flags to compilation command:
   ```
   g++ -Wall -o main main.cpp $(pkg-config --cflags --libs opencv4)
   ```
   or a more explicit version for our particular sample:
   ```
   g++ -Wall -o main main.cpp \
      -I/home/parallels/opencv4.5-custom/include/opencv4 \
      -L/home/parallels/opencv4.5-custom/lib \
      -lopencv_highgui -lopencv_videoio -lopencv_imgcodecs -lopencv_core
   ```
