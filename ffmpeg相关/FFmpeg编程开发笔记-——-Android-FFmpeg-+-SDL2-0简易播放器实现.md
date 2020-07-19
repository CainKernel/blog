在前一篇文章[FFmpeg编程开发笔记 —— Android 移植 FFmpeg + SDL2.0 库](https://www.jianshu.com/p/ff60dc41f876)中，我们介绍了将FFmpeg + SDL2.0移植到Android中，并成功运行。接下来我们在这基础上，利用FFmpeg  和 SDL 实现一个简易的播放器。

1、替换native_render.c文件
我们在src/main/jni/src/目录下，将native_render.c复制一份并重命名为simple_player.c，然后将同目录中的Android.mk文件中的LOCAL_SRC_FILES的native_render.c 替换成 simple_player.c 。然后使用Build->Refresh Linked C++ Projects。这时我们就将simple_player.c文件链接到了SDLMain库中，替换掉原来的native_render.c文件了。

2、开始编写播放器的代码
添加头文件、定义Android Native层的Log等：
```
#include <jni.h>
#include <android/native_window_jni.h>
#include "SDL.h"
#include "SDL_thread.h"
#include "SDL_events.h"
#include "libavcodec/avcodec.h"
#include "libavformat/avformat.h"
#include "libavutil/pixfmt.h"
#include "libswscale/swscale.h"

#ifdef __ANDROID__
#include <jni.h>
#include <android/log.h>
#define ALOGE(...) __android_log_print(ANDROID_LOG_ERROR,"ERROR: ", __VA_ARGS__)
#define ALOGI(...) __android_log_print(ANDROID_LOG_INFO,"INFO: ", __VA_ARGS__)
#else
#define ALOGE(format, ...) printf("ERROR: " format "\n",##__VA_ARGS__)
#define ALOGI(format, ...) printf("INFO: " format "\n",##__VA_ARGS__)
#endif

```
然后在main方法里面执行FFmpeg查找视频文件、打开视频流等操作，这里的流程就是FFmpeg的通用流程，基本上格式都是固定的，如下所示：
```
// 注册所有支持的文件格式以及编解码器
    av_register_all();
    // 分配一个AVFormatContext
    pFormatCtx = avformat_alloc_context();

    // 判断文件是否能打开
    if (avformat_open_input(&pFormatCtx, file_path, NULL, NULL) != 0) {
        ALOGE("can't open the file. \n");
        return -1;
    }
    // 判断能够找到文件流信息
    if (avformat_find_stream_info(pFormatCtx, NULL) < 0) {
        ALOGE("Could't find stream infomation.\n");
        return -1;
    }

    // 打印文件信息
    av_dump_format(pFormatCtx, -1, file_path, 0);
    videoStream = -1;
    for (i = 0; i < pFormatCtx->nb_streams; i++) {
        // 新版ffmpeg将AVCodecContext *codec 替换成*codecpar
//        if (pFormatCtx->streams[i]->codec->codec_type == AVMEDIA_TYPE_VIDEO) {
        if (pFormatCtx->streams[i]->codecpar->codec_type == AVMEDIA_TYPE_VIDEO) {
            videoStream = i;
            break;
        }
    }

    // 判断是否找到video stream
    ALOGI("videoStream: %d", videoStream);
    if (videoStream == -1) {
        ALOGE("Didn't find a video stream.\n");
        return -1;
    }

    // 获取videoStream对应的codec context
    pCodecParameters = pFormatCtx->streams[videoStream]->codecpar;

    // 查找解码器对应的Id
    pCodec = avcodec_find_decoder(pCodecParameters->codec_id);

    // 没有找到
    if (pCodec == NULL) {
        ALOGE("Codec not found.\n");
        return -1;
    }

    // 根据解码器Id创建解码上下文
    pCodecCtx = avcodec_alloc_context3(pCodec);

    if (avcodec_parameters_to_context(pCodecCtx, pCodecParameters) < 0) {
        ALOGE("copy the codec parameters to context fail!");
        return -1;
    }

    // 打开解码器
    if (avcodec_open2(pCodecCtx, pCodec, NULL) < 0) {
        ALOGE("Could not open codec.\n");
        return -1;
    }

    // 创建frame对象
    pFrame = av_frame_alloc();
    if (pFrame == NULL) {
        ALOGE("Unable to allocate an AVFrame!\n");
        return -1;
    }

    // 渲染的YUV
    pFrameYUV = av_frame_alloc();
```
到这里，FFmpeg播放视频的初始化流程基本完成，接下来就是初始化SDL了，初始化也是比较流程化的东西，如下所示：
```
//---------------------------init sdl---------------------------//
    // SDL初始化是否成功
    if (SDL_Init(SDL_INIT_VIDEO | SDL_INIT_AUDIO | SDL_INIT_TIMER)) {
        ALOGE("Could not initialize SDL - %s. \n", SDL_GetError());
        return -1;
    }
    // 创建窗口
    screen = SDL_CreateWindow("My Player Window", SDL_WINDOWPOS_UNDEFINED,
                              SDL_WINDOWPOS_UNDEFINED, pCodecCtx->width, pCodecCtx->height,
                              SDL_WINDOW_FULLSCREEN | SDL_WINDOW_OPENGL);

    if (!screen) {
        ALOGE("SDL: could not set video mode - exiting\n");
        return -1;
    }

    // 创建渲染器
    SDL_Renderer *renderer = SDL_CreateRenderer(screen, -1, 0);

    // 创建Texture
    videoTexture = SDL_CreateTexture(renderer, SDL_PIXELFORMAT_YV12,
                            SDL_TEXTUREACCESS_STREAMING, pCodecCtx->width, pCodecCtx->height);

```
初始化完成后，我们来创建缓冲，为AVPacket分配内存:
```
// 创建缓冲
    numBytes = avpicture_get_size(AV_PIX_FMT_YUV420P, pCodecCtx->width,
                                  pCodecCtx->height);
    out_buffer = (uint8_t *) av_malloc(numBytes * sizeof(uint8_t));
    avpicture_fill((AVPicture *) pFrameYUV, out_buffer, AV_PIX_FMT_YUV420P,
                   pCodecCtx->width, pCodecCtx->height);

    rect.x = 0;
    rect.y = 0;
    rect.w = pCodecCtx->width;
    rect.h = pCodecCtx->height;

    int y_size = pCodecCtx->width * pCodecCtx->height;

    // 创建packet
    packet = (AVPacket *) malloc(sizeof(AVPacket));
    av_new_packet(packet, y_size);
```
到这里，基本上都准备好了。接下来就是从视频文件中逐帧读取出来，并绘制到Surface上，同时给SDL添加返回按钮点击事件监听：
```
    // 读取帧
    while (av_read_frame(pFormatCtx, packet) >= 0) {
        if (packet->stream_index == videoStream) {

            // 旧版本使用方法
//            getFrameCode = avcodec_decode_video2(pCodecCtx, pFrame, &got_picture, packet);

            // 新版本的获取帧方法
            getPacketCode = avcodec_send_packet(pCodecCtx, packet);
            if (getPacketCode == 0) {
                getFrameCode = avcodec_receive_frame(pCodecCtx, pFrame);
                if (getFrameCode < 0) {
                    ALOGE("decode error.\n");
                    return -1;
                }
                if (getFrameCode == 0) {
                    // 缩放帧
                    sws_scale(img_convert_ctx, (uint8_t const * const *) pFrame->data,
                              pFrame->linesize, 0, pCodecCtx->height,
                              pFrameYUV->data, pFrameYUV->linesize);

                    // iPitch 计算yuv一行数据占的字节数
                    SDL_UpdateYUVTexture(videoTexture, &rect,
                                         pFrameYUV->data[0], pFrameYUV->linesize[0],
                                         pFrameYUV->data[1], pFrameYUV->linesize[1],
                                         pFrameYUV->data[2], pFrameYUV->linesize[2]);
                    // 清空数据
                    SDL_RenderClear(renderer);
                    // 复制数据
                    SDL_RenderCopy(renderer, videoTexture, &rect, &rect);
                    // 渲染到屏幕
                    SDL_RenderPresent(renderer);
                    // 延时40毫秒
                    SDL_Delay(40);

                } else if (getFrameCode == AVERROR(EAGAIN)) {
                    ALOGE("%s", "Frame is not available right, please try another input");
                } else if (getFrameCode == AVERROR_EOF) {
                    ALOGE("%s", "the decoder has been fully flushed");
                } else if (getFrameCode == AVERROR(EINVAL)) {
                    ALOGE("%s", "codec not opened, or it is an encoder");
                } else {
                    ALOGI("%s", "legitimate decoding errors");
                }
            }
        }
        // 释放AVPacket
        av_packet_unref(packet);

        // 处理事件
        if (SDL_PollEvent(&event)) {
            SDL_bool needToQuit = SDL_FALSE;
            switch (event.type) {
                // 退出
                case SDL_QUIT:
                // 点击返回键
                case SDL_KEYDOWN:
                    needToQuit = SDL_TRUE;
                    break;
                default:
                    break;
            }

            // 是否需要停止，退出循环
            if (needToQuit) {
                SDL_Quit();
                break;
            }
        }
    }
```
最后，销毁创建的用于渲染显示的Texture，销毁FFmpeg的变量，防止内存泄漏：
```
// 销毁Texture
    SDL_DestroyTexture(videoTexture);

    // 释放缓冲和对象
    sws_freeContext(img_convert_ctx);
    av_free(out_buffer);
    av_free(pFrame);
    av_free(pFrameYUV);
    avcodec_parameters_free(&pCodecParameters);
    avcodec_close(pCodecCtx);
    avformat_close_input(&pFormatCtx);
```
到这里，简易播放器就实现了。但这样是否就可以使用了呢？还不行。首先，我们在main方法里面读取了argv[1] 作为路径。那么这个路径是怎么来的呢？是从SDLActivity里面来的。我们将SDLActivity中的getArguments()方法修改成如下：
```
protected String[] getArguments() {
        String sdcard = Environment.getExternalStorageDirectory().getAbsolutePath();
        String path = sdcard + "/video.mp4";
        return new String[]{ path };
    }
```
然后在手机上复制一个名为video.mp4的文件到存储根目录，也就是 /storage/emulated/0/目录，Android
 Studio重新调用Build->Refresh Linked C++ Projects，然后再运行程序。如无意外，我们就能播放视频了。




