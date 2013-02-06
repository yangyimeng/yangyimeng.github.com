---
layout: post
title: "rtslam install"
description: ""
category: "linux machine"
sumarry: |
  由于最近要开始研究机器人的室内定位的问题，而采用的算法是slam，而目前，有老外已经有了一套开源的项目了，其采用的是EKF——SLAM的解决方案，于是就先将这套开源项目拿来分析一下。而本文的目的在于说明运行这套开源项目的过程。
tags: []
---
{% include JB/setup %}

####摘要

由于最近要开始研究机器人的室内定位的问题，而采用的算法是slam，而目前，有老外已经有了一套开源的项目了，其采用的是EKF——SLAM的解决方案，于是就先将这套开源项目拿来分析一下。而本文的目的在于说明运行这套开源项目的过程。

####目录

1. [Dependencies](#a1)
2. [gdhe](#a2)
3. [Jarfar Installation](#a3)



<a name="a1"> </a>

####Dependencies

###Standard development tools

由于笔者使用的是ubuntu系统，因此，下面的几个依赖都是比较简单的，直接利用apt-get就可以简单的装好。

##gcc-c++

##make

##cmake

##git


###External libraries

##boost >= 1.40

关于boost的话，就不是简单的就可以搞定，需要一些步骤。

<pre class="brush: js;">
wget http://ncu.dl.sourceforge.net/project/boost/boost/1.52.0/boost_1_52_0.tar.bz2
tar -xvf boost_1_52_0.tar.bz2
cd boost_1_52_0
chmod u+x bootstrap.sh
./bootstrap.sh
./b2 install
</pre>

##boost-sandbox/numeric-bindings

<pre class="brush: js;">
svn co http://svn.boost.org/svn/boost/sandbox/numeric_bindings boost-sandbox/
</pre>

boost-sandbox的部分不需要进行编译或者其他的动作，只需要下载后，就可以了。

##lapack

<pre class="brush: js;">
wget http://www.netlib.org/lapack/lapack-3.3.1.tgz
tar -xvf lapack-3.3.1.tgz
cd lapack-3.3.1
cp make.inc.example make.inc
make
make install
</pre>

##opencv &gt;=2.1 &lt;2.4

安装过程

<pre class="brush: js;">
sudo apt-get install build-essential libgtk2.0-dev libavcodec-dev libavformat-dev libjpeg62-dev libtiff4-dev cmake libswscale-dev libjasper-dev

wget http://sourceforge.net/projects/opencvlibrary/files/opencv-unix/2.2/OpenCV-2.2.0.tar.bz2

tar xfv OpenCV-2.2.0.tar.bz2
rm OpenCV-2.2.0.tar.bz2
cd OpenCV-2.2.0
mkdir opencv.build
cd opencv.build


cmake ..
make
sudo make install

</pre>

配置库连接，需要配置如下文件
<pre class="brush: js;">
sudo gedit /etc/ld.so.conf.d/opencv.conf
</pre>
并且添加如下内容
<pre class="brush: js;">
/usr/local/lib
</pre>

然后运行

<pre class="brush: js;">
sudo ldconfig
</pre>

最后，编辑如下文件

<pre class="brush: js;">
sudo gedit /etc/bash.bashrc
</pre>

并且添加如下的内容

<pre class="brush: js;">
PKG_CONFIG_PATH=$PKG_CONFIG_PATH:/usr/local/lib/pkgconfig
export PKG_CONFIG_PATH
</pre>


##qt4 (optional, for 2D display)

这个的安装也是比较的简单，直接的利用apt-get的包管理器进行安装即可。

<pre class="brush: js;">
sudo apt-get install libqt4-dev libqt4-debug libqt4-gui libqt4-sql qt4-dev-tools qt4-doc qt4-designer qt4-qtconfig
</pre>


###LAAS libraries

##[gdhe](#a2)

##viam (optional, for live use firewire cameras)

<pre class="brush: js;">
wget http://distfiles.openrobots.org/viam-libs/viam-libs-1.10.tar.gz
tar -xvf viam-libs-1.10.tar.gz
cd viam-libs-1.10
./configure
make
make install
</pre>

viam的官网在[此](http://homepages.laas.fr/mallet/soft/image/viam-libs)

##[MTI driver](http://trac.laas.fr/openrobots/robotpkg/hardware/MTI/index.html) (optional, for live use of XSens MTi IMU)

<pre class="brush: js;">
wget ftp://ftp.openrobots.org/pub/openrobots/MTI-0.5.tar.gz
tar -xvf MTI-0.5.tar.gz
cd MTI-0.5
./configure
make
make install
</pre>

<a name="a2"> </a>

####gdhe

###Requirements

##GNU make

##mkdep

##libjpeg

##libXpm

##OpenGL and GLU

<pre class="brush: js;">
sudo apt-get install libglut3 libglut3-dev freeglut3
</pre>

##[Tcl and Tk](http://tcl.tk/) &gt;= 8.4

在这里，需要注意的是，需要安装版本8.5，否则若是安装8.4的话，则会报出tk.tcl不可用。应该是版本兼容问题。

<pre class="brush: js;">
wget http://prdownloads.sourceforge.net/tcl/tcl8.5.12-src.tar.gz
tar -xvf tcl8.5.12-src.tar.gz
cd tcl8.5.12-src
cd unix
./configure
make
make install
wget http://prdownloads.sourceforge.net/tcl/tk8.5.12-src.tar.gz
tar -xvf tk8.5.12-src.tar.gz
cd tk8.5.12
cd unix
./configure
make
make install
</pre>

###Download

先下载[gdhe-3.8.2.tar.gz](http://distfiles.openrobots.org/gdhe/gdhe-3.8.2.tar.gz)

<pre class="brush: js;">
wget http://distfiles.openrobots.org/gdhe/gdhe-3.8.2.tar.gz
tar -xvf gdhe-3.8.2.tar.gz
cd gdhe-3.8.2
./configure
make
make install
</pre>

<a name="a3"> </a>

####Jarfar Installation

###Download

<pre class="brush: js;">
git clone git://trac.laas.fr/git/robots/jafar/jafar.git
</pre>

###Configure the build system

<pre class="brush: js;">
mkdir $JAFAR_DIR/build
cd $JAFAR_DIR/build
cmake ../ -DENABLE_TCL=OFF -DENABLE_RUBY=OFF -DCMAKE_BUILD_TYPE=release \
-DBOOST_ROOT=/home/xxx/package/boost_1_51_0 \
-DPATH_TO_BOOST_SANDBOX=/home/xxx/package/boost-sandbox \
-DOpenCV_ROOT_DIR=/home/xxx/package/OpenCV-2.2.0
</pre>

###RT-SLAM installation

<pre class="brush: js;">
cd $JAFAR_DIR
bin/jafar_checkout rtslam
bin/jafar_checkout qdisplay
bin/jafar_checkout gdhe
cd build
make
</pre>






