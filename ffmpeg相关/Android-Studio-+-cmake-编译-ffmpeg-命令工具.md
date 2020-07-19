在前一篇文章[windows环境下编译ffmpeg打包成单个so并使用Cmake集成到Android工程中](http://www.jianshu.com/p/ed2266abe28b) 我们说到了将ffmpeg 编译打包成单个so，并使用cmake 集成到android工程中，现在我们来说说，如何使用cmake 生成能够使用jni 调用 ffmpeg命令工具。首先，你得按照前一篇文章所说的生成了libffmpeg.so包，然后再执行接下来的步骤。

1、将ffmpeg的头文件复制到src/main/cpp/ffmpeg目录，将ffmpeg中的cmdutils.c、cmdutils.h、config.h、ffmpeg.h、ffmpeg.c、ffmpeg_filter.c、ffmpeg_opt.c复制到src/main/cpp/ffmpeg目录。

2、修改 ffmpeg.c 文件，将
```
int main(int argc, char **argv)
```
修改为：
```
int run(int argc, char **argv)
```
为了能够重复使用命令，我们需要修改ffmpeg的清理方法ffmpeg_cleanup，在方法的末尾将一些参数重置，如下所示：
```
    av_freep(&vstats_filename);
    vstats_filename = NULL;

    av_freep(&input_streams);
    input_streams = NULL;
    nb_input_streams = 0;

    av_freep(&input_files);
    input_files = NULL;
    nb_input_files = 0;

    av_freep(&output_streams);
    output_streams = NULL;
    nb_output_streams = 0;

    av_freep(&output_files);
    output_files = NULL;
    nb_input_files = 0;
```
将 sigterm_handler 方法中的
```
exit(123)
```
修改为
```
exit_program(123);
```

在ffmpeg.h 文件的末尾，添加声明
```
int run(int argc, char **argv)；
```
另外在头文件开头声明Android的Log方法：
```
#include <android/log.h>

#ifndef LOG_TAG
#define LOG_TAG "FFMPEG"
#endif

#define LOGV(...) __android_log_print(ANDROID_LOG_VERBOSE, LOG_TAG, __VA_ARGS__)
#define LOGD(...) __android_log_print(ANDROID_LOG_DEBUG ,  LOG_TAG, __VA_ARGS__)
#define LOGI(...) __android_log_print(ANDROID_LOG_INFO  ,  LOG_TAG, __VA_ARGS__)
#define LOGW(...) __android_log_print(ANDROID_LOG_WARN  ,  LOG_TAG, __VA_ARGS__)
#define LOGE(...) __android_log_print(ANDROID_LOG_ERROR  , LOG_TAG, __VA_ARGS__)
```
至此，ffmpeg.c/ffmpeg.h 主文件已经修改完成

3、修改 cmdutils.c 文件，添加头文件和声明：
```
#include <setjmp.h>
extern jmp_buf jmp_exit;
```
修改 exit_program() 方法，如下所示：
```
void exit_program(int ret)
{
    if (program_exit)
    {
        av_log(NULL, AV_LOG_INFO, "run program_exit.\n");
        program_exit(ret);
    }
    else
    {
        av_log(NULL, AV_LOG_INFO, "program_exit is null\n");
    }

    // 转换错误码11，因为ffmpeg命令执行成功返回是1，命令参数错误返回的错误码也是1
    if (ret == 1)
    {
        ret = 11;
    }

    av_log(NULL, AV_LOG_INFO, "exit_program code: %d\n", ret);
    longjmp(jmp_exit, ret);
    // exit(ret);
}
```

在cmdutils.h文件中添加以下声明：
```
#ifdef FFMPEG_RUN_LIB

#ifdef ANDROID

#include <android/log.h>
#ifndef LOG_TAG
#define LOG_TAG "FFMPEG"
#endif
#define XLOGV(...) __android_log_print(ANDROID_LOG_VERBOSE, LOG_TAG, __VA_ARGS__)
#define XLOGD(...) __android_log_print(ANDROID_LOG_DEBUG ,  LOG_TAG, __VA_ARGS__)
#define XLOGI(...) __android_log_print(ANDROID_LOG_INFO  ,  LOG_TAG, __VA_ARGS__)
#define XLOGW(...) __android_log_print(ANDROID_LOG_WARN  ,  LOG_TAG, __VA_ARGS__)
#define XLOGE(...) __android_log_print(ANDROID_LOG_ERROR  , LOG_TAG, __VA_ARGS__)

#else
#include <stdio.h>
#define XLOGV(format, ...) __android_log_print(ANDROID_LOG_VERBOSE, LOG_TAG ": " format "\n", ##__VA_ARGS__)
#define XLOGD(format, ...) __android_log_print(ANDROID_LOG_DEBUG ,  LOG_TAG ": " format "\n", ##__VA_ARGS__)
#define XLOGI(format, ...) __android_log_print(ANDROID_LOG_INFO  ,  LOG_TAG ": " format "\n", ##__VA_ARGS__)
#define XLOGW(format, ...) __android_log_print(ANDROID_LOG_WARN  ,  LOG_TAG ": " format "\n", ##__VA_ARGS__)
#define XLOGE(format, ...) __android_log_print(ANDROID_LOG_ERROR  , LOG_TAG ": " format "\n", ##__VA_ARGS__)

#endif // ANDROID
#endif // FFMPEG_RUN_LIB
```
到这里我们就已经把ffmpeg命令工具移植到了项目中，但是我们还不能使用。接下来，我们需要添加自己的jni调用方法。

4、在src/main/cpp目录下新建一个的ffmpeg_cmd的文件夹，新建ffmpeg_cmd.c/ffmpeg_cmd.h 、ffmpeg_cmd_wrapper.c/ffmpeg_cmd_wrapper.h文件，用来写我们的jni控制调用ffmpeg命令工具。
ffmpeg_cmd.h内容如下：
```
#ifndef CAINCAMERA_FFMPEG_CMD_H
#define CAINCAMERA_FFMPEG_CMD_H

int run_cmd(int argc, char** argv);

#endif //CAINCAMERA_FFMPEG_CMD_H
```

ffmpeg_cmd.c内容如下：
```
#include <setjmp.h>
#include <android/log.h>

#include "ffmpeg_cmd.h"
#include "ffmpeg.h"
#ifdef __cplusplus
extern "C" {
#endif

#define LOGD(...)  __android_log_print(ANDROID_LOG_DEBUG, "FFMPEG", __VA_ARGS__)

jmp_buf jmp_exit;

int run_cmd(int argc, char** argv)
{
    int res = 0;
    if(res = setjmp(jmp_exit))
    {
        LOGD("res=%d", res);
        return res;
    }

    res = run(argc, argv);
    LOGD("res_run=%d", res);
    return res;
}

#ifdef __cplusplus
}
#endif
```

ffmpeg_cmd_wrapper.h内容如下：
```
#ifndef CAINCAMERA_FFMPEG_CMD_WRAPPER_H
#define CAINCAMERA_FFMPEG_CMD_WRAPPER_H

#include"jni.h"

#ifdef __cplusplus
extern "C" {
#endif


JNIEXPORT jint
JNICALL Java_com_cgfay_caincamera_jni_FFmpegCmd_run
        (JNIEnv *env, jclass obj, jobjectArray commands);

#ifdef __cplusplus
}
#endif


#endif //CAINCAMERA_FFMPEG_CMD_WRAPPER_H
```
ffmpeg_cmd_wrapper.c内容如下：
```

#include "ffmpeg_cmd.h"
#include "ffmpeg_cmd_wrapper.h"
#include "jni.h"

#ifdef __cplusplus
extern "C" {
#endif

JNIEXPORT jint
JNICALL Java_com_cgfay_caincamera_jni_FFmpegCmd_run
        (JNIEnv *env, jclass obj, jobjectArray commands)
{
    int argc = (*env)->GetArrayLength(env, commands);
    char *argv[argc];
    jstring jstr[argc];

    int i = 0;;
    for (i = 0; i < argc; i++)
    {
        jstr[i] = (jstring)(*env)->GetObjectArrayElement(env, commands, i);
        argv[i] = (char *) (*env)->GetStringUTFChars(env, jstr[i], 0);
    }

    int status = run_cmd(argc, argv);

    for (i = 0; i < argc; ++i)
    {
        (*env)->ReleaseStringUTFChars(env, jstr[i], argv[i]);
    }

    return status;
}
#ifdef __cplusplus
}
#endif
```
其中，Java_com_cgfay_caincamera_jni_FFmpegCmd_run 跟java中的jni调用方法要对上，这里可以换成你自己的包路径和方法名。接下来我们java层新建一个FFmpegCmd.java文件。文件内容如下：
```
package com.cgfay.caincamera.jni;

import android.util.Log;

import com.cgfay.caincamera.utils.FileUtils;

import java.util.ArrayList;
import java.util.List;
import java.util.Locale;

/**
 * 用于管理FFmpeg命令工具
 * Created by cain.huang on 2017/12/12.
 */

public class FFmpegCmd {
    private static final String TAG = "FFmpegCmd";
    private static boolean VERBOSE = false;

    private static final int RUN_SUCCESS = 0;
    private static final int RUN_FAILED = 1;

    // 是否正在运行命令
    private static boolean mIsRunning = false;

    private static final String STR_DEBUG_PARAM = "-d";

    static {
        System.loadLibrary("ffmpeg");
        System.loadLibrary("ffmpeg_cmd");
    }


    private native static int run(String[] cmd);

    public interface OnCompletionListener {
        void onCompletion(boolean result);
    }

    private static int runSafely(String[] cmd) {
        int result = -1;

        long time = System.currentTimeMillis();
        try {
            result = run(cmd);
            if (VERBOSE) {
                Log.d(TAG, "time = " + (System.currentTimeMillis() - time));
            }
        } catch (Throwable throwable) {
            throwable.printStackTrace();
        }
        return result;
    }

    private static void runSync(ArrayList<String> cmds, final OnCompletionListener listener) {
        if (VERBOSE) {
            cmds.add(STR_DEBUG_PARAM);
        }

        final String[] commands = cmds.toArray(new String[cmds.size()]);
        Runnable runnable = new Runnable() {
            @Override
            public void run() {
                int result = runSafely(commands);
                callbackResult(result, listener);
            }
        };
        mIsRunning = true;
        new Thread(runnable).start();
    }

    private static void callbackResult(int result, OnCompletionListener listener) {
        if (VERBOSE) {
            Log.d(TAG, "result = " + result);
        }

        if (listener != null) {
            listener.onCompletion(result == 1);
        }

        mIsRunning = false;
    }

    /**
     * 音视频混合
     * @param srcVideo      视频路径
     * @param videoVolume   视频声音
     * @param srcAudio      音频路径
     * @param audioVolume   音频声音
     * @param desVideo      目标视频
     * @param callback      状态回调
     * @return
     */
    public static boolean AVMuxer(String srcVideo, float videoVolume,
                                  String srcAudio, float audioVolume, String desVideo,
                                  OnCompletionListener callback) {
        if (srcAudio == null || srcAudio.length() <= 0
                || desVideo == null || desVideo.length() <= 0) {
            return false;
        }

        ArrayList<String> cmds = new ArrayList<>();
        cmds.add("ffmpeg");
        cmds.add("-i");
        cmds.add(srcVideo);
        cmds.add("-i");
        cmds.add(srcAudio);

        cmds.add("-c:v");
        cmds.add("copy");
        cmds.add("-map");
        cmds.add("0:v:0");

        cmds.add("-strict");
        cmds.add("-2");

        if (videoVolume <= 0.001f) { // 使用audio声音
            cmds.add("-c:a");
            cmds.add("aac");

            cmds.add("-map");
            cmds.add("1:a:0");

            cmds.add("-shortest");

            if (audioVolume < 0.99 || audioVolume > 1.01) {
                cmds.add("-vol");
                cmds.add(String.valueOf((int)(audioVolume * 100)));
            }

        } else if (videoVolume > 0.001f && audioVolume > 0.001f) { // 混合音视频声音

            cmds.add("-filter_complex");
            cmds.add(String.format(
                    "[0:a]aformat=sample_fmts=fltp:sample_rates=48000:channel_layouts=stereo,volume=%f[a0]; " +
                    "[1:a]aformat=sample_fmts=fltp:sample_rates=48000:channel_layouts=stereo,volume=%f[a1];" +
                    "[a0][a1]amix=inputs=2:duration=first[aout]", videoVolume, audioVolume));

            cmds.add("-map");
            cmds.add("[aout]");

        } else {
            Log.w(TAG, String.format(Locale.getDefault(),
                    "Illigal volume : SrcVideo = %.2f, SrcAudio = %.2f",
                    videoVolume, audioVolume));
            if (callback != null) {
                callback.onCompletion(RUN_FAILED == 1);
            }
        }

        cmds.add("-f");
        cmds.add("mp4");
        cmds.add("-y");
        cmds.add("-movflags");
        cmds.add("faststart");
        cmds.add(desVideo);

        runSync(cmds, callback);

        return true;
    }

    /**
     * 设置播放速度
     * @param srcVideo
     * @param speed
     * @param desVideo
     * @param callback
     */
    public static void setPlaybackSpeed(String srcVideo, float speed,
                                        String desVideo, OnCompletionListener callback) {
        ArrayList<String> cmds = new ArrayList<>();
        cmds.add("ffmpeg");
        cmds.add("-i");
        cmds.add(srcVideo);

        cmds.add("-y");
        cmds.add("-filter_complex");
        cmds.add("[0:v]setpts=" + speed + "*PTS[v];[0:a]atempo=" + 1 / speed + "[a]");
        cmds.add("-map");
        cmds.add("[v]");
        cmds.add("-map");
        cmds.add("[a]");
        cmds.add(desVideo);

        runSync(cmds, callback);
    }


    /**
     * 剪切视频
     * @param srcVideo
     * @param desVideo
     * @param startTime
     * @param endTime
     * @return
     */
    public static boolean cutVideo(String srcVideo, String desVideo,
                                   float startTime, float endTime) {

        ArrayList<String> cmds = new ArrayList<>();
        cmds.add("ffmpeg");
        cmds.add("-i");
        cmds.add(srcVideo);

        cmds.add("-y");
        cmds.add("-ss");
        cmds.add("" + startTime);
        cmds.add("-t");
        cmds.add("" + endTime);
        cmds.add("-c");
        cmds.add("copy");
        cmds.add(desVideo);

        String[] commands = cmds.toArray(new String[cmds.size()]);

        int result = runSafely(commands);

        return (result == 1);
    }

    /**
     * 将图片转成视频
     * @param picPath   图片路径
     * @param duration  时间
     * @param desVideo  输出视频路径
     * @param callback  回调
     */
    public static void convertPictureToVideo(String picPath, float duration,
                                             String desVideo, OnCompletionListener callback) {

        ArrayList<String> cmds = new ArrayList<>();
        cmds.add("ffmpeg");
        cmds.add("-y");
        cmds.add("-loop");
        cmds.add("1");
        cmds.add("-f");
        cmds.add("image2");
        cmds.add("-i");
        cmds.add(picPath);

        cmds.add("-t");
        cmds.add(""+duration);
        cmds.add("-r");
        cmds.add("15");

        cmds.add(desVideo);

        runSync(cmds, callback);
    }

    /**
     * 添加Gif到视频
     * @param videoPath
     * @param gifPath
     * @param x
     * @param y
     * @param startTime
     * @param desVideo
     * @param callback
     */
    public static void addGifToVideo(String videoPath, String gifPath,
                                     float x, float y,
                                     float startTime, String desVideo,
                                     OnCompletionListener callback) {
        ArrayList<String> cmds = new ArrayList<>();
        cmds.add("ffmpeg");
        cmds.add("-y");
        cmds.add("-i");
        cmds.add(videoPath);
//        cmds.add("-ignore_loop");
//        cmds.add("0");
        cmds.add("-i");
        cmds.add(gifPath);

        cmds.add("-ss");
        cmds.add("" + startTime);

        cmds.add("-filter_complex");
        cmds.add("overlay=" + x + ":" + y);

        cmds.add(desVideo);

        runSync(cmds, callback);
    }

    /**
     * 旋转视频
     * @param srcVideo
     * @param desVideo
     * @param callback
     */
    public static void rotateVideo(String srcVideo, String desVideo,
                                   OnCompletionListener callback) {
        ArrayList<String> cmds = new ArrayList<>();
        cmds.add("ffmpeg");
        cmds.add("-i");
        cmds.add(srcVideo);

        cmds.add("-vf");
//        cmds.add("transpose=1:portrait");
        cmds.add("rotate=PI/2");
        cmds.add(desVideo);

        runSync(cmds, callback);
    }

    /**
     * 添加水印
     * @param srcVideo
     * @param waterMark
     * @param desVideo
     * @param callback
     */
    public static void addWaterMark(String srcVideo, String waterMark,
                                    String desVideo, OnCompletionListener callback) {

        ArrayList<String> cmds = new ArrayList<>();
        cmds.add("ffmpeg");
        cmds.add("-i");
        cmds.add(srcVideo);
        cmds.add("-i");
        cmds.add(waterMark);

        cmds.add("-y");
        cmds.add("-filter_complex");
        cmds.add("[0:v][1:v]overlay=main_w-overlay_w-10:main_h-overlay_h-10[out]"); // 位置
        cmds.add("-map");
        cmds.add("[out]");
        cmds.add("-map");
        cmds.add("0:a");
        cmds.add("-codec:a"); // keep audio
        cmds.add("copy");
        cmds.add(desVideo);

        runSync(cmds, callback);
    }

    /**
     * 将视频转成Gif
     * @param videoPath 视频路径
     * @param gifPath   gif路径
     * @param callback  回调
     */
    public static void convertVideoToGif(String videoPath, String gifPath,
                                         OnCompletionListener callback) {
        ArrayList<String> cmds = new ArrayList<>();
        cmds.add("ffmpeg");
        cmds.add("-i");
        cmds.add(videoPath);

        cmds.add("-f");
        cmds.add("gif");
        cmds.add(gifPath);

        runSync(cmds, callback);
    }

    /**
     * 合并多个视频
     * @param videoPathList 视频列表
     * @param desVideo      输出视频
     * @return
     */
    public static boolean combineVideo(List<String> videoPathList, String desVideo) {
        String tmpFile = "/sdcard/videolist.txt";
        String content = "ffconcat version 1.0\n";

        for (String path : videoPathList) {
            content += "\nfile " + path;
        }

        FileUtils.writeFile(tmpFile, content, false);

        ArrayList<String> cmds = new ArrayList<>();
        cmds.add("ffmpeg");
        cmds.add("-y");

        cmds.add("-safe");
        cmds.add("0");

        cmds.add("-f");
        cmds.add("concat");

        cmds.add("-i");
        cmds.add(tmpFile);


        cmds.add("-c");
        cmds.add("copy");

        cmds.add(desVideo);

        if (VERBOSE) {
            cmds.add(STR_DEBUG_PARAM);
        }

        String[] commands = cmds.toArray(new String[cmds.size()]);
        int result = runSafely(commands);
        FileUtils.deleteFile(tmpFile);

        return result == 1;
    }

    /**
     * 检测视频文件是否正确
     * @param videoPath 视频路径
     * @param time      时间
     * @param picPath
     * @param callback
     */
    public static void getVideoShoot(String videoPath, float time, String picPath,
                                     OnCompletionListener callback) {

        ArrayList<String> cmds = new ArrayList<>();
        cmds.add("ffmpeg");
        cmds.add("-y");
        cmds.add("-ss");
        cmds.add("" + time);
        cmds.add("-i");
        cmds.add(videoPath);
        cmds.add("-r");
        cmds.add("1");
        cmds.add("-vframes");
        cmds.add("1");
//        cmds.add("-vf");
//        cmds.add("select=eq(pict_type\\,I)");
        cmds.add("-an");
        cmds.add("-f");
        cmds.add("mjpeg");
        cmds.add(picPath);

        runSync(cmds, callback);
    }

    /**
     * 裁剪视频
     * @param srcPath   视频路径
     * @param x         x起始坐标
     * @param y         y起始坐标
     * @param width     宽度
     * @param height    高度
     * @param destPath  目标路径
     * @param callback  回调
     */
    public static void cropVideo(String srcPath, int x, int y, int width, int height,
                                 String destPath, OnCompletionListener callback) {
        ArrayList<String> cmds = new ArrayList<>();
        cmds.add("ffmpeg");
        cmds.add("-y");
        cmds.add("-i");
        cmds.add(srcPath);
        cmds.add("-filter:v");
        cmds.add("crop=" + width + ":" + height + ":" + x + ":" + y);
        cmds.add(destPath);

        runSync(cmds, callback);
    }

}
```
至此，我们就已经编写好我们的ffmpeg命令工具了。但到了这里，还不能执行。因为我们还没有在CmakeLists.txt中添加对应的文件，如下：
```
# 设置cmake最低版本
cmake_minimum_required(VERSION 3.4.1)

# 设置路径
set(distribution_DIR ${CMAKE_SOURCE_DIR}/../../../../libs)

# 加载ffmpeg库
add_library( ffmpeg
             SHARED
             IMPORTED )
set_target_properties( ffmpeg
                       PROPERTIES IMPORTED_LOCATION
                       ../../../../libs/armeabi-v7a/libffmpeg.so )

# 加载头文件
# 这里需要添加ffmpeg库的路径，用来编译生成ffmpeg_cmd.so
# 这里如果不引入头文件，会出现libavresample/avresample.h等头文件找不到的情况
include_directories(D:/FFmpeg/ffmpeg-3.3.3
                    src/main/cpp/ffmpeg
                    src/main/cpp/ffmpeg/include )

# 添加自身的jni库
add_library( ffmpeg_cmd

             SHARED

             src/main/cpp/ffmpeg/cmdutils.c
             src/main/cpp/ffmpeg/ffmpeg.c
             src/main/cpp/ffmpeg/ffmpeg_opt.c
             src/main/cpp/ffmpeg/ffmpeg_filter.c
             src/main/cpp/ffmpeg_cmd/ffmpeg_cmd.c
             src/main/cpp/ffmpeg_cmd/ffmpeg_cmd_wrapper.c )

# 查找Android存在的库
find_library( log-lib

              log )

set(CMAKE_EXE_LINKER_FLAGS "-lz -ldl")

# 链接库文件
target_link_libraries(
                       ffmpeg_cmd

                       # ffmpeg库
                       ffmpeg

                       ${log-lib} )
```
sync之后，如果在include_directories 中没有包含ffmpeg库的目录，会提示出错，找不到路径:
![找不到路径](http://upload-images.jianshu.io/upload_images/2103804-6887ef259b50daf2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
原因是我们没有引入头文件，前面复制到cpp目录下的头文件不全，我们并没有用到相应的库，因此打包出来的结果是没有相应的头文件的，比如libavresample，我们用了libswresample。因此这里需要引入ffmpeg库的头文件路径进行编译。
我们编译一个apk包，解压之后，可以看到 libffmpeg_cmd.so已经生成。经过裁剪后的 libffmpeg.so 和 libffmpeg_cmd.so 包体积分别之后3.5MB 和 200K，release 包的体积更小。打包得到libffmpeg_cmd.so之后，我们就可以从CmakeList.txt中注释掉编译libffmpeg_cmd.so库的代码。然后将libffmpeg_cmd.so复制到libs/armeabi-v7a目录。

还有什么不明白的，你可以看我的相机项目 [CainCamera](https://github.com/CainKernel/CainCamera)  是怎么处理的。在录制多段视频完成后，在预览页面点击保存，可以看到DCIM目录下生成了一个合并后的视频，这个就是用了ffmpeg命令执行得到。截止本篇文章发布前，相机项目的多段视频合成功能基本完成，后续会添加合成进度提示等细节，欢迎下载体验和Stars。（由于本人发现FFmpeg命令行在多段视频合成时，如果存在切换滤镜，会出现合成成功，但播放不了的情况，目前多段视频合成部分切换成MediaExtractor + MediaMuxer的方式实现）

