
1、下载ffmpeg。我下载的是ffmpeg-3.3.3
下载地址：
[https://ffmpeg.org/download.html](https://ffmpeg.org/download.html)
这里特别提一句，如果你使用本文编译的话，请不要用ffmpeg-3.4，因为我在使用FFmpeg-3.4的编译的时候一直报libavutil/timer.h: fatal error  linux/perf_event.h : No
 such file 的错误，一开始我以为是MinGW的库没下载全，后面把所有的库都下载下载，编译依旧出现同样的错误，我在网上找了一下，发现也有人跟我一样出现同样的错误，但没有解决方法：
[FFmpeg 在CentOS7 下进行编译，一直报错](http://bbs.csdn.net/topics/392284004?page=1)
后来我在FFmpeg官网重新下载了ffmpeg-3.3.3版本，重新编译一次，直接就过了。似乎是MinGW工具链跟ffmpeg-3.4版本的依赖方式存在冲突(知道原因的话，请告诉我，在此先谢谢大家了)？

2、下载mingw
下载地址：
[https://sourceforge.net/projects/mingw/files/](https://sourceforge.net/projects/mingw/files/)
下载后运行，会自动下载安装器，安装器下载好之后会自动打开MinGW Installation Manager， 在安装管理器中选择Base Setup，选中需要安装的库，在安装时，最好选择网络比较稳定的时间，比如早上，因为如果msys下载失败，后面会无法编译通过的。
![安装内容选择](http://upload-images.jianshu.io/upload_images/2103804-2e560bab76c806d0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

3、下载[x264](http://www.videolan.org/developers/x264.html) 和 [FFmpeg](http://ffmpeg.org/download.html)。如果需要使用到H264解码功能，则需要集成x264的库。

4、编译x264。将以下内容保存为build_script.sh脚本文件，其中NDK表示你的路径，由于本人的NDK是从Android Studio里面下载的，因此路径默认在SDK下的ndk-bundle目录中，你也可以使用你自己下载的NDK，不过建议升级到最新的NDK 人4b版本，另外一点就是，TOOLCHAIN 路径需要你对一下，如果你用的版本不是NDK r14b，有可能使用的工具链版本不太一样，所以这里请确认toolchains下的arm-linux-androideabi版本是否对得上。在编译之前，先把x264的库解压到ffmpeg中，并改名为libx264：
```
#!/bin/bash
NDK=D:/Android/sdk/ndk-bundle
PLATFORM=$NDK/platforms/android-19/arch-arm
TOOLCHAIN=$NDK/toolchains/arm-linux-androideabi-4.9/prebuilt/windows-x86_64
PREFIX=./android/arm

EXTRA_CFLAGS="-march=armv7-a -mfloat-abi=softfp -mfpu=neon -D__ARM_ARCH_7__ -D__ARM_ARCH_7A__"

function build_one
{
./configure \
--prefix=$PREFIX \
--enable-static \
--enable-pic \
--enable-strip \
--host=arm-linux-androideabi \
--cross-prefix=$TOOLCHAIN/bin/arm-linux-androideabi- \
--sysroot=$PLATFORM \
--extra-cflags="-Os -fpic $EXTRA_CFLAGS" \
--extra-ldflags="" \

$ADDITIONAL_CONFIGURE_FLAG
make clean
make -j4
make install

}
build_one
```
打开之前下载的MinGW路径中的C:\MinGW\msys\1.0目录下的msys.dat，cd到你下载的x264解压的目录，然后执行刚才保存的脚本文件： ./build_script.sh， 编译成功后会在libx264目录下生成了一个叫做android的文件夹，并且将libx264.a复制到了该文件夹的arm/lib文件夹中，下面是输出的log信息：
![编译成功输出](http://upload-images.jianshu.io/upload_images/2103804-d2f86d1ecd7f8ddd.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

5、编译包含x264的FFmpeg库。进入ffmpeg解压的目录，将以下内容保存为build_script.sh脚本文件，将NDK和TOOLCHAIN改成你自己的路径：
```
#!/bin/bash

NDK=D:/Android/sdk/ndk-bundle
PLATFORM=$NDK/platforms/android-19/arch-arm
TOOLCHAIN=$NDK/toolchains/arm-linux-androideabi-4.9/prebuilt/windows-x86_64

basepath=$(cd `dirname $0`; pwd)
X264_INCLUDE=$basepath/libx264/android/arm/include

X264_LIB=$basepath/libx264/android/arm/lib

function build_one
{
    ./configure \
--prefix=$PREFIX \
--arch=arm \
--cpu=armv7-a \
--target-os=android \
--enable-cross-compile \
--cross-prefix=$TOOLCHAIN/bin/arm-linux-androideabi- \
--sysroot=$PLATFORM \
--extra-cflags="-I$X264_INCLUDE -I$PLATFORM/usr/include" \
--extra-ldflags="-L$X264_LIB" \
--cc=$TOOLCHAIN/bin/arm-linux-androideabi-gcc \
--nm=$TOOLCHAIN/bin/arm-linux-androideabi-nm \
--disable-shared \
--enable-static \
--enable-gpl \
--enable-version3 \
--enable-pthreads \
--enable-runtime-cpudetect \
--enable-small \
--disable-network \
--disable-vda \
--disable-iconv \
--enable-asm \
--enable-neon \
--enable-yasm \
--disable-encoders \
--enable-libx264 \
--enable-encoder=h263 \
--enable-encoder=libx264 \
--enable-encoder=aac \
--enable-encoder=mpeg4 \
--enable-encoder=mjpeg \
--enable-encoder=png \
--enable-encoder=gif \
--enable-encoder=bmp \
--disable-muxers \
--enable-muxer=h264 \
--enable-muxer=flv \
--enable-muxer=gif \
--enable-muxer=mp3 \
--enable-muxer=dts \
--enable-muxer=mp4 \
--enable-muxer=mov \
--enable-muxer=mpegts \
--disable-decoders \
--enable-decoder=aac \
--enable-decoder=aac_latm \
--enable-decoder=mp3 \
--enable-decoder=h263 \
--enable-decoder=h264 \
--enable-decoder=mpeg4 \
--enable-decoder=mjpeg \
--enable-decoder=gif \
--enable-decoder=png \
--enable-decoder=bmp \
--enable-decoder=yuv4 \
--disable-demuxers \
--enable-demuxer=image2 \
--enable-demuxer=h263 \
--enable-demuxer=h264 \
--enable-demuxer=flv \
--enable-demuxer=gif \
--enable-demuxer=aac \
--enable-demuxer=ogg \
--enable-demuxer=dts \
--enable-demuxer=mp3 \
--enable-demuxer=mov \
--enable-demuxer=m4v \
--enable-demuxer=concat \
--enable-demuxer=mpegts \
--enable-demuxer=mjpeg \
--enable-demuxer=mpegvideo \
--enable-demuxer=rawvideo \
--enable-demuxer=yuv4mpegpipe \
--disable-parsers \
--enable-parser=aac \
--enable-parser=ac3 \
--enable-parser=h264 \
--enable-parser=mjpeg \
--enable-parser=png \
--enable-parser=bmp\
--enable-parser=mpegvideo \
--enable-parser=mpegaudio \
--disable-protocols \
--enable-protocol=file \
--enable-protocol=hls \
--enable-protocol=concat \
--disable-filters \
--disable-filters \
--enable-filter=aresample \
--enable-filter=asetpts \
--enable-filter=setpts \
--enable-filter=ass \
--enable-filter=scale \
--enable-filter=concat \
--enable-filter=atempo \
--enable-filter=movie \
--enable-filter=overlay \
--enable-filter=rotate \
--enable-filter=transpose \
--enable-filter=hflip \
--enable-zlib \
--disable-outdevs \
--disable-doc \
--disable-ffplay \
--disable-ffmpeg \
--disable-ffserver \
--disable-debug \
--disable-ffprobe \
--disable-postproc \
--enable-avdevice \
--disable-symver \
--disable-stripping \
--extra-cflags="-Os -fpic $OPTIMIZE_CFLAGS" \
--extra-ldflags="$ADDI_LDFLAGS" \
$ADDITIONAL_CONFIGURE_FLAG


    make clean
    make -j8
    make install


$TOOLCHAIN/bin/arm-linux-androideabi-ld \
-rpath-link=$PLATFORM/usr/lib \
-L$PLATFORM/usr/lib \
-L$PREFIX/lib \
-L$X264_LIB \
-soname libffmpeg.so -shared -nostdlib -Bsymbolic --whole-archive --no-undefined -o \
$PREFIX/libffmpeg.so \
libavcodec/libavcodec.a \
libavfilter/libavfilter.a \
libswresample/libswresample.a \
libavformat/libavformat.a \
libavutil/libavutil.a \
libswscale/libswscale.a \
libavdevice/libavdevice.a \
libx264/libx264.a \
-lc -lm -lz -ldl -llog --dynamic-linker=/system/bin/linker \
$TOOLCHAIN/lib/gcc/arm-linux-androideabi/4.9.x/libgcc.a
}
# arm v7vfp
CPU=arm-v7a
OPTIMIZE_CFLAGS="-mfloat-abi=softfp -mfpu=neon -marm -march=armv7-a "
ADDI_CFLAGS="-marm"
PREFIX=./android/$CPU
build_one
```
以上脚本包含了x264的路径，并且对ffmpeg做了相应的裁剪。在编译完成后，我把多个.a静态库文件合并到了一个so当中。这里编译包含x264的ffmpeg库，则需要指定x264的静态库路径，也就是libx264文件夹，最后合并的x264静态库的路径是libx264/libx264.a。

在FFmpeg源码目录下，打开conigure文件，找到以下几行：
```
SLIBNAME_WITH_MAJOR='$(SLIBNAME).$(LIBMAJOR)'  
LIB_INSTALL_EXTRA_CMD='$$(RANLIB)"$(LIBDIR)/$(LIBNAME)"'  
SLIB_INSTALL_NAME='$(SLIBNAME_WITH_VERSION)'  
SLIB_INSTALL_LINKS='$(SLIBNAME_WITH_MAJOR)$(SLIBNAME)'
```
将其替换成：
```
SLIBNAME_WITH_MAJOR='$(SLIBPREF)$(FULLNAME)-$(LIBMAJOR)$(SLIBSUF)'  
LIB_INSTALL_EXTRA_CMD='$$(RANLIB)"$(LIBDIR)/$(LIBNAME)"'  
SLIB_INSTALL_NAME='$(SLIBNAME_WITH_MAJOR)'  
SLIB_INSTALL_LINKS='$(SLIBNAME)'
```
这是因为Android平台下，不能识别FFmpeg编译出来的so，比如 “libavcodec.so.5.100.1”，只能识别 “libavcodec.so”。如果你是将静态库打包进去的话，比如"libavcodec.a"，则可以不考虑。

执行脚本，如无意外，编译成功后，将会在ffmpeg目录下面生成一个android目录，点击去可以看到生成了相应的libffmpeg.so，至此，我们把多个静态库打包成了单个so文件，如下图所示：
![生成的目录情况](http://upload-images.jianshu.io/upload_images/2103804-5b6679bcc2a62244.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


更多关于FFmpeg命令配置问题，可以参考这篇文章：
 [ffmpeg ./configure参数说明](http://www.cnblogs.com/azraelly/archive/2012/12/31/2840541.html)

6、Android Studio 里面集成x264 和 FFmpeg
新建一个包含C++ Project的工程，如下图所示：
![新建工程](http://upload-images.jianshu.io/upload_images/2103804-fa495b60e09a1351.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
在main目录下新增一个cpp目录，然后在cpp目录下新增include目录，把ffmpeg的头文件和x264的头文件复制到该目录下，将生成的libffmpeg.so文件复制到工程的app/libs/armeabi-v7a/ 目录，如下图所示：
![复制文件](http://upload-images.jianshu.io/upload_images/2103804-9654d78eff9f3973.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

7、使用Cmake 配置FFmpeg
由于Android Studio 2.2以后集成了Cmake，本人也更喜欢用Cmake，习惯使用ndk-build的同学请自行查找资料，使用ndk-build的资料非常多，这里就不介绍了。首先在build.gradle下配置前面复制到armeabi-v7a目录的libffmpeg.so文件：
```
defaultConfig {
        applicationId "com.cgfay.ffmpegsample"
        minSdkVersion 21
        targetSdkVersion 26
        versionCode 1
        versionName "1.0"
        testInstrumentationRunner "android.support.test.runner.AndroidJUnitRunner"
        externalNativeBuild {
            cmake {
                cppFlags "-std=c++11"
            }
            ndk {
                abiFilters "armeabi-v7a"
            }
        }
    }
    buildTypes {
        release {
            minifyEnabled false
            proguardFiles getDefaultProguardFile('proguard-android.txt'), 'proguard-rules.pro'
        }
    }

    sourceSets.main {
        jniLibs.srcDirs = ['libs']
        jni.srcDirs = []
    }

    externalNativeBuild {
        cmake {
            path "CMakeLists.txt"
        }
    }
```
这样，so库文件就加载进来了，接下来我们需要我们需要配置CMakeLists.txt。
CMakeLists.txt的配置如下：
```
# 设置cmake最低版本
cmake_minimum_required(VERSION 3.4.1)

# 设置路径
set(distribution_DIR ${CMAKE_SOURCE_DIR}/../../../../libs)

# 加载头文件
include_directories(src/main/cpp/include)

# 加载ffmpeg库
add_library( ffmpeg
             SHARED
             IMPORTED )
set_target_properties( ffmpeg
                       PROPERTIES IMPORTED_LOCATION
                       ../../../../libs/armeabi-v7a/libffmpeg.so )

# 添加自身的jni库
add_library( native-lib

             SHARED

             src/main/cpp/native-lib.cpp )

# 查找Android存在的库
find_library( log-lib

              log )

# 链接库文件
target_link_libraries(
                       native-lib

                       # ffmpeg库
                       ffmpeg

                       ${log-lib} )
```
Cmake的详细配置过程可参考以下这篇文章：
[android studio cmake 配置.a连接库](http://blog.csdn.net/YoYo_Newbie/article/details/74427938)

接下来用Gradle Sync同步一下，之后我们就可以开始编写JNI调用了。

我们在MainActivity里面添加一个stringFromFFmpeg的Native()，如下：
```
package com.cgfay.ffmpegsample;

import android.support.v7.app.AppCompatActivity;
import android.os.Bundle;
import android.widget.TextView;

public class MainActivity extends AppCompatActivity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        // Example of a call to a native method
        TextView tv = (TextView) findViewById(R.id.sample_text);
        tv.setText(stringFromFFmpeg());
    }

    public native String stringFromFFmpeg();

    static {
        System.loadLibrary("native-lib");
    }
}
```

这里加载了库 native-lib，也就是我们前面CMakeLists中target_link_libraries 链接输出的库，然后我们在native-lib.cpp文件中编写stringFromFFmpeg方法实体，如下：
```
#include <jni.h>
#include <string>


extern "C" {

#include <libavcodec/avcodec.h>

JNIEXPORT jstring
JNICALL
Java_com_cgfay_ffmpegsample_MainActivity_stringFromFFmpeg(
        JNIEnv *env,
        jobject /* this */) {
    char info[10000] = { 0 };
    sprintf(info, "%s\n", avcodec_configuration());
    return env->NewStringUTF(info);
}

}
```
这里要用extern "C" 将头文件和方法包裹起来，因为FFmpeg是一个C语言库，而我们使用的则是Cmake +CPP编写代码。至此，我们就把FFmpeg集成到工程里面了，编译运行就可以看到以下界面啦：
![运行界面](http://upload-images.jianshu.io/upload_images/2103804-3425d1352bbecafc.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
我们将Demo编译成APK，然后解压可以看到，经过之前的步骤，打包后的libffmpeg.so包只有6.63MB，也不算太大，应该来说满足集成需求的，如下图所示：
![libffmpeg.so的大小](http://upload-images.jianshu.io/upload_images/2103804-458729331b1818a0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

Demo地址：[FFmpegSample](https://github.com/CainKernel/FFmpegSample)

个人建议，第一次编译使用FFmpeg的同学，最好自己手动操作一遍，纸上得到终觉浅，我在编译的时候也踩了很多坑。操作过一次之后，后面就方便了许多。
