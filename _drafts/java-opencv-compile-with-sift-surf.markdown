---
layout: post
title:  "How to compile OpenCV with SIFT and SURF support, and use them in a Java program"
date:   2021-01-27 18:00:00 +0100
categories: opencv java
---
## Image features introduction
As [Wikipedia](https://en.wikipedia.org/wiki/Feature_(computer_vision)) states:

> In computer vision and image processing, a feature is a piece of information about the content of an image; typically about <u>whether a certain region of the image has certain properties</u>.

However, [it also states](https://en.wikipedia.org/wiki/Feature_detection_(computer_vision)):

> There is <u>no universal or exact definition</u> of what constitutes a feature, and the exact definition often depends on the problem or the type of application. Nevertheless, a feature is typically defined as an <u>"interesting" part of an image</u>, and features are used as a starting point for many computer vision algorithms.

The result is something like the following:

![](https://upload.wikimedia.org/wikipedia/commons/8/8d/Writing_Desk_with_Harris_Detector.png)
*Source: https://commons.wikimedia.org/wiki/File:Writing_Desk_with_Harris_Detector.png*

There are many algorithms to perform this task. In particular, [SIFT](https://en.wikipedia.org/wiki/Scale-invariant_feature_transform) and [SURF](https://en.wikipedia.org/wiki/Speeded_up_robust_features) are two very popular choices. However, since they are both patented algorithms, even if the source code is available, they are not included by default in OpenCV.

In this post I describe how to build [OpenCV](https://opencv.org) (version 4 or greater) from source with built-in support for these non-free algorithms.

I also show how to build, configure and use the __OpenCV Java wrapper__.

Note that, if you have working __Python__ installation, the build process describe below will also build the OpenCV Python library. It will be installed into path `lib/python3.x/site-packages/cv2` of the `dist` directory. However, in this post I will not describe how to install and use it.

## Prerequisites
To compile OpenCV you will need the following:
- A C/C++ compiler
- [cmake](https://cmake.org)
- The Java JDK
- [Apache Ant](http://ant.apache.org/)

## Downloading OpenCV source code
In this tutorial I'm using version `4.1.2`. I expect that other versions (in particular `4.x`) would work the same way.

Download the following tarballs:
- [https://github.com/opencv/opencv_contrib/archive/4.1.2.tar.gz](https://github.com/opencv/opencv_contrib/archive/4.1.2.tar.gz)
- [https://github.com/opencv/opencv/archive/4.1.2.tar.gz](https://github.com/opencv/opencv/archive/4.1.2.tar.gz)

Uncompress both files to the same path. You should have the following directories:
```
opencv-4.1.2
opencv_contrib-4.1.2
```

## Build OpenCV and Java wrapper
Create the build directory and `cd` to it:
```
cd opencv-4.1.2
mkdir build
cd build
```
Run `cmake` to configure the build process:
```
cmake -DOPENCV_ENABLE_NONFREE:BOOL=ON -DBUILD_SHARED_LIBS=OFF \
  -DCMAKE_BUILD_TYPE=Release -DCMAKE_INSTALL_PREFIX=../dist \
  -DOPENCV_EXTRA_MODULES_PATH=../opencv_contrib/modules ..
```
This will show a long output, but shouldn't take too long. To be sure that everything is configured correctly, you have to check that the following messages are printed:
```
--   Extra modules:
--     Location (extra):            <some path>/opencv_contrib-4.1.2/modules
...
--     Non-free algorithms:         YES
...
--   Java:
--     ant:                         <some path>
--     JNI:                         <some path>
--     Java wrappers:               YES
--     Java tests:                  YES
```

Now run the actual build with `make`. You can choose how many compiler processes to run in parallel (with the `-j` argument), depending to how many CPUs you have. Note that this may take a while.
```
make -j4
```

Finally, install the build result into the `dist` directory and move it somewhere (this is an optional step, you could also leave it there if you want).
```
make install
mv ../dist /usr/local/opt/opencv-4.1.2
```

## Run feature extraction in Java
I assume here that you use [Maven](https://maven.apache.org), but this could be easily adapted to other build tools.

First of all, you have to install the compiled jar in your local repository:
```
mvn install:install-file -Dfile=/usr/local/opt/opencv-4.1.2/share/java/opencv4/opencv-412.jar \
-DgroupId=org -DartifactId=opencv -Dversion=4.1.2 -Dpackaging=jar
```

Then, create a Maven project as usual and add the following dependency to your `pom.xml`:
```
<dependency>
    <groupId>org</groupId>
    <artifactId>opencv</artifactId>
    <version>4.1.2</version>
</dependency>
```

Now, create the following class:
{% highlight java %}
import org.opencv.core.KeyPoint;
import org.opencv.core.Mat;
import org.opencv.core.MatOfKeyPoint;
import org.opencv.imgcodecs.Imgcodecs;
import org.opencv.xfeatures2d.SIFT;

public class OpenCVFeatureExtraction {
  public static void main(String[] args) {
    System.loadLibrary(org.opencv.core.Core.NATIVE_LIBRARY_NAME);

    SIFT sift = SIFT.create();
    String imgPath = "some_image.png";
    Mat img1 = Imgcodecs.imread(imgPath, Imgcodecs.IMREAD_GRAYSCALE);

    MatOfKeyPoint kpts1 = new MatOfKeyPoint();
    Mat desc1 = new Mat();

    sift.detectAndCompute(img1, new Mat(), kpts1, desc1);

    System.out.println("Found n. keypoints: " + kpts1.size());
    KeyPoint[] keyPoints = kpts1.toArray();
    for (int i = 0; i < keyPoints.length; i++) {
      KeyPoint kp = keyPoints[i];
            System.out.println(i + ": pt " + kp.pt
              + ", size = " + kp.size
              + ", angle = " + kp.angle
              + ", octave = " + kp.octave);
    }
  }
}
{% endhighlight %}

Finally, to run it you have to add to the JVM arguments the path of the OpenCV JNI library:
```
-Djava.library.path=/usr/local/opt/opencv-4.1.2/share/java/opencv4
```
Of course, change `imgPath` to point to an existing image file.

When you run the class, you will see the list of all _keypoints_ found by the algorithm in the image. A keypoint is actually the representation of an image feature.

To use SURF instead of SIFT, just do:
{% highlight java %}
SURF surf = SURF.create();
{% endhighlight %}
