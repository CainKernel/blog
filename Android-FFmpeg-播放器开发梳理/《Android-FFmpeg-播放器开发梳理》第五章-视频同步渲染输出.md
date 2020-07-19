前面一章，我们讲解了音频重采样以及变速变调处理的逻辑。这一章我们将会讲解视频同步的处理逻辑。
### MediaClock 时钟对象
MediaClock 时钟对象用于记录pts、上一次更新时间以及时间差值等数据。其代码如下：
```
class MediaClock {

public:
    MediaClock();

    virtual ~MediaClock();

    // 初始化
    void init();

    // 获取时钟
    double getClock();

    // 设置时钟
    void setClock(double pts, double time);

    // 设置时钟
    void setClock(double pts);

    // 设置速度
    void setSpeed(double speed);

    // 同步到从属时钟
    void syncToSlave(MediaClock *slave);

    // 获取时钟速度
    double getSpeed() const;

private:
    double pts;
    double pts_drift;
    double last_updated;
    double speed;
    int paused;
};

MediaClock::MediaClock() {
    init();
}

MediaClock::~MediaClock() {

}

void MediaClock::init() {
    speed = 1.0;
    paused = 0;
    setClock(NAN);
}

double MediaClock::getClock() {
    if (paused) {
        return pts;
    } else {
        double time = av_gettime_relative() / 1000000.0;
        return pts_drift + time - (time - last_updated) * (1.0 - speed);
    }
}

void MediaClock::setClock(double pts, double time) {
    this->pts = pts;
    this->last_updated = time;
    this->pts_drift = this->pts - time;
}

void MediaClock::setClock(double pts) {
    double time = av_gettime_relative() / 1000000.0;
    setClock(pts, time);
}

void MediaClock::setSpeed(double speed) {
    setClock(getClock());
    this->speed = speed;
}

void MediaClock::syncToSlave(MediaClock *slave) {
    double clock = getClock();
    double slave_clock = slave->getClock();
    if (!isnan(slave_clock) && (isnan(clock) || fabs(clock - slave_clock) > AV_NOSYNC_THRESHOLD)) {
        setClock(slave_clock);
    }
}

double MediaClock::getSpeed() const {
    return speed;
}
```

### MediaSync 媒体同步器
MediaSync 媒体同步器包含了音频时钟、视频时钟以及外部时钟，分别对应三种同步类型。同时媒体同步器另外开启了一条同步视频的线程。在线程中不断刷新并记录剩余时间，根据剩余时间来判断是否更新视频画面。refreshVideo刷新视频画面的代码如下：
```
void MediaSync::refreshVideo(double *remaining_time) {
    double time;

    // 检查外部时钟
    if (!abortRequest & !playerState->pauseRequest && playerState->realTime &&
        playerState->syncType == AV_SYNC_EXTERNAL) {
        checkExternalClockSpeed();
    }

    Frame *af = (Frame *) av_mallocz(sizeof(Frame));
    af->frame = av_frame_alloc();

    for (;;) {

        if (abortRequest || playerState->abortRequest || !videoDecoder) {
            break;
        }

        // 判断是否存在帧队列是否存在数据
        if (videoDecoder->getFrameSize() > 0) {
            double lastDuration, duration, delay;
            Frame *currentFrame, *lastFrame;
            // 上一帧
            lastFrame = videoDecoder->getFrameQueue()->lastFrame();
            // 当前帧
            currentFrame = videoDecoder->getFrameQueue()->currentFrame();
            // 判断是否需要强制更新帧的时间
            if (frameTimerRefresh) {
                frameTimer = av_gettime_relative() / 1000000.0;
                frameTimerRefresh = 0;
            }

            // 如果处于暂停状态，则直接显示
            if (playerState->abortRequest || playerState->pauseRequest) {
                break;
            }

            // 计算上一次显示时长
            lastDuration = calculateDuration(lastFrame, currentFrame);
            // 根据上一次显示的时长，计算延时
            delay = calculateDelay(lastDuration);
            // 获取当前时间
            time = av_gettime_relative() / 1000000.0;
            if (isnan(frameTimer) || time < frameTimer) {
                frameTimer = time;
            }
            // 如果当前时间小于帧计时器的时间 + 延时时间，则表示还没到当前帧
            if (time < frameTimer + delay) {
                *remaining_time = FFMIN(frameTimer + delay - time, *remaining_time);
                break;
            }

            // 更新帧计时器
            frameTimer += delay;
            // 帧计时器落后当前时间超过了阈值，则用当前的时间作为帧计时器时间
            if (delay > 0 && time - frameTimer > AV_SYNC_THRESHOLD_MAX) {
                frameTimer = time;
            }

            // 更新视频时钟的pts
            mMutex.lock();
            if (!isnan(currentFrame->pts)) {
                videoClock->setClock(currentFrame->pts);
                extClock->syncToSlave(videoClock);
            }
            mMutex.unlock();

            // 如果队列中还剩余超过一帧的数据时，需要拿到下一帧，然后计算间隔，并判断是否需要进行舍帧操作
            if (videoDecoder->getFrameSize() > 1) {
                Frame *nextFrame = videoDecoder->getFrameQueue()->nextFrame();
                duration = calculateDuration(currentFrame, nextFrame);
                // 如果不处于同步到视频状态，并且处于跳帧状态，则跳过当前帧
                if ((time > frameTimer + duration)
                    && (playerState->frameDrop > 0
                        || (playerState->frameDrop && playerState->syncType != AV_SYNC_VIDEO))) {
                    videoDecoder->getFrameQueue()->popFrame();
                    continue;
                }
            }

            // 下一帧
            videoDecoder->getFrameQueue()->popFrame();
            forceRefresh = 1;
        }

        break;
    }

    // 显示画面
    if (!playerState->displayDisable && forceRefresh && videoDecoder
        && videoDecoder->getFrameQueue()->getShowIndex()) {
        renderVideo();
    }
    forceRefresh = 0;
}

void MediaSync::checkExternalClockSpeed() {
    if (videoDecoder && videoDecoder->getPacketSize() <= EXTERNAL_CLOCK_MIN_FRAMES
        || audioDecoder && audioDecoder->getPacketSize() <= EXTERNAL_CLOCK_MIN_FRAMES) {
        extClock->setSpeed(FFMAX(EXTERNAL_CLOCK_SPEED_MIN,
                                 extClock->getSpeed() - EXTERNAL_CLOCK_SPEED_STEP));
    } else if ((!videoDecoder || videoDecoder->getPacketSize() > EXTERNAL_CLOCK_MAX_FRAMES)
               && (!audioDecoder || audioDecoder->getPacketSize() > EXTERNAL_CLOCK_MAX_FRAMES)) {
        extClock->setSpeed(FFMIN(EXTERNAL_CLOCK_SPEED_MAX,
                                 extClock->getSpeed() + EXTERNAL_CLOCK_SPEED_STEP));
    } else {
        double speed = extClock->getSpeed();
        if (speed != 1.0) {
            extClock->setSpeed(speed + EXTERNAL_CLOCK_SPEED_STEP * (1.0 - speed) / fabs(1.0 - speed));
        }
    }
}

double MediaSync::calculateDelay(double delay) {
    double sync_threshold, diff = 0;
    // 如果不是同步到视频流，则需要计算延时时间
    if (playerState->syncType != AV_SYNC_VIDEO) {
        // 计算差值
        diff = videoClock->getClock() - getMasterClock();
        // 用差值与同步阈值计算延时
        sync_threshold = FFMAX(AV_SYNC_THRESHOLD_MIN, FFMIN(AV_SYNC_THRESHOLD_MAX, delay));
        if (!isnan(diff) && fabs(diff) < maxFrameDuration) {
            if (diff <= -sync_threshold) {
                delay = FFMAX(0, delay + diff);
            } else if (diff >= sync_threshold && delay > AV_SYNC_FRAMEDUP_THRESHOLD) {
                delay = delay + diff;
            } else if (diff >= sync_threshold) {
                delay = 2 * delay;
            }
        }
    }

    av_log(NULL, AV_LOG_TRACE, "video: delay=%0.3f A-V=%f\n", delay, -diff);

    return delay;
}

double MediaSync::calculateDuration(Frame *vp, Frame *nextvp) {
    double duration = nextvp->pts - vp->pts;
    if (isnan(duration) || duration <= 0 || duration > maxFrameDuration) {
        return vp->duration;
    } else {
        return duration;
    }
}
```
以上代码就是线程运行起来后，通过不断地调用refreshVideo(double *remaining_time)更新一个视频帧。refreshVideo(double *remaining_time) 方法通过不断取出frameQueue中上一帧、当前帧、下一帧进行对比计算得到帧间时间间隔、需要延时时间、MediaSync的计时器落后的时间是否超过时间阈值，如果超过阈值，则用新的时间作为当前计时器的值，并更新视频时钟对象。接着判断是否存在下一帧，如果存在，则判断需要要进行丢帧处理。在处理完上述逻辑之后，我们就可以渲染输出视频画面了。

### 视频转码输出
经过前面的处理后，进入渲染输出视频画面的逻辑，代码如下：
```
void MediaSync::renderVideo() {
    mMutex.lock();
    if (!videoDecoder || !videoDevice) {
        mMutex.unlock();
        return;
    }
    Frame *vp = videoDecoder->getFrameQueue()->lastFrame();
    int ret = 0;
    if (!vp->uploaded) {
        // 根据图像格式更新纹理数据
        switch (vp->frame->format) {
            // YUV420P 和 YUVJ420P 除了色彩空间不一样之外，其他的没什么区别
            // YUV420P表示的范围是 16 ~ 235，而YUVJ420P表示的范围是0 ~ 255
            // 这里做了兼容处理，后续可以优化，shader已经过验证
            case AV_PIX_FMT_YUVJ420P:
            case AV_PIX_FMT_YUV420P: {

                // 初始化纹理
                videoDevice->onInitTexture(vp->frame->width, vp->frame->height,
                                           FMT_YUV420P, BLEND_NONE);

                if (vp->frame->linesize[0] < 0 || vp->frame->linesize[1] < 0 || vp->frame->linesize[2] < 0) {
                    av_log(NULL, AV_LOG_ERROR, "Negative linesize is not supported for YUV.\n");
                    return;
                }
                ret = videoDevice->onUpdateYUV(vp->frame->data[0], vp->frame->linesize[0],
                                               vp->frame->data[1], vp->frame->linesize[1],
                                               vp->frame->data[2], vp->frame->linesize[2]);
                if (ret < 0) {
                    return;
                }
                break;
            }

            // 直接渲染BGRA，对应的是shader->argb格式
            case AV_PIX_FMT_BGRA: {
                videoDevice->onInitTexture(vp->frame->width, vp->frame->height,
                                           FMT_ARGB, BLEND_NONE);
                ret = videoDevice->onUpdateARGB(vp->frame->data[0], vp->frame->linesize[0]);
                if (ret < 0) {
                    return;
                }
                break;
            }

            // 其他格式转码成BGRA格式再做渲染
            default: {
                swsContext = sws_getCachedContext(swsContext,
                                                  vp->frame->width, vp->frame->height,
                                                  (AVPixelFormat) vp->frame->format,
                                                  vp->frame->width, vp->frame->height,
                                                  AV_PIX_FMT_BGRA, SWS_BICUBIC, NULL, NULL, NULL);
                if (!mBuffer) {
                    int numBytes = av_image_get_buffer_size(AV_PIX_FMT_BGRA, vp->frame->width, vp->frame->height, 1);
                    mBuffer = (uint8_t *)av_malloc(numBytes * sizeof(uint8_t));
                    av_image_fill_arrays(pFrameARGB->data, pFrameARGB->linesize, mBuffer, AV_PIX_FMT_BGRA,
                                         vp->frame->width, vp->frame->height, 1);
                }
                if (swsContext != NULL) {
                    sws_scale(swsContext, (uint8_t const *const *) vp->frame->data,
                              vp->frame->linesize, 0, vp->frame->height,
                              pFrameARGB->data, pFrameARGB->linesize);
                }

                videoDevice->onInitTexture(vp->frame->width, vp->frame->height,
                                           FMT_ARGB, BLEND_NONE);
                ret = videoDevice->onUpdateARGB(pFrameARGB->data[0], pFrameARGB->linesize[0]);
                if (ret < 0) {
                    return;
                }
                break;
            }
        }
        vp->uploaded = 1;
    }
    // 请求渲染视频
    if (videoDevice != NULL) {
        videoDevice->onRequestRender(vp->frame->linesize[0] < 0 ? FLIP_VERTICAL : FLIP_NONE);
    }
    mMutex.unlock();
}
```
视频输出的逻辑就是根据视频帧(AVFrame)的格式，判断是否需要进行转码，然后通知VideoDevice请求渲染画面。

### 视频输出对象：
VideoDevice 是一个视频设备的基类，主要包括结束方法(terminate)、初始化文理(onInitTexture)、更新YUV/ARGB数据(onUpdateYUV/onUpdateARGB)，请求渲染画面(onRequestRender)几个方法组成，基类啥也不做处理，其代码如下：
```
class VideoDevice {
public:
    VideoDevice();

    virtual ~VideoDevice();

    virtual void terminate();

    // 初始化视频纹理宽高
    virtual void onInitTexture(int width, int height, TextureFormat format, BlendMode blendMode);

    // 更新YUV数据
    virtual int onUpdateYUV(uint8_t *yData, int yPitch,
                            uint8_t *uData, int uPitch,
                            uint8_t *vData, int vPitch);

    // 更新ARGB数据
    virtual int onUpdateARGB(uint8_t *rgba, int pitch);

    // 请求渲染
    virtual int onRequestRender(FlipDirection direction);

};
```
然后我们构建一个Android环境下的视频输出设备。这里我们采用OpenGLES来渲染视频。因此我们创建GLESDevice类，继承VideoDevice基类。其代码实现如下：
```
class GLESDevice : public VideoDevice {
public:
    GLESDevice();

    virtual ~GLESDevice();

    void surfaceCreated(ANativeWindow *window);

    void surfaceChanged(int width, int height);

    void surfaceDestroyed();

    void terminate() override;

    void terminate(bool releaseContext);

    void onInitTexture(int width, int height, TextureFormat format, BlendMode blendMode) override;

    int onUpdateYUV(uint8_t *yData, int yPitch, uint8_t *uData, int uPitch,
                    uint8_t *vData, int vPitch) override;

    int onUpdateARGB(uint8_t *rgba, int pitch) override;

    int onRequestRender(FlipDirection direction) override;


private:
    Mutex mMutex;
    Condition mCondition;

    ANativeWindow *mWindow;             // Surface窗口
    EGLSurface eglSurface;              // eglSurface
    EglHelper *eglHelper;               // EGL帮助器
    bool mHasSurface;                   // 是否存在Surface
    bool mHaveEGLSurface;               // EGLSurface
    bool mHaveEGlContext;               // 释放资源

    int mSurfaceWidth;                  // 显示宽度
    int mSurfaceHeight;                 // 显示高度

    Texture *mVideoTexture;             // 视频纹理
    Renderer *mRenderer;                // 渲染器
};
GLESDevice::GLESDevice() {
    mWindow = NULL;
    eglSurface = EGL_NO_SURFACE;
    eglHelper = new EglHelper();

    mHaveEGLSurface = false;
    mHaveEGlContext = false;
    mHasSurface = false;

    mSurfaceWidth = 0;
    mSurfaceHeight = 0;

    mVideoTexture = (Texture *) malloc(sizeof(Texture));
    memset(mVideoTexture, 0, sizeof(Texture));
    mRenderer = NULL;
}

GLESDevice::~GLESDevice() {
    mMutex.lock();
    terminate();
    mMutex.unlock();
}

void GLESDevice::surfaceCreated(ANativeWindow *window) {
    mMutex.lock();
    if (mWindow != NULL) {
        ANativeWindow_release(mWindow);
        mWindow = NULL;
    }
    mWindow = window;
    mHasSurface = true;
    mMutex.unlock();
}

void GLESDevice::surfaceChanged(int width, int height) {
    mMutex.lock();
    mSurfaceWidth = width;
    mSurfaceHeight = height;
    mMutex.unlock();
}

void GLESDevice::surfaceDestroyed() {
    mMutex.lock();
    mHasSurface = false;
    mMutex.unlock();
}

void GLESDevice::terminate() {
    terminate(true);
}

void GLESDevice::terminate(bool releaseContext) {
    if (eglSurface != EGL_NO_SURFACE) {
        eglHelper->destroySurface(eglSurface);
        eglSurface = EGL_NO_SURFACE;
        mHaveEGLSurface = false;
    }
    if (eglHelper->getEglContext() != EGL_NO_CONTEXT && releaseContext) {
        eglHelper->release();
        mHaveEGlContext = false;
    }
}

void GLESDevice::onInitTexture(int width, int height, TextureFormat format, BlendMode blendMode) {
    mMutex.lock();

    // 创建EGLContext
    if (!mHaveEGlContext) {
        mHaveEGlContext = eglHelper->init(NULL, FLAG_TRY_GLES3);
        ALOGD("mHaveEGlContext = %d", mHaveEGlContext);
    }

    if (!mHaveEGlContext) {
        return;
    }
    // 创建/释放EGLSurface
    if (eglSurface == EGL_NO_SURFACE && mWindow != NULL) {
        if (mHasSurface && !mHaveEGLSurface) {
            eglSurface = eglHelper->createSurface(mWindow);
            if (eglSurface != EGL_NO_SURFACE) {
                mHaveEGLSurface = true;
                ALOGD("mHaveEGLSurface = %d", mHaveEGLSurface);
            }
        }
    } else if (eglSurface != EGL_NO_SURFACE && mHaveEGLSurface) {
        // 处于SurfaceDestroyed状态，释放EGLSurface
        if (!mHasSurface) {
            terminate(false);
        }
    }
    mVideoTexture->width = width;
    mVideoTexture->height = height;
    mVideoTexture->format = format;
    mVideoTexture->blendMode = blendMode;
    mVideoTexture->direction = FLIP_NONE;
    if (mRenderer == NULL) {
        if (format == FMT_YUV420P) {
            mRenderer = new YUV420PRenderer();
        } else if (format == FMT_ARGB) {
            mRenderer = new BGRARenderer();
        } else {
            mRenderer = NULL;
        }
        if (mRenderer != NULL) {
            eglHelper->makeCurrent(eglSurface);
            mRenderer->onInit(mVideoTexture);
            eglHelper->swapBuffers(eglSurface);
        }
    }
    mMutex.unlock();
}

int GLESDevice::onUpdateYUV(uint8_t *yData, int yPitch, uint8_t *uData, int uPitch, uint8_t *vData,
                            int vPitch) {
    if (!mHaveEGlContext) {
        return -1;
    }
    mMutex.lock();
    mVideoTexture->pitches[0] = yPitch;
    mVideoTexture->pitches[1] = uPitch;
    mVideoTexture->pitches[2] = vPitch;
    mVideoTexture->pixels[0] = yData;
    mVideoTexture->pixels[1] = uData;
    mVideoTexture->pixels[2] = vData;
    if (mRenderer != NULL && eglSurface != EGL_NO_SURFACE) {
        eglHelper->makeCurrent(eglSurface);
        mRenderer->uploadTexture(mVideoTexture);
    }
    mMutex.unlock();
    return 0;
}

int GLESDevice::onUpdateARGB(uint8_t *rgba, int pitch) {
    if (!mHaveEGlContext) {
        return -1;
    }
    mMutex.lock();
    mVideoTexture->pitches[0] = pitch;
    mVideoTexture->pixels[0] = rgba;
    if (mRenderer != NULL && eglSurface != EGL_NO_SURFACE) {
        eglHelper->makeCurrent(eglSurface);
        mRenderer->uploadTexture(mVideoTexture);
    }
    mMutex.unlock();
    return 0;
}

int GLESDevice::onRequestRender(FlipDirection direction) {
    if (!mHaveEGlContext) {
        return -1;
    }
    mMutex.lock();
    mVideoTexture->direction = direction;
    if (mRenderer != NULL && eglSurface != EGL_NO_SURFACE) {
        eglHelper->makeCurrent(eglSurface);
        mRenderer->renderTexture(mVideoTexture);
        eglHelper->swapBuffers(eglSurface);
    }
    mMutex.unlock();
    return 0;
}
```
我们在代码里面根据传递进来的格式创建不同的Renderer进行渲染视频画面。视频帧的数据存放在Texture 结构体中，并传递给Renderer对象进行处理。Renderer又分成BGRARenderer 和 YUV420PRenderer对象：
BGRARenderer:
```
BGRARenderer::BGRARenderer() {
    programHandle = 0;
    positionHandle = 0;
    texCoordHandle = 0;
    mvpMatrixHandle = 0;
    for (int i = 0; i < GLES_MAX_PLANE; ++i) {
        textureHandle[i] = 0;
        textures[i] = 0;
    }
    reset();
}

BGRARenderer::~BGRARenderer() {

}

void BGRARenderer::reset() {
    resetVertices();
    resetTexVertices();
    mInited = false;
}

void BGRARenderer::resetVertices() {
    vertices[0] = -1.0f;
    vertices[1] = -1.0f;
    vertices[2] =  1.0f;
    vertices[3] = -1.0f;
    vertices[4] = -1.0f;
    vertices[5] =  1.0f;
    vertices[6] =  1.0f;
    vertices[7] =  1.0f;
}

void BGRARenderer::resetTexVertices() {
    texVetrices[0] = 0.0f;
    texVetrices[1] = 1.0f;
    texVetrices[2] = 1.0f;
    texVetrices[3] = 1.0f;
    texVetrices[4] = 0.0f;
    texVetrices[5] = 0.0f;
    texVetrices[6] = 1.0f;
    texVetrices[7] = 0.0f;
}

int BGRARenderer::onInit(Texture *texture) {
    // 判断是否已经初始化
    if (mInited && programHandle != 0) {
        return 0;
    }
    programHandle = GLUtils::createProgram(GetDefaultVertexShader(), GetFragmentShader_BGRA());
    GLUtils::checkGLError("createProgram");
    positionHandle = glGetAttribLocation(programHandle, "aPosition");
    texCoordHandle = glGetAttribLocation(programHandle, "aTexCoord");
    textureHandle[0] = glGetUniformLocation(programHandle, "inputTexture");

    glPixelStorei(GL_UNPACK_ALIGNMENT, 1);
    glUseProgram(programHandle);

    if (textures[0] == 0) {
        glGenTextures(1, textures);
    }
    for (int i = 0; i < 1; ++i) {
        glActiveTexture(GL_TEXTURE0 + i);
        glBindTexture(GL_TEXTURE_2D, textures[i]);

        glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MAG_FILTER, GL_LINEAR);
        glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MIN_FILTER, GL_LINEAR);
        glTexParameterf(GL_TEXTURE_2D, GL_TEXTURE_WRAP_S, GL_CLAMP_TO_EDGE);
        glTexParameterf(GL_TEXTURE_2D, GL_TEXTURE_WRAP_T, GL_CLAMP_TO_EDGE);
    }

    mInited = true;
    return 0;
}

GLboolean BGRARenderer::uploadTexture(Texture *texture) {
    if (!texture || programHandle == 0) {
        return GL_FALSE;
    }
    // 需要设置4字节对齐
    glPixelStorei(GL_UNPACK_ALIGNMENT, 1);
    glUseProgram(programHandle);
    glClear(GL_COLOR_BUFFER_BIT);

    // 更新纹理数据
    glActiveTexture(GL_TEXTURE0);
    glBindTexture(GL_TEXTURE_2D, textures[0]);
    glTexImage2D(GL_TEXTURE_2D,
                 0,
                 GL_RGBA,                 // 对于YUV来说，数据格式是GL_LUMINANCE亮度值，而对于BGRA来说，这个则是颜色通道值
                 texture->pitches[0] / 4, // pixels中存放的数据是BGRABGRABGRA方式排列的，这里除4是为了求出对齐后的宽度，也就是每个颜色通道的数值
                 texture->height,
                 0,
                 GL_RGBA,
                 GL_UNSIGNED_BYTE,
                 texture->pixels[0]);
    glUniform1i(textureHandle[0], 0);

    return GL_TRUE;
}

GLboolean BGRARenderer::renderTexture(Texture *texture) {
    if (!texture || programHandle == 0) {
        return GL_FALSE;
    }

    // 绑定顶点坐标
    glVertexAttribPointer(positionHandle, 2, GL_FLOAT, GL_FALSE, 0, vertices);
    glEnableVertexAttribArray(positionHandle);
    // 绑定纹理坐标
    glVertexAttribPointer(texCoordHandle, 2, GL_FLOAT, GL_FALSE, 0, texVetrices);
    glEnableVertexAttribArray(texCoordHandle);

    // 绘制
    glDrawArrays(GL_TRIANGLE_STRIP, 0, 4);
    // 解绑
    glDisableVertexAttribArray(texCoordHandle);
    glDisableVertexAttribArray(positionHandle);
    glUseProgram(0);
    return GL_TRUE;
}
```
YUV420PRenderer:
```
YUV420PRenderer::YUV420PRenderer() {
    programHandle = 0;
    positionHandle = 0;
    texCoordHandle = 0;
    mvpMatrixHandle = 0;
    for (int i = 0; i < GLES_MAX_PLANE; ++i) {
        textureHandle[i] = 0;
        textures[i] = 0;
    }
    reset();
}

YUV420PRenderer::~YUV420PRenderer() {

}

void YUV420PRenderer::reset() {
    resetVertices();
    resetTexVertices();
    mInited = false;
}

void YUV420PRenderer::resetVertices() {
    vertices[0] = -1.0f;
    vertices[1] = -1.0f;
    vertices[2] =  1.0f;
    vertices[3] = -1.0f;
    vertices[4] = -1.0f;
    vertices[5] =  1.0f;
    vertices[6] =  1.0f;
    vertices[7] =  1.0f;
}

void YUV420PRenderer::resetTexVertices() {
    texVetrices[0] = 0.0f;
    texVetrices[1] = 1.0f;
    texVetrices[2] = 1.0f;
    texVetrices[3] = 1.0f;
    texVetrices[4] = 0.0f;
    texVetrices[5] = 0.0f;
    texVetrices[6] = 1.0f;
    texVetrices[7] = 0.0f;
}


int YUV420PRenderer::onInit(Texture *texture) {
    // 判断是否已经初始化
    if (mInited && programHandle != 0) {
        return 0;
    }
    programHandle = GLUtils::createProgram(GetDefaultVertexShader(), GetFragmentShader_YUV420P());
    GLUtils::checkGLError("createProgram");
    positionHandle = glGetAttribLocation(programHandle, "aPosition");
    texCoordHandle = glGetAttribLocation(programHandle, "aTexCoord");
    textureHandle[0] = glGetUniformLocation(programHandle, "inputTextureY");
    textureHandle[1] = glGetUniformLocation(programHandle, "inputTextureU");
    textureHandle[2] = glGetUniformLocation(programHandle, "inputTextureV");

    glPixelStorei(GL_UNPACK_ALIGNMENT, 1);
    glUseProgram(programHandle);

    if (textures[0] == 0) {
        glGenTextures(3, textures);
    }
    for (int i = 0; i < 3; ++i) {
        glActiveTexture(GL_TEXTURE0 + i);
        glBindTexture(GL_TEXTURE_2D, textures[i]);

        glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MAG_FILTER, GL_LINEAR);
        glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MIN_FILTER, GL_LINEAR);
        glTexParameterf(GL_TEXTURE_2D, GL_TEXTURE_WRAP_S, GL_CLAMP_TO_EDGE);
        glTexParameterf(GL_TEXTURE_2D, GL_TEXTURE_WRAP_T, GL_CLAMP_TO_EDGE);
        glUniform1i(textureHandle[i], i);
    }

    mInited = true;
    return 0;
}

GLboolean YUV420PRenderer::uploadTexture(Texture *texture) {
    if (!texture || programHandle == 0) {
        return GL_FALSE;
    }
    // 需要设置4字节对齐
    glPixelStorei(GL_UNPACK_ALIGNMENT, 1);
    glUseProgram(programHandle);
    glClear(GL_COLOR_BUFFER_BIT);

    // 更新纹理数据
    const GLsizei heights[3] = { texture->height, texture->height / 2, texture->height / 2};
    for (int i = 0; i < 3; ++i) {
        glActiveTexture(GL_TEXTURE0 + i);
        glBindTexture(GL_TEXTURE_2D, textures[i]);
        glTexImage2D(GL_TEXTURE_2D,
                     0,
                     GL_LUMINANCE,
                     texture->pitches[i],
                     heights[i],
                     0,
                     GL_LUMINANCE,
                     GL_UNSIGNED_BYTE,
                     texture->pixels[i]);
        glUniform1i(textureHandle[i], i);
    }

    return 0;
}

GLboolean YUV420PRenderer::renderTexture(Texture *texture) {
    // 绑定顶点坐标
    glVertexAttribPointer(positionHandle, 2, GL_FLOAT, GL_FALSE, 0, vertices);
    glEnableVertexAttribArray(positionHandle);
    // 绑定纹理坐标
    glVertexAttribPointer(texCoordHandle, 2, GL_FLOAT, GL_FALSE, 0, texVetrices);
    glEnableVertexAttribArray(texCoordHandle);
    // 绘制
    glDrawArrays(GL_TRIANGLE_STRIP, 0, 4);
    // 解绑
    glDisableVertexAttribArray(texCoordHandle);
    glDisableVertexAttribArray(positionHandle);
    glUseProgram(0);
    return GL_TRUE;
}
```
GLESDevice由于使用了ANativeWindow来构建EGLSurface，对应Java层的Surface，此时会有surfaceCreated、surfaceChanged、surfaceDestroyed三个方法来改变状态的。
为了方便使用，我们打算将视频输出设备对象，从外面传进来 ，这样我们就可以根据不同的OS环境进行更换输出设备了，比如你想接入iOS设备的时候，播放器的逻辑基本不需要更改，只需要将音频输出设备、视频输出设备进行更换即可。Android环境下的使用方式如下：
```
extern "C"
JNIEXPORT void JNICALL
Java_com_cgfay_media_MediaPlayer_AVMediaPlayer_nativeSetup(JNIEnv *env, jobject instance) {
    if (mediaPlayer == NULL) {
        mediaPlayer = new MediaPlayer();
    }
    mediaPlayer->setPlayerCallback(new JniMediaPlayerCallback(javaVM, env, &instance));
    videoDevice = new GLESDevice();
}

extern "C"
JNIEXPORT void JNICALL
Java_com_cgfay_media_MediaPlayer_AVMediaPlayer_nativeRelease(JNIEnv *env, jobject instance) {

    if (mediaPlayer != NULL) {
        mediaPlayer->stop();
        delete mediaPlayer;
        mediaPlayer = NULL;
    }

    if (videoDevice != NULL) {
        delete videoDevice;
        videoDevice = NULL;
    }
}

extern "C"
JNIEXPORT void JNICALL
Java_com_cgfay_media_MediaPlayer_AVMediaPlayer_nativeSurfaceCreated(JNIEnv *env,
                                                                        jobject instance,
                                                                        jobject surface) {
    if (videoDevice) {
        ANativeWindow *window = NULL;
        if (surface != NULL) {
            window = ANativeWindow_fromSurface(env, surface);
        }
        videoDevice->surfaceCreated(window);
    }
    if (mediaPlayer) {
        mediaPlayer->setVideoDevice(videoDevice);
    }
}

extern "C"
JNIEXPORT void JNICALL
Java_com_cgfay_media_MediaPlayer_AVMediaPlayer_nativeSurfaceChanged(JNIEnv *env, jobject instance,
                                                                    jint width, jint height) {
    if (videoDevice) {
        videoDevice->surfaceChanged(width, height);
    }
}

extern "C"
JNIEXPORT void JNICALL
Java_com_cgfay_media_MediaPlayer_AVMediaPlayer_nativeSurfaceDestroyed(JNIEnv *env, jobject instance) {

    if (videoDevice) {
        videoDevice->surfaceDestroyed();
    }
}
```

至此。整个播放器的整体逻辑就已经讲解完了，我们这样就从头开始写了一个播放器，虽然播放器还有不少的BUG，但播放器的大致框架基本上搭建好，接下来就是改BUG，接入硬解码、接入不同输出设备、姐夫AVFilter等功能。由于本系列文章只是讲解播放器开发的基础，因此，这里不会深入讲解这些功能，后续有时间我再慢慢将坑填了。
完整代码请参考本人的播放器项目：[CainPlayer](https://github.com/CainKernel/CainPlayer)
