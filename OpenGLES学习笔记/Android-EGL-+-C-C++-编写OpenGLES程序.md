本人在编写基于FFmpeg的播放器时，需要将解码后的视频帧数据upload到GPU进行渲染输出，方便给视频添加滤镜之类的。输出部分有两种方案，一种是使用GLSurfaceView，就是将Native解码得到的数据回到到Java层进行渲染。第二种是使用EGL + ANativeWindow 直接在Native层利用GPU进行渲染。第一种方案需要在Native层与Java层之间不断进行数据交换，这方的方式其实并不太好，如果遇到多线程处理，由于Java层与Native层之间各自线程空间的不同，个人不太推荐这样的方式。下面来讲解一下如何在Native层使用EGL进行渲染。如果不了解什么是EGL的话，建议先看看本人的文章[EGL简介以及窗口初始化](https://www.jianshu.com/p/1e49ae3cf3ac)了解一下。另外，大家有兴趣的话，可以看下GLSurfaceView的源码，GLSurfaceView也是通过开辟一个Looper线程GLThread来封装EGL来实现的。我们接下来实现一个Native层封装EGL + ANativeWindow的项目，项目的Github地址如下：
[EGLNativeRender](https://github.com/CainKernel/EGLNativeRender)

###### EGL的封装
由于EGL的API使用起来比较复杂，为了方便使用，我们需要将EGL做一层封装。关于EGL的API使用方式，可以参考[Grafika](https://github.com/google/grafika)项目中的EglCore.java、EglSurfaceBase.java、WindowSurface.java和OffscreenSurface.java，我的CainCamera 项目中也用到了Grafika的EGL封装类，使用方便。由于Grafika的EglCore是用Java层封装的，我们现在需要在Native层实现类似的EglCore、EglSurfaceBase、WindowSurface和OffscreenSurface类。废话不多数，直接上源码：
EglCore.h:
```
#ifndef CAINCAMERA_EGLCORE_H
#define CAINCAMERA_EGLCORE_H

#include <android/native_window.h>
#include <EGL/egl.h>
#include <EGL/eglext.h>
#include <EGL/eglplatform.h>

/**
 * Constructor flag: surface must be recordable.  This discourages EGL from using a
 * pixel format that cannot be converted efficiently to something usable by the video
 * encoder.
 */
#define FLAG_RECORDABLE 0x01

/**
 * Constructor flag: ask for GLES3, fall back to GLES2 if not available.  Without this
 * flag, GLES2 is used.
 */
#define FLAG_TRY_GLES3 002

// Android-specific extension
#define EGL_RECORDABLE_ANDROID 0x3142

typedef EGLBoolean (EGLAPIENTRYP EGL_PRESENTATION_TIME_ANDROIDPROC)(EGLDisplay display, EGLSurface surface, khronos_stime_nanoseconds_t time);

class EglCore {

private:
    EGLDisplay mEGLDisplay = EGL_NO_DISPLAY;
    EGLConfig  mEGLConfig = NULL;
    EGLContext mEGLContext = EGL_NO_CONTEXT;

    // 设置时间戳方法
    EGL_PRESENTATION_TIME_ANDROIDPROC eglPresentationTimeANDROID = NULL;

    int mGlVersion = -1;
    // 查找合适的EGLConfig
    EGLConfig getConfig(int flags, int version);

public:
    EglCore();
    ~EglCore();
    EglCore(EGLContext sharedContext, int flags);
    bool init(EGLContext sharedContext, int flags);
    // 释放资源
    void release();
    // 获取EglContext
    EGLContext getEGLContext();
    // 销毁Surface
    void releaseSurface(EGLSurface eglSurface);
    //  创建EGLSurface
    EGLSurface createWindowSurface(ANativeWindow *surface);
    // 创建离屏EGLSurface
    EGLSurface createOffscreenSurface(int width, int height);
    // 切换到当前上下文
    void makeCurrent(EGLSurface eglSurface);
    // 切换到某个上下文
    void makeCurrent(EGLSurface drawSurface, EGLSurface readSurface);
    // 没有上下文
    void makeNothingCurrent();
    // 交换显示
    bool swapBuffers(EGLSurface eglSurface);
    // 设置pts
    void setPresentationTime(EGLSurface eglSurface, long nsecs);
    // 判断是否属于当前上下文
    bool isCurrent(EGLSurface eglSurface);
    // 执行查询
    int querySurface(EGLSurface eglSurface, int what);
    // 查询字符串
    const char *queryString(int what);
    // 获取当前的GLES 版本号
    int getGlVersion();
    // 检查是否出错
    void checkEglError(const char *msg);
};


#endif //CAINCAMERA_EGLCORE_H
```
EglCore.cpp:
```
#include "../common/native_log.h"
#include "EglCore.h"
#include <assert.h>

EglCore::EglCore() {
    init(NULL, 0);
}


EglCore::~EglCore() {
    release();
}

/**
 * 构造方法
 * @param sharedContext
 * @param flags
 */
EglCore::EglCore(EGLContext sharedContext, int flags) {
    init(sharedContext, flags);
}

/**
 * 初始化
 * @param sharedContext
 * @param flags
 * @return
 */
bool EglCore::init(EGLContext sharedContext, int flags) {
    assert(mEGLDisplay == EGL_NO_DISPLAY);
    if (mEGLDisplay != EGL_NO_DISPLAY) {
        ALOGE("EGL already set up");
        return false;
    }
    if (sharedContext == NULL) {
        sharedContext = EGL_NO_CONTEXT;
    }

    mEGLDisplay = eglGetDisplay(EGL_DEFAULT_DISPLAY);
    assert(mEGLDisplay != EGL_NO_DISPLAY);
    if (mEGLDisplay == EGL_NO_DISPLAY) {
        ALOGE("unable to get EGL14 display.\n");
        return false;
    }

    if (!eglInitialize(mEGLDisplay, 0, 0)) {
        mEGLDisplay = EGL_NO_DISPLAY;
        ALOGE("unable to initialize EGL14");
        return false;
    }

    // 尝试使用GLES3
    if ((flags & FLAG_TRY_GLES3) != 0) {
        EGLConfig config = getConfig(flags, 3);
        if (config != NULL) {
            int attrib3_list[] = {
                    EGL_CONTEXT_CLIENT_VERSION, 3,
                    EGL_NONE
            };
            EGLContext context = eglCreateContext(mEGLDisplay, config,
                                                  sharedContext, attrib3_list);
            checkEglError("eglCreateContext");
            if (eglGetError() == EGL_SUCCESS) {
                mEGLConfig = config;
                mEGLContext = context;
                mGlVersion = 3;
            }
        }
    }
    // 如果GLES3没有获取到，则尝试使用GLES2
    if (mEGLContext == EGL_NO_CONTEXT) {
        EGLConfig config = getConfig(flags, 2);
        assert(config != NULL);
        int attrib2_list[] = {
                EGL_CONTEXT_CLIENT_VERSION, 2,
                EGL_NONE
        };
        EGLContext context = eglCreateContext(mEGLDisplay, config,
                                              sharedContext, attrib2_list);
        checkEglError("eglCreateContext");
        if (eglGetError() == EGL_SUCCESS) {
            mEGLConfig = config;
            mEGLContext = context;
            mGlVersion = 2;
        }
    }

    // 获取eglPresentationTimeANDROID方法的地址
    eglPresentationTimeANDROID = (EGL_PRESENTATION_TIME_ANDROIDPROC)
            eglGetProcAddress("eglPresentationTimeANDROID");
    if (!eglPresentationTimeANDROID) {
        ALOGE("eglPresentationTimeANDROID is not available!");
    }

    int values[1] = {0};
    eglQueryContext(mEGLDisplay, mEGLContext, EGL_CONTEXT_CLIENT_VERSION, values);
    ALOGD("EGLContext created, client version %d", values[0]);

    return true;
}


/**
 * 获取合适的EGLConfig
 * @param flags
 * @param version
 * @return
 */
EGLConfig EglCore::getConfig(int flags, int version) {
    int renderableType = EGL_OPENGL_ES2_BIT;
    if (version >= 3) {
        renderableType |= EGL_OPENGL_ES3_BIT_KHR;
    }
    int attribList[] = {
            EGL_RED_SIZE, 8,
            EGL_GREEN_SIZE, 8,
            EGL_BLUE_SIZE, 8,
            EGL_ALPHA_SIZE, 8,
            //EGL_DEPTH_SIZE, 16,
            //EGL_STENCIL_SIZE, 8,
            EGL_RENDERABLE_TYPE, renderableType,
            EGL_NONE, 0,      // placeholder for recordable [@-3]
            EGL_NONE
    };
    int length = sizeof(attribList) / sizeof(attribList[0]);
    if ((flags & FLAG_RECORDABLE) != 0) {
        attribList[length - 3] = EGL_RECORDABLE_ANDROID;
        attribList[length - 2] = 1;
    }
    EGLConfig configs = NULL;
    int numConfigs;
    if (!eglChooseConfig(mEGLDisplay, attribList, &configs, 1, &numConfigs)) {
        ALOGW("unable to find RGB8888 / %d  EGLConfig", version);
        return NULL;
    }
    return configs;
}

/**
 * 释放资源
 */
void EglCore::release() {
    if (mEGLDisplay != EGL_NO_DISPLAY) {
        eglMakeCurrent(mEGLDisplay, EGL_NO_SURFACE, EGL_NO_SURFACE, EGL_NO_CONTEXT);
        eglDestroyContext(mEGLDisplay, mEGLContext);
        eglReleaseThread();
        eglTerminate(mEGLDisplay);
    }

    mEGLDisplay = EGL_NO_DISPLAY;
    mEGLContext = EGL_NO_CONTEXT;
    mEGLConfig = NULL;
}

/**
 * 获取EGLContext
 * @return
 */
EGLContext EglCore::getEGLContext() {
    return mEGLContext;
}

/**
 * 销毁EGLSurface
 * @param eglSurface
 */
void EglCore::releaseSurface(EGLSurface eglSurface) {
    eglDestroySurface(mEGLDisplay, eglSurface);
}

/**
 * 创建EGLSurface
 * @param surface
 * @return
 */
EGLSurface EglCore::createWindowSurface(ANativeWindow *surface) {
    assert(surface != NULL);
    if (surface == NULL) {
        ALOGE("ANativeWindow is NULL!");
        return NULL;
    }
    int surfaceAttribs[] = {
            EGL_NONE
    };
    ALOGD("eglCreateWindowSurface start");
    EGLSurface eglSurface = eglCreateWindowSurface(mEGLDisplay, mEGLConfig, surface, surfaceAttribs);
    checkEglError("eglCreateWindowSurface");
    assert(eglSurface != NULL);
    if (eglSurface == NULL) {
        ALOGE("EGLSurface is NULL!");
        return NULL;
    }
    return eglSurface;
}

/**
 * 创建离屏渲染的EGLSurface
 * @param width
 * @param height
 * @return
 */
EGLSurface EglCore::createOffscreenSurface(int width, int height) {
    int surfaceAttribs[] = {
            EGL_WIDTH, width,
            EGL_HEIGHT, height,
            EGL_NONE
    };
    EGLSurface eglSurface = eglCreatePbufferSurface(mEGLDisplay, mEGLConfig, surfaceAttribs);
    assert(eglSurface != NULL);
    if (eglSurface == NULL) {
        ALOGE("Surface was null");
        return NULL;
    }
    return eglSurface;
}

/**
 * 切换到当前的上下文
 * @param eglSurface
 */
void EglCore::makeCurrent(EGLSurface eglSurface) {
    if (mEGLDisplay == EGL_NO_DISPLAY) {
        ALOGD("Note: makeCurrent w/o display.\n");
    }
    if (!eglMakeCurrent(mEGLDisplay, eglSurface, eglSurface, mEGLContext)) {
        // TODO 抛出异常
    }
}

/**
 * 切换到某个上下文
 * @param drawSurface
 * @param readSurface
 */
void EglCore::makeCurrent(EGLSurface drawSurface, EGLSurface readSurface) {
    if (mEGLDisplay == EGL_NO_DISPLAY) {
        ALOGD("Note: makeCurrent w/o display.\n");
    }
    if (!eglMakeCurrent(mEGLDisplay, drawSurface, readSurface, mEGLContext)) {
        // TODO 抛出异常
    }
}

/**
 *
 */
void EglCore::makeNothingCurrent() {
    if (!eglMakeCurrent(mEGLDisplay, EGL_NO_SURFACE, EGL_NO_SURFACE, EGL_NO_CONTEXT)) {
        // TODO 抛出异常
    }
}

/**
 * 交换显示
 * @param eglSurface
 * @return
 */
bool EglCore::swapBuffers(EGLSurface eglSurface) {
    return eglSwapBuffers(mEGLDisplay, eglSurface);
}

/**
 * 设置显示时间戳pts
 * @param eglSurface
 * @param nsecs
 */
void EglCore::setPresentationTime(EGLSurface eglSurface, long nsecs) {
    eglPresentationTimeANDROID(mEGLDisplay, eglSurface, nsecs);
}

/**
 * 是否处于当前上下文
 * @param eglSurface
 * @return
 */
bool EglCore::isCurrent(EGLSurface eglSurface) {
    return mEGLContext == eglGetCurrentContext() &&
            eglSurface == eglGetCurrentSurface(EGL_DRAW);
}

/**
 * 查询surface
 * @param eglSurface
 * @param what
 * @return
 */
int EglCore::querySurface(EGLSurface eglSurface, int what) {
    int value;
    eglQuerySurface(mEGLContext, eglSurface, what, &value);
    return value;
}

/**
 * 查询字符串
 * @param what
 * @return
 */
const char* EglCore::queryString(int what) {
    return eglQueryString(mEGLDisplay, what);
}

/**
 * 获取GLES版本号
 * @return
 */
int EglCore::getGlVersion() {
    return mGlVersion;
}

/**
 * 检查是否出错
 * @param msg
 */
void EglCore::checkEglError(const char *msg) {
    int error;
    if ((error = eglGetError()) != EGL_SUCCESS) {
        // TODO 抛出异常
        ALOGE("%s: EGL error: %x", msg, error);
    }
}
```
到这里，我们就封装好了EglCore核心，那么接下来我们需要对EGLSurface进行封装处理：
EglSurfaceBase.h:
```
#ifndef CAINCAMERA_EGLSURFACEBASE_H
#define CAINCAMERA_EGLSURFACEBASE_H

#include "EglCore.h"
#include "../common/native_log.h"

class EglSurfaceBase {

public:
    EglSurfaceBase(EglCore *eglCore);
    // 创建窗口Surface
    void createWindowSurface(ANativeWindow *nativeWindow);
    // 创建离屏Surface
    void createOffscreenSurface(int width, int height);
    // 获取宽度
    int getWidth();
    // 获取高度
    int getHeight();
    // 释放EGLSurface
    void releaseEglSurface();
    // 切换到当前上下文
    void makeCurrent();
    // 交换缓冲区，显示图像
    bool swapBuffers();
    // 设置显示时间戳
    void setPresentationTime(long nsecs);
    // 获取当前帧缓冲
    char *getCurrentFrame();

protected:
    EglCore *mEglCore;
    EGLSurface mEglSurface;
    int mWidth;
    int mHeight;

};


#endif //CAINCAMERA_EGLSURFACEBASE_H
```
EglSurfaceBase.cpp:
```
#include <assert.h>
#include <GLES2/gl2.h>
#include "EglSurfaceBase.h"


EglSurfaceBase::EglSurfaceBase(EglCore *eglCore) : mEglCore(eglCore) {
    mEglSurface = EGL_NO_SURFACE;
}

/**
 * 创建显示的Surface
 * @param nativeWindow
 */
void EglSurfaceBase::createWindowSurface(ANativeWindow *nativeWindow) {
    assert(mEglSurface == EGL_NO_SURFACE);
    if (mEglSurface != EGL_NO_SURFACE) {
        ALOGE("surface already created\n");
        return;
    }
    mEglSurface = mEglCore->createWindowSurface(nativeWindow);
}

/**
 * 创建离屏surface
 * @param width
 * @param height
 */
void EglSurfaceBase::createOffscreenSurface(int width, int height) {
    assert(mEglSurface == EGL_NO_SURFACE);
    if (mEglSurface != EGL_NO_SURFACE) {
        ALOGE("surface already created\n");
        return;
    }
    mEglSurface = mEglCore->createOffscreenSurface(width, height);
    mWidth = width;
    mHeight = height;
}

/**
 * 获取宽度
 * @return
 */
int EglSurfaceBase::getWidth() {
    if (mWidth < 0) {
        return mEglCore->querySurface(mEglSurface, EGL_WIDTH);
    } else {
        return mWidth;
    }
}

/**
 * 获取高度
 * @return
 */
int EglSurfaceBase::getHeight() {
    if (mHeight < 0) {
        return mEglCore->querySurface(mEglSurface, EGL_HEIGHT);
    } else {
        return mHeight;
    }
}

/**
 * 释放EGLSurface
 */
void EglSurfaceBase::releaseEglSurface() {
    mEglCore->releaseSurface(mEglSurface);
    mEglSurface = EGL_NO_SURFACE;
    mWidth = mHeight = -1;
}

/**
 * 切换到当前EGLContext
 */
void EglSurfaceBase::makeCurrent() {
    mEglCore->makeCurrent(mEglSurface);
}

/**
 * 交换到前台显示
 * @return
 */
bool EglSurfaceBase::swapBuffers() {
    bool result = mEglCore->swapBuffers(mEglSurface);
    if (!result) {
        ALOGD("WARNING: swapBuffers() failed");
    }
    return result;
}

/**
 * 设置当前时间戳
 * @param nsecs
 */
void EglSurfaceBase::setPresentationTime(long nsecs) {
    mEglCore->setPresentationTime(mEglSurface, nsecs);
}

/**
 * 获取当前像素
 * @return
 */
char* EglSurfaceBase::getCurrentFrame() {
    char *pixels = NULL;
    glReadPixels(0, 0, getWidth(), getHeight(), GL_RGBA, GL_UNSIGNED_BYTE, pixels);
    return pixels;
}
```
好了，我们封装好了EGLSurface的基类，接下来我们需要实现一个用于显示的WindowSurface 和 用于离屏渲染的OffscreenSurface，实现如下：
WindowSurface.h:
```
#ifndef CAINCAMERA_WINDOWSURFACE_H
#define CAINCAMERA_WINDOWSURFACE_H


#include "EglSurfaceBase.h"

class WindowSurface : public EglSurfaceBase {

public:
    WindowSurface(EglCore *eglCore, ANativeWindow *window, bool releaseSurface);
    WindowSurface(EglCore *eglCore, ANativeWindow *window);
    // 释放资源
    void release();
    // 重新创建
    void recreate(EglCore *eglCore);

private:
    ANativeWindow  *mSurface;
    bool mReleaseSurface;
};
#endif //CAINCAMERA_WINDOWSURFACE_H
```
WindowSurface.cpp:
```
#include <assert.h>
#include "WindowSurface.h"

WindowSurface::WindowSurface(EglCore *eglCore, ANativeWindow *window, bool releaseSurface)
        : EglSurfaceBase(eglCore) {
    mSurface = window;
    createWindowSurface(mSurface);
    mReleaseSurface = releaseSurface;
}

WindowSurface::WindowSurface(EglCore *eglCore, ANativeWindow *window)
        : EglSurfaceBase(eglCore) {
    createWindowSurface(window);
    mSurface = window;
}

void WindowSurface::release() {
    releaseEglSurface();
    if (mSurface != NULL) {
        ANativeWindow_release(mSurface);
        mSurface = NULL;
    }

}

void WindowSurface::recreate(EglCore *eglCore) {
    assert(mSurface != NULL);
    if (mSurface == NULL) {
        ALOGE("not yet implemented ANativeWindow");
        return;
    }
    mEglCore = eglCore;
    createWindowSurface(mSurface);
}
```
OffscreenSurface.h:
```
#ifndef CAINCAMERA_OFFSCREENSURFACE_H
#define CAINCAMERA_OFFSCREENSURFACE_H


#include "EglSurfaceBase.h"

class OffscreenSurface : public EglSurfaceBase {
public:
    OffscreenSurface(EglCore *eglCore, int width, int height);
    void release();
};

#endif //CAINCAMERA_OFFSCREENSURFACE_H
```
OffscreenSurface.cpp:
```
#include "OffscreenSurface.h"

OffscreenSurface::OffscreenSurface(EglCore *eglCore, int width, int height)
        : EglSurfaceBase(eglCore) {
    createOffscreenSurface(width, height);
}

void OffscreenSurface::release() {
    releaseEglSurface();
}
```
到这里，我们就在Native实现了跟Grafika 的Java层一样的EGL封装。在封装好EGL 之后，我们需要一个GLRender，跟Java 层 GLSurfaceView 差不多。废话不多说，直接上源码：
GLRender.h:
```
#ifndef EGLNATIVESAMPLE_GLRENDER_H
#define EGLNATIVESAMPLE_GLRENDER_H


#include <android/native_window.h>
#include "../caingles/EglCore.h"
#include "../caingles/WindowSurface.h"
#include "Triangle.h"

class GLRender {
public:
    GLRender();

    virtual ~GLRender();

    void surfaceCreated(ANativeWindow *window);

    void surfaceChanged(int width, int height);

    void surfaceDestroyed(void);

private:
    EglCore *mEglCore;
    WindowSurface *mWindowSurface;
    Triangle *mTriangle;
};
#endif //EGLNATIVESAMPLE_GLRENDER_H
```
GLRender.cpp:
```
#include "GLRender.h"
#include "../caingles/GlUtils.h"

#include <GLES2/gl2.h>
#include <GLES2/gl2ext.h>
#include <GLES2/gl2platform.h>
#include <assert.h>

GLRender::GLRender() {
    mEglCore = NULL;
    mWindowSurface = NULL;
}

GLRender::~GLRender() {
    if (mEglCore) {
        mEglCore->release();
        delete mEglCore;
        mEglCore = NULL;
    }
}

void GLRender::surfaceCreated(ANativeWindow *window) {

    if (mEglCore == NULL) {
        mEglCore = new EglCore(NULL, FLAG_RECORDABLE);
    }
    mWindowSurface = new WindowSurface(mEglCore, window, false);
    assert(mWindowSurface != NULL && mEglCore != NULL);
    mWindowSurface->makeCurrent();
    mTriangle = new Triangle();
    mTriangle->init();
}

void GLRender::surfaceChanged(int width, int height) {
    mWindowSurface->makeCurrent();
    mTriangle->onDraw(width, height);
    mWindowSurface->swapBuffers();
}

void GLRender::surfaceDestroyed() {
    if (mTriangle) {
        mTriangle->destroy();
        delete mTriangle;
        mTriangle = NULL;
    }
    if (mWindowSurface) {
        mWindowSurface->release();
        delete mWindowSurface;
        mWindowSurface = NULL;
    }
    if (mEglCore) {
        mEglCore->release();
        delete mEglCore;
        mEglCore = NULL;
    }
}
```
其中Triangle是使用OpenGLES绘制三角形的类，实现如下：
Triangle.h:
```
#ifndef EGLNATIVERENDER_TRIANGLE_H
#define EGLNATIVERENDER_TRIANGLE_H

#include "../caingles/GlUtils.h"
#include "../caingles/GlShaders.h"
#include "../common/native_log.h"

class Triangle {
public:
    Triangle();

    virtual ~Triangle();

    int init(void);

    void onDraw(int width, int height);

    void destroy();

private:
    int programHandle;
};

#endif //EGLNATIVERENDER_TRIANGLE_H
```
Triangle.cpp:
```
#include "Triangle.h"

const GLint COORDS_PER_VERTEX = 3;
const GLint vertexStride = COORDS_PER_VERTEX * 4;

Triangle::Triangle() {}

Triangle::~Triangle() {

}

int Triangle::init() {
    char vertexShader[] =
            "#version 300 es\n"
                    "layout(location = 0) in vec4 a_position;\n"
                    "layout(location = 1) in vec4 a_color;\n"
                    "out vec4 v_color;"
                    "void main()\n"
                    "{\n"
                    "   gl_Position = a_position;\n"
                    "   v_color = a_color;\n"
                    "}\n";

    char fragmentShader[] =
            "#version 300 es\n"
                    "precision mediump float;\n"
                    "in vec4 v_color;\n"
                    "out vec4 fragColor;\n"
                    "void main()\n"
                    "{\n"
//                    "   fragColor = vec4 ( 1.0, 0.0, 0.0, 1.0 );\n"
                    "   fragColor = v_color;\n"
                    "}\n";
    programHandle = createProgram(vertexShader, fragmentShader);
    if (programHandle <= 0) {
        return -1;
    }
    glClearColor(0.0f, 0.0f, 0.0f, 0.0f);
    return 0;
}


void Triangle::onDraw(int width, int height) {
    GLfloat vertices[] = {
            0.0f,  0.5f, 0.0f,
            -0.5f, -0.5f, 0.0f,
            0.5f, -0.5f, 0.0f
    };

    GLfloat color[] = {
            1.0f, 0.0f, 0.0f, 1.0f
    };

    GLint vertexCount = sizeof(vertices) / (sizeof(vertices[0]) * COORDS_PER_VERTEX);

    glViewport(0, 0, width, height);

    glClear(GL_COLOR_BUFFER_BIT);

    glUseProgram(programHandle);

    GLint positionHandle = glGetAttribLocation(programHandle, "a_position");
    glVertexAttribPointer(positionHandle, COORDS_PER_VERTEX, GL_FLOAT, GL_FALSE, vertexStride, vertices);
    glEnableVertexAttribArray(positionHandle);
    glVertexAttrib4fv(1, color);

    glDrawArrays(GL_TRIANGLES, 0, vertexCount);

    glDisableVertexAttribArray(positionHandle);
}

void Triangle::destroy() {
    if (programHandle > 0) {
        glDeleteProgram(programHandle);
    }
    programHandle = -1;
}
```

###### Looper线程的实现
由于在使用OpenGLES时需要单独一个线程空间，可参考GLSurfaceview的源码，GLSurfaceView就是通过开辟一个Looper 线程GLThread来调用EGL的API实现的。因此，我们也要创建一个Looper 线程，但Native层并没有提供跟Java层那么好用的Looper，只提供了ALooper，但个人觉得ALooper 使用并不是很友好。Looper的底层是基于epoll来实现的。我们这里不用epoll，而是采用信号量来搭建Looper，可以参考Google官方的android-ndk项目中的[native-codec](https://github.com/googlesamples/android-ndk/tree/master/native-codec)，native-codec项目就是用了信号量来创建Looper，并调用AMediaCodec渲染的。我们也有样学样，直接封装一个比较实用的Looper线程。废话不多数，直接上代码：
Looper.h:
```
#ifndef CAINPLAYER_LOOPER_H
#define CAINPLAYER_LOOPER_H

#include <pthread.h>
#include <sys/types.h>
#include <semaphore.h>

struct LooperMessage {
    int what;
    int arg1;
    int arg2;
    void *obj;
    LooperMessage *next;
    bool quit;
};

class Looper {

public:
    Looper();
    Looper&operator=(const Looper& ) = delete;
    Looper(Looper&) = delete;
    virtual ~Looper();

    // 发送消息
    void postMessage(int what, bool flush = false);
    void postMessage(int what, void *obj, bool flush = false);
    void postMessage(int what, int arg1, int arg2, bool flush = false);
    void postMessage(int what, int arg1, int arg2, void *obj, bool flush = false);

    // 退出Looper循环
    void quit();

    // 处理消息
    virtual void handleMessage(LooperMessage *msg);

private:
    // 添加消息
    void addMessage(LooperMessage *msg, bool flush);

    // 消息线程句柄
    static void *trampoline(void *p);

    // 循环体
    void loop(void);

    LooperMessage *head;
    pthread_t worker;
    sem_t headwriteprotect;
    sem_t headdataavailable;
    bool running;

};

#endif //CAINPLAYER_LOOPER_H
```
Looper.cpp:
```
#include "Looper.h"

#include <jni.h>
#include <pthread.h>
#include <stdio.h>
#include <string.h>
#include <unistd.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <semaphore.h>
#include "native_log.h"

struct LooperMessage;
typedef struct LooperMessage LooperMessage;



/**
 * 线程执行句柄
 * @param p
 * @return
 */
void* Looper::trampoline(void* p) {
    ((Looper*)p)->loop();
    return NULL;
}


Looper::Looper() {
    head = NULL;

    sem_init(&headdataavailable, 0, 0);
    sem_init(&headwriteprotect, 0, 1);
    pthread_attr_t attr;
    pthread_attr_init(&attr);

    pthread_create(&worker, &attr, trampoline, this);
    running = true;
}

Looper::~Looper() {
    if (running) {
        ALOGD("Looper deleted while still running. Some messages will not be processed");
        quit();
    }
}

/**
 * 入队消息
 * @param what
 * @param flush
 */
void Looper::postMessage(int what, bool flush) {
    postMessage(what, 0, 0, NULL, flush);
}

/**
 * 入队消息
 * @param what
 * @param obj
 * @param flush
 */
void Looper::postMessage(int what, void *obj, bool flush) {
    postMessage(what, 0, 0, obj, flush);
}

/**
 * 入队消息
 * @param what
 * @param arg1
 * @param arg2
 * @param flush
 */
void Looper::postMessage(int what, int arg1, int arg2, bool flush) {
    postMessage(what, arg1, arg2, NULL, flush);
}

/**
 *
 * @param what
 * @param arg1
 * @param arg2
 * @param obj
 * @param flush
 */
void Looper::postMessage(int what, int arg1, int arg2, void *obj, bool flush) {
    LooperMessage *msg = new LooperMessage();
    msg->what = what;
    msg->obj = obj;
    msg->arg1 = arg1;
    msg->arg2 = arg2;
    msg->next = NULL;
    msg->quit = false;
    addMessage(msg, flush);
}

/**
 * 添加消息
 * @param msg
 * @param flush
 */
void Looper::addMessage(LooperMessage *msg, bool flush) {
    sem_wait(&headwriteprotect);
    LooperMessage *h = head;

    if (flush) {
        while(h) {
            LooperMessage *next = h->next;
            delete h;
            h = next;
        }
        h = NULL;
    }
    if (h) {
        while (h->next) {
            h = h->next;
        }
        h->next = msg;
    } else {
        head = msg;
    }
    ALOGD("post msg %d", msg->what);
    sem_post(&headwriteprotect);
    sem_post(&headdataavailable);
}

/**
 * 循环体
 */
void Looper::loop() {
    while(true) {
        // wait for available message
        sem_wait(&headdataavailable);

        // get next available message
        sem_wait(&headwriteprotect);
        LooperMessage *msg = head;
        if (msg == NULL) {
            ALOGD("no msg");
            sem_post(&headwriteprotect);
            continue;
        }
        head = msg->next;
        sem_post(&headwriteprotect);

        if (msg->quit) {
            ALOGD("quitting");
            delete msg;
            return;
        }
        ALOGD("processing msg %d", msg->what);
        handleMessage(msg);
        delete msg;
    }
}

/**
 * 退出Looper循环
 */
void Looper::quit() {
    ALOGD("quit");
    LooperMessage *msg = new LooperMessage();
    msg->what = 0;
    msg->obj = NULL;
    msg->next = NULL;
    msg->quit = true;
    addMessage(msg, false);
    void *retval;
    pthread_join(worker, &retval);
    sem_destroy(&headdataavailable);
    sem_destroy(&headwriteprotect);
    running = false;
}

/**
 * 处理消息
 * @param what
 * @param data
 */
void Looper::handleMessage(LooperMessage *msg) {
    ALOGD("dropping msg %d %p", msg->what, msg->obj);
}
```
此时，我们封装了一个Looper 基类，里面集成了跟Handler一样的消息队列，这里就不再需要创建MessageQueue了。
接下来，我们在使用的时候，可以继承的Looper，实现MyLooper对象：

MyLooper.h
```
#ifndef EGLNATIVESAMPLE_MYLOOPER_H
#define EGLNATIVESAMPLE_MYLOOPER_H

#include "Looper.h"
#include "../cainrender/GLRender.h"

enum {
    kMsgSurfaceCreated,
    kMsgSurfaceChanged,
    kMsgSurfaceDestroyed
};

class MyLooper : public Looper {

public:
    MyLooper();

    virtual ~MyLooper();

    virtual void handleMessage(LooperMessage *msg);

private:
    GLRender *mRender;

};

#endif //EGLNATIVESAMPLE_MYLOOPER_H
```
MyLooper.cpp:
```
#include "MyLooper.h"

MyLooper::MyLooper() {
    mRender = new GLRender();
}

void MyLooper::handleMessage(LooperMessage *msg) {
    switch (msg && msg->what) {
        case kMsgSurfaceCreated:
            if (mRender) {
                mRender->surfaceCreated((ANativeWindow *) msg->obj);
            }
            break;

        case kMsgSurfaceChanged:
            if (mRender) {
                mRender->surfaceChanged(msg->arg1, msg->arg2);
            }
            break;

        case kMsgSurfaceDestroyed:
            if (mRender) {
                mRender->surfaceDestroyed();
            }
            break;
    }
}

MyLooper::~MyLooper() {
    delete mRender;
}
```
至此，我们在Native层实现了自己的EGL以及GLRender的封装。那么接下来我们应该怎么使用呢？请看下面的步骤：
1、在MainActivity中添加一个SurfaceView，并实现SurfaceHolder.Callback 方法。加载nativeEgl库，并创建几个native方法，如下所示：
```
public class MainActivity extends AppCompatActivity implements SurfaceHolder.Callback {

    // Used to load the 'native-lib' library on application startup.
    static {
        System.loadLibrary("nativeEgl");
    }
    private native void nativeInit();
    private native void nativeRelease();
    private native void onSurfaceCreated(Surface surface);
    private native void onSurfaceChanged(int width, int height);
    private native void onSurfaceDestroyed();

    private SurfaceView mSurfaceView;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        nativeInit();
        mSurfaceView = (SurfaceView) findViewById(R.id.surface_view);
        mSurfaceView.getHolder().addCallback(this);
    }


    @Override
    protected void onDestroy() {
        nativeRelease();
        super.onDestroy();
    }

    @Override
    public void surfaceCreated(SurfaceHolder holder) {
        onSurfaceCreated(holder.getSurface());
    }

    @Override
    public void surfaceChanged(SurfaceHolder holder, int format, int width, int height) {
        onSurfaceChanged(width, height);
    }

    @Override
    public void surfaceDestroyed(SurfaceHolder holder) {
        onSurfaceDestroyed();
    }

}
```
native层实现上面的方法如下：
NativeEglController.cpp:
```
#include <jni.h>
#include "common/Looper.h"
#include "common/MyLooper.h"
#include <android/native_window.h>
#include <android/native_window_jni.h>

MyLooper *mLooper = NULL;
ANativeWindow *mWindow = NULL;

extern "C"
JNIEXPORT void JNICALL
Java_com_cgfay_eglnativerender_MainActivity_nativeInit(JNIEnv *env, jobject instance) {

    // TODO
    mLooper = new MyLooper();
}

extern "C"
JNIEXPORT void JNICALL
Java_com_cgfay_eglnativerender_MainActivity_nativeRelease(JNIEnv *env, jobject instance) {
    if (mLooper != NULL) {
        mLooper->quit();
        delete mLooper;
        mLooper = NULL;
    }
    if (mWindow) {
        ANativeWindow_release(mWindow);
        mWindow = NULL;
    }
}

extern "C"
JNIEXPORT void JNICALL
Java_com_cgfay_eglnativerender_MainActivity_onSurfaceCreated(JNIEnv *env, jobject instance,
                                                             jobject surface) {
    if (mWindow) {
        ANativeWindow_release(mWindow);
        mWindow = NULL;
    }
    mWindow = ANativeWindow_fromSurface(env, surface);
    if (mLooper) {
        mLooper->postMessage(kMsgSurfaceCreated, mWindow);
    }
}

extern "C"
JNIEXPORT void JNICALL
Java_com_cgfay_eglnativerender_MainActivity_onSurfaceChanged(JNIEnv *env, jobject instance,
                                                             jint width, jint height) {
    if (mLooper) {
        mLooper->postMessage(kMsgSurfaceChanged, width, height);
    }

}

extern "C"
JNIEXPORT void JNICALL
Java_com_cgfay_eglnativerender_MainActivity_onSurfaceDestroyed(JNIEnv *env, jobject instance) {

    if (mLooper) {
        mLooper->postMessage(kMsgSurfaceDestroyed);
    }

}
```
编译运行程序，就可以看到Demo成功绘制了一个红色的三角形，如下图所示：
![运行结果](https://upload-images.jianshu.io/upload_images/2103804-5329312129997851.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

使用方式跟我们Java层使用GLSurfaceView没什么两样，至此，我们就封装好EGL + C/C++ 编写OpenGLES的基本框架。Github地址：[EGLNativeRender](https://github.com/CainKernel/EGLNativeRender)
