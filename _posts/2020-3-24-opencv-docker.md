---
layout: post
title: "Show images from inside docker container"
---
Docker and OpenCV are both great tools. Docker can be especially useful for development. [OpenCV](https://opencv.org) is the defacto gold standard in developing computer vision applications. However, it can be a little challanging at first to set up these tools to work together. More specifically to display images from inside the container on the host when calling opencv functions such as imshow. This post will demonstrate by example one way to get imshow working with a simple c++ application, while installing and running  OpenCV inside a docker container.

So.. lets get into it.

Create a new project folder named whatever you like, inside of it create a file named Dockerfile.

Copy one of the two options below into your Dockerfile you just created.

**The first example builds OpenCV from source in docker container for your reference, it will take a while to complete should you build yourself.**
**The second pulls down my already built version of the first one from Dockerhub.**

{% highlight docker %}
FROM debian:buster

# Install Dependencies
RUN apt-get update && apt-get install build-essential -y
RUN apt-get install \
    cmake git pkg-config libgtk-3-dev \
    libavcodec-dev libavformat-dev libswscale-dev libv4l-dev \
    libxvidcore-dev libx264-dev libjpeg-dev libpng-dev libtiff-dev \
    gfortran openexr libatlas-base-dev python3-dev python3-numpy \
    libtbb2 libtbb-dev libdc1394-22-dev -y
# Create OPENCV on a layer
RUN git clone https://github.com/opencv/opencv.git && \
    git clone https://github.com/opencv/opencv_contrib.git && \
    cd /opencv && git checkout 4.2.0 && \
    cd /opencv_contrib && git checkout 4.2.0 && \
    cd /opencv && \
    mkdir build && \
    cd /opencv/build && \
    cmake -D CMAKE_BUILD_TYPE=RELEASE \
        -D CMAKE_INSTALL_PREFIX=/usr/local .. \
        -D INSTALL_C_EXAMPLES=ON \
        -D INSTALL_PYTHON_EXAMPLES=ON \
        -D OPENCV_GENERATE_PKGCONFIG=ON \
        -D OPENCV_EXTRA_MODULES_PATH=../../opencv_contrib/modules \
        -D BUILD_EXAMPLES=ON .. && \
    make -j5 && make install && \
    cd / && \
    rm opencv -rf && \
    rm opencv_contrib -rf

{% endhighlight %}

{% highlight docker %}
From firstcaptain/opencv:4.2.0

RUN apt-get install -y \
     scons \
     libboost-all-dev
{% endhighlight %}


To get this built first you'll need to install docker
[Install Docker](https://docs.docker.com/install/ "Install Docker")

Once docker is properly installed run this command in your terminal from the same directory as your Dockerfile.
{% highlight bash %}
docker build -t <tagname> .
{% endhighlight %}

### Now to create some c++ code

Inside the same directory create a folder called app.
Then inside app create another folder called src.
Inside of src you will create main.cpp and inside of it paste the following code.

{% highlight c++ %}
#include <opencv2/core/core.hpp>
#include <opencv2/imgcodecs.hpp>
#include <opencv2/highgui/highgui.hpp>
#include <iostream>

using namespace std;
using namespace cv;

int main()
{
    Mat image = imread("test.png", IMREAD_COLOR);
    cout << image.size << endl;
    imwrite("out.png",image);
    namedWindow("TEST", WINDOW_NORMAL );
    imshow("TEST", image );
    waitKey(0);
    return 0;
}
{% endhighlight %}

inside of app directory create a file named SConstruct and paste in the below contents.

{% highlight python %}
libs = ['opencv_core','opencv_imgproc','opencv_highgui','opencv_imgcodecs']

env=Environment(CPPPATH=['/usr/include/boost/', '/usr/local/include/opencv4'],
    CPPDEFINES=[],
    LIBS=libs,
    SCONS_CXX_STANDARD="c++11"
    )

env.Program('test', Glob('src/*.cpp'))
{% endhighlight %}

In addition copy over a sample png of your choosing and name it test.png into to the folder you should have the following folder structure.


{% highlight bash %}
project
│   Dockerfile
└───app
│   │   SConstruct
|   |   test.png
│   └───src
│       │   main.cpp
│       │
{% endhighlight %}


Finally to run use this command.

{% highlight bash %}
docker run -it --rm -e DISPLAY=$DISPLAY \
   -v /tmp/.X11-unix:/tmp/.X11-unix \
   --network=host \
   --volume=/data/vision/app:/app \
   --user=$(id -u $USER) \
   --workdir=/app opencv /bin/bash
{% endhighlight %}

After running you will see that it says no user specified inside the docker container. This can be addressed but I havn't find it necessary for my purposes in local development with OpenCV.

Inside your docker container you can run to build and run your code.

{% highlight bash %}
scons && ./test
{% endhighlight %}

And vuela you should see the test.png image on your host machine. I hope this helps you.
[Code in Github](https://github.com/FirstCaptain/opencv-docker)
