在[windows环境下编译ffmpeg打包成单个so并使用Cmake集成到Android工程中](https://www.jianshu.com/p/ed2266abe28b) 文章中，我们介绍了如何在Windows环境下编译打包成libffmpeg.so的过程。由于工程的需要，我们需要使用fdk-aac 作为aac的编解码库。但是本人尝试之后发现，在Windows环境下fdk-aac的编译比较麻烦，各种出错。因此，这里将会介绍如何在Linux环境下编译 FFmpeg + x264 + fdk-aac，并打包成单个libffmpeg.so的过程。

### 编译环境介绍
* Linux: Ubuntu 16.04
* Android NDK: android-ndk-r13b
* fdk-aac:  fdk-aac-0.1.5.tar.gz
* x264：官网最新版
* FFmpeg: FFmpeg-4.0.tar.bz2

### 编译预处理
安装fdk-aac编译所需要的automake工具：
```
$ sudo apt-get install automake
```
安装编译ffmpeg x86/x86_64版本编译所需要的yasm工具：
```
sudo apt-get install yasm
```
### 编译共享库体积裁剪
FFmpeg4.0 全部内容打包进去，大约在19M左右。共享库非常大，对于移动端来说，很多东西都不必要。因此，为了减少体积，我们需要通过配置configure对FFmpeg共享库进行裁剪处理。

到ffmpeg解压目录下执行以下命令查看支持的配置选项：
```
$ ./configure --help
```
对于移动端来说，一般情况下都会配置以下编译选项：
* --enable-small : 使用编译速度来换取编译包大小
* --disable-runtime-cpudetect : 禁止运行时检测CPU性能，可以编译出较小的包
* --disable-doc : 禁止编译文档，可以避免将文档编译入包中
* --disable-htmlpages : 禁止编译html文档
* --disable-manpages : 禁止编译man手册
* --disable-podpages : 禁止编译pod手册
* --disable-txtpages ： 禁止编译txt文档 
* --disable-programs : 禁止编译命令行工具
* --disable-ffmpeg : 禁止编译 ffmpeg 工具
* --disable-ffplay : 禁止编译 ffplay 工具
* --disable-ffprobe : 禁止编译ffprobe 工具
* --disable-ffserver : 禁止编译 ffserver 工具，FFmpeg4.0中该选项已经没有了。
* --disable-devices : 禁止所有设备的编译
* --disable-indevs : 禁止所有输入设备的编译
* --disable-outdevs : 禁止所有输出设备的编译

前面是禁止一些基础的工具，接下来需要对各个模块进行配置：
#### libavdevice
主要负责与硬件设备的交互。对于移动端来说，该模块并没有实际的用处，一般会禁止，禁止选项为：--disable-avdevice
#### libavcodec
编解码模块，不要禁止。
#### libavformat
解封装和封装模块，不要禁用。
#### libswresample
音频重采样模块，一般不禁止
#### libswscale
视频转码模块，一般不禁止。
#### libpostproc
音视频后期处理模块。对于移动端来说，这个基本没用，一般会禁止，禁止选项为：--disable-postproc
#### libavfilter
音视频滤镜模块，包括剪切、裁剪、水印、音视频分离合成、变速、重采样等处理。不需要可以通过 --disable-avfilter 禁止。如果要支持命令行，那就不能禁止该模块了。
#### libavresample
音视频封装编解码格式预设，默认不编译。

### 减少不必要解析器的编译
* --disable-parsers : 禁止所有解析器的编译
* --disable-parsers : 禁止特定解析器的编译
* --enable-parser=NAME : 特定的解析器编译
查看ffmpeg支持解析器的名称，可以在终端执行以下命令获取：
```
$ ./configure --list-parsers
```
FFmpeg4.0中支持的解析器如下：
```
aac			   dvdsub		      png
aac_latm		   flac			      pnm
ac3			   g729			      rv30
adx			   gsm			      rv40
bmp			   h261			      sbc
cavsvideo		   h263			      sipr
cook			   h264			      tak
dca			   hevc			      vc1
dirac			   mjpeg		      vorbis
dnxhd			   mlp			      vp3
dpx			   mpeg4video		      vp8
dvaudio			   mpegaudio		      vp9
dvbsub			   mpegvideo		      xma
dvd_nav			   opus
```

#### 减少不必要的二进制流过滤器的编译
* --disable-bsfs : 禁止所有二进制流过滤器的编译
* --disable-bsf=NAME : 禁止特定二进制流过滤器的编译
* --enable-bsf=NAME : 允许特定二进制流过滤器的编译，配合 --disable-bsfs 可以实现支持某些特定的二进制流过滤器。
查看FFmpeg中支持二进制流过滤器，可以通过以下命令来查看
```
$ ./configure --list-bsfs
```
FFmpeg4.0中支持的二进制流过滤器如下：
```
aac_adtstoasc		   hapqa_extract	      mpeg4_unpack_bframes
chomp			   hevc_metadata	      noise
dca_core		   hevc_mp4toannexb	      null
dump_extradata		   imx_dump_header	      remove_extradata
eac3_core		   mjpeg2jpeg		      text2movsub
extract_extradata	   mjpega_dump_header	      trace_headers
filter_units		   mov2textsub		      vp9_raw_reorder
h264_metadata		   mp3_header_decompress      vp9_superframe
h264_mp4toannexb	   mpeg2_metadata	      vp9_superframe_split
h264_redundant_pps
```
#### 减少不必要的协议编译
FFmpeg中对读取数据输出数据制定了一套协议。我们可以通过指定支持的输入输出方式来避免不必要的编译。
* --disable-protocols : 禁止所有输入输出方式的编译
* --disable-protocol : 禁止特定输入输出方式的编译
* --enable-protocol : 允许特定输入输出方式的编译，搭配 --disable-protocols 可以单纯指定某些输入输出方式。
可以通过以下命令来查看支持的协议：
```
$ ./configure --list-protocols
```
FFmpeg4.0中支持的协议如下：
```
async			   icecast		      rtmpe
bluray			   librtmp		      rtmps
cache			   librtmpe		      rtmpt
concat			   librtmps		      rtmpte
crypto			   librtmpt		      rtmpts
data			   librtmpte		      rtp
ffrtmpcrypt		   libsmbclient		      sctp
ffrtmphttp		   libsrt		      srtp
file			   libssh		      subfile
ftp			   md5			      tcp
gopher			   mmsh			      tee
hls			   mmst			      tls
http			   pipe			      udp
httpproxy		   prompeg		      udplite
https			   rtmp			      unix
```

#### 减少不必要的组件编译
FFmpeg 处理音视频时，一般涉及几个流程：
解封装、解码、编码、封装
这几个流程分别对应以下几个组件：
分离器(demuxer)、解码器(decoder)、编码器(encoder)、复用器(muxer)。

##### 1、分离器(demuxer) 编译选项
* --disable-demuxers : 禁止所有分离器的编译。
* --disable-demuxer=NAME : 禁止特定分离器的编译
* --enable-demuxer=NAME : 允许特定分离器的编译

我们可以通过以下命令行来查看FFmpeg中支持的分离器：
```
$ ./configure --list-demuxers
```
由于支持的分离器比较多，这里就不贴出来了，自行查看。

##### 2、解码器(decoder)编译选项
* --disable-decoders : 禁止所有解码器的编译
* --disable-decoder=NAME : 禁止特定解码器的编译
* --enable-decoder=NAME : 允许特定解码器的编译
我们可以通过以下命令查看FFmpeg中支持的解码器名称：
```
$ ./configure --list-decoders
```
由于支持的解码器比较多，这里就不贴出来了，自行查看。

##### 3、编码器(encoder)编译选项
* --disable-encoder : 禁止所有编码器的编译
* --disable-encoder=NAME : 禁止特定编码器的编译
* --enable-encoder=NAME : 允许特定编码器的编译
我们可以通过以下命令查看FFmpeg中支持的编码器名称：
```
$ ./configure --list-encoders
```
由于支持的编码器比较多，这里就不贴出来了，自行查看。

##### 4、复用器(muxer)编译选项
* --disable-muxers : 禁止所有复用器的编译
* --disable-muxer=NAME : 禁止特定复用器的编译
* --enable-muxer=NAME : 允许特定复用器的编译
我们可以通过以下命令查看FFmpeg中支持的复用器名称：
```
$ ./configure --list-muxers
```
由于支持的复用器比较多，这里就不贴出来了，自行查看。

#### 减少不必要的过滤器编译
如果想要在移动端支持命令行工具。可以通过-enable-avfilter来开启。由于filters有很多，如果使用有限的过滤器，则可以通过 --disable-filters 禁止所有滤镜，然后使用 --enable-filter=NAME 逐个添加所需要的滤镜。支持的过滤器名称可以通过终端执行以下命令查看：
```
$./configure --list-filters
```
由于支持的过滤器比较多，这里就不贴做出来了，自行查看。

### Android 环境支持硬解码编译选项
FFmpeg3.1以后开始支持Android环境(API >= 21)的MediaCodec硬解码，但不支持硬编码。硬解码的实现代码可以到libavcodec目录下，查看 mediacodec.c、mediacodec_surface.c、mediacodec_wrapper.c、mediacodec_common.c、mediacoder_sw_buffer.c等文件，里面仅仅实现了解码的逻辑。FFmpeg 在这几个文件中使用的是NDK中的AMediaCodec，而NDK的AMediaCodec只支持Android 5.0以及以上版本。对于要支持Android 5.0以下版本的硬解码，只能自行实现自定义的解码器。关于如何自定义解码器，这里不做介绍，自行找资料。

其中，FFmpeg4.0中支持Android硬解码的格式有：
h264_mediacodec、hevc_mediacodec、mpeg2_mediacodec、mpeg4_mediacodec、vp8_mediacodec、vp9_mediacodec

#### 添加支持MediaCodec的编译选项
* --enable-jni : 添加 jni支持
* --enable-mediacodec : 允许mediacodec
* --enable-decoder=h264_mediacodec : 允许h264_mediacodec 解码器
* --enable-hwaccel=h264_mediacodec : 允许h264_mediacodec硬解码

* --target-os=android : 使用MediaCodec硬解码时，必须将--target-os设置为android，否则会提示以下出错信息：
ERROR：jni not found

###### 为ffmpeg配置java虚拟机
编译得到 libffmpeg.so之后，我们需要在JNI_OnLoad方法中调用av_jni_set_java_vm方法绑定JavaVM，该方法在 libavcodec/jni.h中。

###### 在FFmpeg中使用MediaCodec
MediaCodec要想使用，需要指定前面的硬解码格式名称。如果获取解码器失败，则再通过id来获取软解码器，实现代码如下：
```
const char *forcedCodecName = "h264_mediacodec";
if (forcedCodecName) {
            codec = avcodec_find_decoder_by_name(forcedCodecName);
        }

        // 如果没有找到指定的解码器，则查找默认的解码器
        if (!codec) {
            if (forcedCodecName) {
                av_log(NULL, AV_LOG_WARNING,
                       "No codec could be found with name '%s'\n", forcedCodecName);
            }
            codec = avcodec_find_decoder(avctx->codec_id);
        }
```

### 编译脚本处理
前面介绍了关于ffmpeg 的编译配置，接下来我们编写编译脚本。
首先我们构造这样一个目录：
--> build --> fdk-aac/
              --> ffmpeg/
              --> x264/
--> tools --> build_fdk_aac_armeabi-v7a.sh
              --> build_ffmpeg_armeabi-v7a.sh
              --> build_x264_armeabi-v7a.sh
--> build_armeabi-v7a.sh
--> fdk-aac.tar.gz
--> ffmpeg.tar.gz
--> x264.tar.gz

### 顶层编译脚本：
build_armeabi-v7a.sh 内容如下：
```
#!/bin/sh
# 根目录
export ROOT_SOURCE=$(cd `dirname $0`; pwd)

#判断系统类型
SYSTEM=$(uname -s)

# Linux环境
if [ "${SYSTEM}" = "Linux" ]; then
  export NDK=/opt/Android/android-ndk-r13b
  export TOOLCHAIN=$NDK/toolchains/arm-linux-androideabi-4.9/prebuilt/linux-x86_64
fi

# MacOS环境
if [ "${SYSTEM}" = "Darwin" ]; then
  export NDK=/opt/Android/android-ndk-r13b
  export TOOLCHAIN=$NDK/toolchains/arm-linux-androideabi-4.9/prebuilt/darwin-x86_64
fi

# 指定编译的API
export ANDROID_API=android-21
export PLATFORM=$NDK/platforms/${ANDROID_API}/arch-arm

# 编译目录是否存在
if [ ! -d "build" ]; then
  mkdir ${ROOT_SOURCE}/build/
fi

# 源码是否存在，解压出来
if [ ! -d "build/ffmpeg" ]; then
  tar -zxvf ffmpeg.tar.gz -C build/
fi

if [ ! -d "build/x264" ]; then
  tar -zxvf x264.tar.gz -C build/
fi

if [ ! -d "build/fdk-aac" ]; then
  tar -zxvf fdk-aac.tar.gz -C build/
fi

# 到工具目录执行编译
# 编译x264
cd ${ROOT_SOURCE}/tools
if [ ! -x "build_x264_armeabi-v7a.sh" ]; then
   echo "can not find x264 build script"
else 
   ./build_x264_armeabi-v7a.sh
fi

# 编译fdk-aac
cd ${ROOT_SOURCE}/tools
if [! -x "build_fdk-aac_armeabi-v7a.sh" ]; then
  echo "can not find fdk-aac build script"
else 
  ./build_fdk-aac_armeabi-v7a.sh
fi

# 编译ffmpeg
cd ${ROOT_SOURCE}/tools
if [ ! -x "build_ffmpeg_armeabi-v7a.sh" ]; then
  echo "can not find ffmpeg build script"
else
  ./build_ffmpeg_armeabi-v7a.sh
fi
```
顶层编译脚本主要用于检测系统内环境，配置对应的NDK目录，支持Linux 和MacOS、指定NDK、编译工具链、API 和Platform路径、解压文件到编译目录、分别执行x264、fdk-aac 和 ffmpeg的编译脚本。

### 添加libx264的的支持
下载x264并解压，armeabi-v7a的编译脚本如下：
```
#!/bin/sh

# x264 源码目录
X264_SOURCE=${ROOT_SOURCE}/build/x264
# 输出路径
PREFIX=${X264_SOURCE}/android/armeabi-v7a

# 配置和编译
cd ${X264_SOURCE}

./configure \
--prefix=${PREFIX} \
--disable-shared \
--enable-static \
--enable-pic \
--enable-strip \
--host=arm-linux-androideabi \
--cross-prefix=${TOOLCHAIN}/bin/arm-linux-androideabi- \
--sysroot=${PLATFORM} \
--extra-cflags="-Os -fpic" \
--extra-ldflags="" \
${ADDITIONAL_CONFIGURE_FLAG}

make clean
make -j4
make install
```

### 添加 fdk-aac 的支持
fdk-aac 的 armeabi-v7a编译脚本如下：
```
#!/bin/sh

#Android目录
CROSS_COMPILE=${TOOLCHAIN}/bin/arm-linux-androideabi-

ARM_INC=${PLATFORM}/usr/include


LDFLAGS=" -nostdlib -Bdynamic -Wl,--whole-archive -Wl,--no-undefined -Wl,-z,noexecstack  -Wl,-z,nocopyreloc -Wl,-soname,/system/lib/libz.so -Wl,-rpath-link=$ARM_LIB,-dynamic-linker=/system/bin/linker -L$NDK/sources/cxx-stl/gnu-libstdc++/libs/armeabi -L${TOOLCHAIN}/arm-linux-androideabi/lib -L${PLATFORM}/usr/lib  -lc -lgcc -lm -ldl "

FDK_FLAGS="--enable-static --host=arm-linux-androideabi --target=android --disable-asm --disable-shared"

export CXX="${CROSS_COMPILE}g++ --sysroot=${PLATFORM}"

export LDFLAGS="$LDFLAGS"

export CC="${CROSS_COMPILE}gcc --sysroot=${PLATFORM}"

# fdk-aac 源码目录
FDK_AAC_SOURCE=${ROOT_SOURCE}/build/fdk-aac

# 输出路径
FDK_PREFIX=${FDK_AAC_SOURCE}/android/armeabi-v7a

cd ${FDK_AAC_SOURCE}
# 创建编译输出路径
if [ ! -d ${FDK_PREFIX} ]; then
mkdir -p ${FDK_PREFIX}
fi

./configure ${FDK_FLAGS} \
--prefix=${FDK_PREFIX}

$ADDITIONAL_CONFIGURE_FLAG

make clean
make -j4
make install
```
