0、编译FFmpeg的库
FFmpeg的编译流程可参考我的文章[windows环境下编译ffmpeg打包成单个so并使用Cmake集成到Android工程中](http://www.jianshu.com/p/ed2266abe28b)，在这里我们不做详细的介绍，这里我们默认你能够把libffmpeg.so编译出来

1、下载SDL的源码
到[SDL官网](http://www.libsdl.org/) 下载源码，我下载的源码版本是2.0.7的。将源码解压后，将android-project 目录复制到另外一个地方并改成自己的工程名字，比如我改成了CainPlayer，将SDL源码根目录的src、include目录、Android.mk文件复制到android-project的jni目录下的sdl目录，如下图所示：
![准备工程文件](http://upload-images.jianshu.io/upload_images/2103804-dd494678c6cc049a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
原来的android-project工程的jni目录下面存在一个src的目录、Android.mk文件以及Application.mk文件，我们暂时不要动这些。这样我们就将SDL的源码导入了android-project工程中了。

2、导入为Android工程
用Android Studio 导入工程，此时，我们得到了一个Android工程，但是这时候还没有编译不了，需要修改build.gradle文件，添加ndk-build的支持，这里我试过Cmake，发现走不通，也不知道是啥原因。添加如下：
```
apply plugin: 'com.android.application'

android {
    compileSdkVersion 26
    buildToolsVersion "26.0.2"

    defaultConfig {
        applicationId "org.libsdl.app"
        minSdkVersion 19
        targetSdkVersion 26

        ndk {
            abiFilter("armeabi-v7a")
            moduleName "SDLmain"
            ldLibs "log", "z", "m", "jnigraphics", "android"
        }
    }

    buildTypes {
        release {
            minifyEnabled false
            proguardFiles getDefaultProguardFile('proguard-android.txt'), 'proguard-rules.txt'
        }
    }

    externalNativeBuild {
        ndkBuild {
            path 'src/main/jni/Android.mk'
        }
    }
}
```
3、添加main代码
然后我们在src/main/jni/src/目录下创建一个native_render.c的文件，文件内容如下：
```
#include "SDL.h" 

 int main(int argc, char *argv[]) {

     SDL_Window *window;

     SDL_Renderer *renderer;

     SDL_Event event;

 

     //配置一个图像缩放的效果,linear效果更平滑,也叫抗锯齿

     //SDL_setenv(SDL_HINT_RENDER_SCALE_QUALITY,"linear",0);

     // 初始化SDL

     if (SDL_Init(SDL_INIT_VIDEO) < 0)

         return 1;

 

     // 创建一个窗口

     window = SDL_CreateWindow("SDL_RenderClear" , SDL_WINDOWPOS_CENTERED,

                               SDL_WINDOWPOS_CENTERED, 0, 0, SDL_WINDOW_SHOWN);

     // 创建一个渲染器

     renderer = SDL_CreateRenderer(window, -1, 0);

 

     // 创建一个Surface

     SDL_Surface *bmp = SDL_LoadBMP("image.bmp" );

     //设置图片中的白色为透明色

     SDL_SetColorKey(bmp, SDL_TRUE, 0xffffff);

     // 创建一个Texture

     SDL_Texture *texture = SDL_CreateTextureFromSurface(renderer, bmp);

     //清除所有事件

     SDL_FlushEvents(SDL_FIRSTEVENT, SDL_LASTEVENT);

     //进入主循环

     while  (1) {

         if  (SDL_PollEvent(&event)) {

             if  (event.type == SDL_QUIT || event.type == SDL_KEYDOWN || event.type == SDL_FINGERDOWN)

                 break;

         }

         //使用红色填充背景

         SDL_SetRenderDrawColor(renderer, 255, 0, 0, 255);

         SDL_RenderClear(renderer);

         // 将纹理布置到渲染器

         SDL_RenderCopy(renderer, texture, NULL, NULL);

         // 刷新屏幕

         SDL_RenderPresent(renderer);

 

     }

      // 释放Surface

     SDL_FreeSurface(bmp);

     //  释放Texture

     SDL_DestroyTexture(texture);

     //释放渲染器

     SDL_DestroyRenderer(renderer);

     //释放窗口

     SDL_DestroyWindow(window);

     //延时

     //SDL_Delay(8000);

     //退出

     SDL_Quit();

     return  0;

}
```
然后，我们将同目录下的Android.mk中的 LOCAL_SRC_FILES 换成native_render.c。如下所示：
```
LOCAL_PATH := $(call my-dir)

include $(CLEAR_VARS)

LOCAL_MODULE := SDLmain

SDL_PATH := ../sdl

LOCAL_C_INCLUDES := $(LOCAL_PATH)/$(SDL_PATH)/include

# Add your application source files here...
LOCAL_SRC_FILES := native_render.c

LOCAL_SHARED_LIBRARIES := SDL2

LOCAL_LDLIBS := -lGLESv1_CM -lGLESv2 -llog

include $(BUILD_SHARED_LIBRARY)
```
我把库的名称改成了SDLmain，这个名字跟前面的build.gradle添加的ndk里面的moduleName "SDLmain"对应，你也可以改成你自己想要的库名称。然后修改jni目录下Application.mk，指定ABI 和 最小支持平台, 如下：
```
APP_ABI := armeabi-v7a
# Min SDK level
APP_PLATFORM=android-19
```
然后在main目录下新建一个assets目录，将官网提到的一个[图片地址](http://www.dinomage.com/wp-content/uploads/2013/01/image.bmp) 保存为image.bmp文件，并放入 assets中。然后回到前面添加的native_render.c文件中，创建surface里面的文件名改成image.bmp,如下：
```
    // 创建一个Surface
    SDL_Surface *bmp = SDL_LoadBMP("image.bmp" );
```
到这里，就可以编译运行android-project工程了。如无意外，即可显示图片了，如下图所示：
![成功运行截图](http://upload-images.jianshu.io/upload_images/2103804-c17c1afe24b9480a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

3、修改包名和图标
SDL的源码已经限定了android-project的包名和方法名称，如果只是修改Java层的报名，这里是会运行失败的。我们来看看SDLActivity文件中的native方法，如下图所示：
![SDLActivity中的native方法](http://upload-images.jianshu.io/upload_images/2103804-031a4720a7a58554.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
如果我们点击跳转，会发现跳转到了sdl/src/core/android/SDL_android.c文件中，然后我们可以查看这样的定义：
```
/* 包名和方法名 */
#define SDL_JAVA_PREFIX                                 org_libsdl_app
#define CONCAT1(prefix, class, function)                CONCAT2(prefix, class, function)
#define CONCAT2(prefix, class, function)                Java_ ## prefix ## _ ## class ## _ ## function
#define SDL_JAVA_INTERFACE(function)                    CONCAT1(SDL_JAVA_PREFIX, SDLActivity, function)
#define SDL_JAVA_AUDIO_INTERFACE(function)              CONCAT1(SDL_JAVA_PREFIX, SDLAudioManager, function)
#define SDL_JAVA_CONTROLLER_INTERFACE(function)         CONCAT1(SDL_JAVA_PREFIX, SDLControllerManager, function)
#define SDL_JAVA_INTERFACE_INPUT_CONNECTION(function)   CONCAT1(SDL_JAVA_PREFIX, SDLInputConnection, function)


/* Java class SDLActivity */
JNIEXPORT void JNICALL SDL_JAVA_INTERFACE(nativeSetupJNI)(
        JNIEnv* mEnv, jclass cls);

JNIEXPORT int JNICALL SDL_JAVA_INTERFACE(nativeRunMain)(
        JNIEnv* env, jclass cls,
        jstring library, jstring function, jobject array);

JNIEXPORT void JNICALL SDL_JAVA_INTERFACE(onNativeDropFile)(
        JNIEnv* env, jclass jcls,
        jstring filename);

JNIEXPORT void JNICALL SDL_JAVA_INTERFACE(onNativeResize)(
        JNIEnv* env, jclass jcls,
        jint width, jint height, jint format, jfloat rate);

JNIEXPORT void JNICALL SDL_JAVA_INTERFACE(onNativeSurfaceChanged)(
        JNIEnv* env, jclass jcls);

JNIEXPORT void JNICALL SDL_JAVA_INTERFACE(onNativeSurfaceDestroyed)(
        JNIEnv* env, jclass jcls);

JNIEXPORT void JNICALL SDL_JAVA_INTERFACE(onNativeKeyDown)(
        JNIEnv* env, jclass jcls,
        jint keycode);

JNIEXPORT void JNICALL SDL_JAVA_INTERFACE(onNativeKeyUp)(
        JNIEnv* env, jclass jcls,
        jint keycode);

JNIEXPORT void JNICALL SDL_JAVA_INTERFACE(onNativeKeyboardFocusLost)(
        JNIEnv* env, jclass jcls);

JNIEXPORT void JNICALL SDL_JAVA_INTERFACE(onNativeTouch)(
        JNIEnv* env, jclass jcls,
        jint touch_device_id_in, jint pointer_finger_id_in,
        jint action, jfloat x, jfloat y, jfloat p);

JNIEXPORT void JNICALL SDL_JAVA_INTERFACE(onNativeMouse)(
        JNIEnv* env, jclass jcls,
        jint button, jint action, jfloat x, jfloat y);

JNIEXPORT void JNICALL SDL_JAVA_INTERFACE(onNativeAccel)(
        JNIEnv* env, jclass jcls,
        jfloat x, jfloat y, jfloat z);

JNIEXPORT void JNICALL SDL_JAVA_INTERFACE(onNativeClipboardChanged)(
        JNIEnv* env, jclass jcls);

JNIEXPORT void JNICALL SDL_JAVA_INTERFACE(nativeLowMemory)(
        JNIEnv* env, jclass cls);

JNIEXPORT void JNICALL SDL_JAVA_INTERFACE(nativeQuit)(
        JNIEnv* env, jclass cls);

JNIEXPORT void JNICALL SDL_JAVA_INTERFACE(nativePause)(
        JNIEnv* env, jclass cls);

JNIEXPORT void JNICALL SDL_JAVA_INTERFACE(nativeResume)(
        JNIEnv* env, jclass cls);

JNIEXPORT jstring JNICALL SDL_JAVA_INTERFACE(nativeGetHint)(
        JNIEnv* env, jclass cls,
        jstring name);

/* Java class SDLInputConnection */
JNIEXPORT void JNICALL SDL_JAVA_INTERFACE_INPUT_CONNECTION(nativeCommitText)(
        JNIEnv* env, jclass cls,
        jstring text, jint newCursorPosition);

JNIEXPORT void JNICALL SDL_JAVA_INTERFACE_INPUT_CONNECTION(nativeSetComposingText)(
        JNIEnv* env, jclass cls,
        jstring text, jint newCursorPosition);

/* Java class SDLAudioManager */
JNIEXPORT void JNICALL SDL_JAVA_AUDIO_INTERFACE(nativeSetupJNI)(
        JNIEnv *env, jclass jcls);

/* Java class SDLControllerManager */
JNIEXPORT void JNICALL SDL_JAVA_CONTROLLER_INTERFACE(nativeSetupJNI)(
        JNIEnv *env, jclass jcls);

JNIEXPORT jint JNICALL SDL_JAVA_CONTROLLER_INTERFACE(onNativePadDown)(
        JNIEnv* env, jclass jcls,
        jint device_id, jint keycode);

JNIEXPORT jint JNICALL SDL_JAVA_CONTROLLER_INTERFACE(onNativePadUp)(
        JNIEnv* env, jclass jcls,
        jint device_id, jint keycode);

JNIEXPORT void JNICALL SDL_JAVA_CONTROLLER_INTERFACE(onNativeJoy)(
        JNIEnv* env, jclass jcls,
        jint device_id, jint axis, jfloat value);

JNIEXPORT void JNICALL SDL_JAVA_CONTROLLER_INTERFACE(onNativeHat)(
        JNIEnv* env, jclass jcls,
        jint device_id, jint hat_id, jint x, jint y);

JNIEXPORT jint JNICALL SDL_JAVA_CONTROLLER_INTERFACE(nativeAddJoystick)(
        JNIEnv* env, jclass jcls,
        jint device_id, jstring device_name, jstring device_desc, jint is_accelerometer,
        jint nbuttons, jint naxes, jint nhats, jint nballs);

JNIEXPORT jint JNICALL SDL_JAVA_CONTROLLER_INTERFACE(nativeRemoveJoystick)(
        JNIEnv* env, jclass jcls,
        jint device_id);

JNIEXPORT jint JNICALL SDL_JAVA_CONTROLLER_INTERFACE(nativeAddHaptic)(
        JNIEnv* env, jclass jcls,
        jint device_id, jstring device_name);

JNIEXPORT jint JNICALL SDL_JAVA_CONTROLLER_INTERFACE(nativeRemoveHaptic)(
        JNIEnv* env, jclass jcls,
        jint device_id);
```
这里就定义了包名org_libsdl_app，也就是build.gradle中的applicationId "org.libsdl.app"以及Manifest.xml，以及SDLActivity等java层的东西。如果单纯修改Java层的东西，运行会崩溃，说找不到相应的类的。这里可以改成你自己的包名，比如我改成了自己的com.cgfay.cainplayer，注意，这里build.gradle 和Manifest.xml都的包名需要一致。然后将SDL_android.c中的SDL_JAVA_PREFIX 改成 com_cgfay_cainplayer。改完之后，重新编译一次，重新运行，我们就可以发现，已经成功将报名改掉了，而且编译运行也没有问题。如下所示：
![改完报名后](http://upload-images.jianshu.io/upload_images/2103804-59d6b848ba5ad1cf.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
想要修改为自己的图标的话，把@drawable/ic_launcher的图标换成自己的图标就行。
关于native_render.c中的main方法，其实这里也是可以修改的，定义在sdl/include/SDL_main.h中：
```
#if defined(SDL_MAIN_NEEDED) || defined(SDL_MAIN_AVAILABLE)
#define main    SDL_main
#endif

/**
 *  The prototype for the application's main() function
 */
extern C_LINKAGE DECLSPEC int SDL_main(int argc, char *argv[]);
```
如果我们想要修改成我们自己的名称，#define main    SDL_main改成我们自己的方法就行，比如#define run SDL_main，不过参数接口不能做修改。

4、添加FFmpeg库
首先在src/main目录下新建一个libs目录，然后再在libs目录下新建一个armeabi-v7a的目录，将前面已经编译好的libffmpeg.so库复制到该目录下。在src/main/jni 目录下新建一个ffmpeg目录，然后在目录下新建一个include目录，将ffmpeg的头文件复制到这里。在ffmpeg目录下新建一个Android.mk文件，文件内容如下：
```
LOCAL_PATH := $(call my-dir)

###########################
#
# FFmpeg shared library
#
###########################

include $(CLEAR_VARS)

LOCAL_MODULE:= ffmpeg

LOCAL_SRC_FILES:= $(LOCAL_PATH)/../libs/armeabi-v7a/libffmpeg.so

LOCAL_EXPORT_C_INCLUDES:= $(LOCAL_PATH)/include

include $(PREBUILT_SHARED_LIBRARY)
```
这时我们就把ffmpeg的共享库加载进来了。我们修改src/main/jni/src/Android.mk文件，在Android.mk文件中添加libffmpeg共享库，并把ffmpeg的头文件加载进来，如下所示：
```
LOCAL_PATH := $(call my-dir)

include $(CLEAR_VARS)

LOCAL_MODULE := SDLmain

SDL_PATH := ../sdl
FFMPEG_PATH := ../ffmpeg

LOCAL_C_INCLUDES := $(LOCAL_PATH)/$(SDL_PATH)/include \
                    $(LOCAL_PATH)/$(FFMPEG_PATH)/include

# Add your application source files here...
LOCAL_SRC_FILES := native_render.c

LOCAL_SHARED_LIBRARIES := SDL2 ffmpeg

LOCAL_LDLIBS := -lGLESv1_CM -lGLESv2 -llog

include $(BUILD_SHARED_LIBRARY)
```
重新编译，我们就把FFmpeg的共享库给编译进去了。我们需要在SDLActivity的方法中添加ffmpeg库，如下：
```
protected String[] getLibraries() {
        return new String[] {
                "SDL2",
                "ffmpeg",   // 顺序要在SDLmain之前
                "SDLmain"
        };
    }
```
然后，我们在native_rander.c中添加打印avcodec_configuration()的方法，如下：
```
#include "SDL.h" 

#ifdef __ANDROID__
#include <jni.h>
#include <android/log.h>
#define ALOGE(...) __android_log_print(ANDROID_LOG_ERROR,"ERROR: ", __VA_ARGS__)
#define ALOGI(...) __android_log_print(ANDROID_LOG_INFO,"INFO: ", __VA_ARGS__)
#else
#define ALOGE(format, ...) printf("ERROR: " format "\n",##__VA_ARGS__)
#define ALOGI(format, ...) printf("INFO: " format "\n",##__VA_ARGS__)
#endif

#include "libavcodec/avcodec.h"

 int main(int argc, char *argv[]) {
     // 打印ffmpeg信息
     const char *str = avcodec_configuration();
     ALOGI("avcodec_configuration: %s", str);

     SDL_Window *window;

     SDL_Renderer *renderer;

     SDL_Event event;

     //配置一个图像缩放的效果,linear效果更平滑,也叫抗锯齿
     SDL_setenv(SDL_HINT_RENDER_SCALE_QUALITY,"linear", 0);

     // 初始化SDL
     if (SDL_Init(SDL_INIT_VIDEO) < 0) {
         return 1;
     }

     // 创建一个窗口
     window = SDL_CreateWindow("SDL_RenderClear" , SDL_WINDOWPOS_CENTERED,

                               SDL_WINDOWPOS_CENTERED, 0, 0, SDL_WINDOW_SHOWN);

     // 创建一个渲染器
     renderer = SDL_CreateRenderer(window, -1, 0);

     // 创建一个Surface
     SDL_Surface *bmp = SDL_LoadBMP("image.bmp" );

     //设置图片中的白色为透明色
     SDL_SetColorKey(bmp, SDL_TRUE, 0xffffff);

     // 创建一个Texture
     SDL_Texture *texture = SDL_CreateTextureFromSurface(renderer, bmp);

     //清除所有事件
     SDL_FlushEvents(SDL_FIRSTEVENT, SDL_LASTEVENT);

     //进入主循环
     while  (1) {
         if  (SDL_PollEvent(&event)) {
             if  (event.type == SDL_QUIT || event.type == SDL_KEYDOWN || event.type == SDL_FINGERDOWN)
                 break;

         }

         //使用红色填充背景
         SDL_SetRenderDrawColor(renderer, 255, 0, 0, 255);

         SDL_RenderClear(renderer);

         // 将纹理布置到渲染器
         SDL_RenderCopy(renderer, texture, NULL, NULL);

         // 刷新屏幕
         SDL_RenderPresent(renderer);
     }

      // 释放Surface
     SDL_FreeSurface(bmp);

     //  释放Texture
     SDL_DestroyTexture(texture);

     //释放渲染器
     SDL_DestroyRenderer(renderer);

     //释放窗口
     SDL_DestroyWindow(window);

     //延时
     //SDL_Delay(8000);

     //退出
     SDL_Quit();

     return  0;

}
```
重新Refresh Linked C++ Projects之后编译运行，可以看到已经打印出以下的log：
```
12-22 15:50:56.249 8752-8777/com.cgfay.cainplayer I/INFO:: avcodec_configuration: --prefix=./android/arm-v7a --arch=arm --cpu=armv7-a --target-os=android --enable-cross-compile --cross-prefix='D:/Android/sdk/ndk-bundle/toolchains/arm-linux-androideabi-4.9/prebuilt/windows-x86_64/bin/arm-linux-androideabi-' --sysroot='D:/Android/sdk/ndk-bundle/platforms/android-19/arch-arm' --extra-cflags='-I/d/FFmpeg/ffmpeg-3.3.3/libx264/android/arm/include -ID:/Android/sdk/ndk-bundle/platforms/android-19/arch-arm/usr/include' --extra-ldflags=-L/d/FFmpeg/ffmpeg-3.3.3/libx264/android/arm/lib --cc='D:/Android/sdk/ndk-bundle/toolchains/arm-linux-androideabi-4.9/prebuilt/windows-x86_64/bin/arm-linux-androideabi-gcc' --nm='D:/Android/sdk/ndk-bundle/toolchains/arm-linux-androideabi-4.9/prebuilt/windows-x86_64/bin/arm-linux-androideabi-nm' --disable-shared --enable-static --enable-gpl --enable-version3 --enable-pthreads --enable-runtime-cpudetect --enable-small --disable-network --disable-vda --disable-iconv --enable-asm --enable-neon --enable-yasm --disable-encoders --enable-libx264 --ena
```
此时，我们已经将FFmpeg + SDL2.0 编译成功，并集成到SDLmain中了。










