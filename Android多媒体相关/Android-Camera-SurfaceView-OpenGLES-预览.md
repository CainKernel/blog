在Android Camera SurfaceView 预览拍照一节中( 链接： http://www.jianshu.com/p/9e0f3fc5a3b4 )，我们介绍了怎样使用SurfaceView 预览Camera中的数据，但如果要对预览帧进行操作，比如滤镜、变形等实时操作，则需要引入OpenGLES对SurfaceView 中的Surface进行操作。

在实现之前，我们来比较一下SurfaceView、GLSurfaceView、TextureView吧。
GLSurfaceView是Google封装的继承于SurfaceView的封装了EGLContext、EGLDisplay、EGLSurface的View，使用起来相对方便很多，但是如果要定制更多功能就有点力不从心了，比如我需要使用多个EGLSurface，通过EGLContext上下文进行共享的话，GLSurfaceView实现起来就有点麻烦了，GLSurfaceView 是封装好OpenGLES单线程渲染的View，多线程渲染实现起来有难度，这时候就没SurfaceView好用了。TextureView是4.0之后才出的，相比GLSurfaceView，除了省电之外，GLSurfaceView该有的功能，Android5.0之前TextureView在View hierachy层进行绘制的，Android5.0以后引入了单独的渲染线程之后，TextureView是在渲染线程中进行的，TextureView 也可以实现SurfaceView + SurfaceTexture的功能，实现起来相对麻烦些，流程大体一致。三者的详细分析请参考以下这篇文章，写得比较详细，我这里就不详细介绍了：
http://blog.csdn.net/jinzhuojun/article/details/44062175
另外，如果是使用opengles 做3D渲染的话，个人建议使用SurfaceView，这是从功耗方面考虑的。性能上SurfaceView、GLSurfaceView、TextureView并无明显的差距

好了，我们接下来看看如何实现该功能：
1、首先自定义一个SurfaceView：
```
public class CameraSurfaceView extends SurfaceView implements SurfaceHolder.Callback {

    private static final String TAG = "CameraSurfaceView";

    public CameraSurfaceView(Context context) {
        super(context);
        init();
    }

    public CameraSurfaceView(Context context, AttributeSet attrs) {
        super(context, attrs);
        init();
    }

    public CameraSurfaceView(Context context, AttributeSet attrs, int defStyleAttr) {
        super(context, attrs, defStyleAttr);
        init();
    }

    private void init() {
        getHolder().addCallback(this);
    }


    @Override
    public void surfaceCreated(SurfaceHolder holder) {
        CameraDrawer.INSTANCE.surfaceCreated(holder);
    }

    @Override
    public void surfaceChanged(SurfaceHolder holder, int format, int width, int height) {
        CameraDrawer.INSTANCE.surfacrChanged(width, height);
    }

    @Override
    public void surfaceDestroyed(SurfaceHolder holder) {
        CameraDrawer.INSTANCE.surfaceDestroyed();
    }
}
```
自定义的SurfaceView跟上一篇文章实现方式一致，只是在SurfaceHolder.Callback回调方法中引入了CameraDrawer。这个CameraDrawer是实现OpenGLES绘制Camera预览数据的最核心的类，我们来看看CameraDrawer是如何实现的吧：
CameraDrawer 采用enum实现的单例，关于单例模式的实现方式一般情况下有七种，这里采用的是Effective Java作者Josh Bloch 提倡的方式，采用enum单例模式不仅能避免多线程同步问题，还防止反序列化重新创建新对象。只是enum是Java1.5之后才加入的特性，这种写法估计比较少人用，这里也可以采用双重校验锁的方式实现，这里不做谈论。
创建和销毁HandlerThread 和 Handler的方法：
```
    /**
     * 创建HandlerThread和Handler
     */
    private void create() {
        mHandlerThread = new HandlerThread("CameraDrawer Thread");
        mHandlerThread.start();
        mDrawerHandler = new CameraDrawerHandler(mHandlerThread.getLooper());
        mDrawerHandler.sendEmptyMessage(CameraDrawerHandler.MSG_INIT);
        loopingInterval = CameraUtils.DESIRED_PREVIEW_FPS;
    }

    /**
     * 销毁当前持有的Looper 和 Handler
     */
    public void destory() {
        // Handler不存在时，需要销毁当前线程，否则可能会出现重新打开不了的情况
        if (mDrawerHandler == null) {
            if (mHandlerThread != null) {
                mHandlerThread.quitSafely();
                try {
                    mHandlerThread.join();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                mHandlerThread = null;
            }
            return;
        }
        synchronized (mSynOperation) {
            mDrawerHandler.sendEmptyMessage(CameraDrawerHandler.MSG_DESTROY);
            mHandlerThread.quitSafely();
            try {
                mHandlerThread.join();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            mHandlerThread = null;
            mDrawerHandler = null;
        }
    }
```
如果不熟悉HandlerThread，可以自己找一下资料了解HandlerThread 的源码实现。本质上HandlerThread也是一个Thread，只不过是拥有自己的Looper，创建Handler时，可以绑定该Thread的Looper，这样就能够在同一个线程内处理消息。如果不使用HandlerThread，想要在子线程里面处理消息，你需要通过自定义Thread，然后通过Looper.prepare()、Looper.loop()来创建绑定到Looper。至于手动调用Looper怎么实现，大家可以自己找一下资料，这里就不错详细介绍了。接下来就是暴露出给外界调用的一些逻辑方法，主要是与Surfaceholder回调绑定的方法、开始预览、停止预览、录制、停止录制、拍照、销毁资源等逻辑方法，这些方法通过调用CameraDrawerHandler传递消息进行处理：
```
    public void surfaceCreated(SurfaceHolder holder) {
        create();
        if (mDrawerHandler != null) {
            mDrawerHandler.sendMessage(mDrawerHandler
                    .obtainMessage(CameraDrawerHandler.MSG_SURFACE_CREATED, holder));
        }
    }

    public void surfacrChanged(int width, int height) {
        if (mDrawerHandler != null) {
            mDrawerHandler.sendMessage(mDrawerHandler
                    .obtainMessage(CameraDrawerHandler.MSG_SURFACE_CHANGED, width, height));
        }
        startPreview();
    }

    public void surfaceDestroyed() {
        stopPreview();
        if (mDrawerHandler != null) {
            mDrawerHandler.sendMessage(mDrawerHandler
                    .obtainMessage(CameraDrawerHandler.MSG_SURFACE_DESTROYED));
        }
        destory();
    }

    @Override
    public void onFrameAvailable(SurfaceTexture surfaceTexture) {
        if (mDrawerHandler != null) {
            mDrawerHandler.addNewFrame();
        }
    }

    /**
     * 开始预览
     */
    public void startPreview() {
        if (mDrawerHandler == null) {
            return;
        }
        synchronized (mSynOperation) {
            mDrawerHandler.sendMessage(mDrawerHandler
                    .obtainMessage(CameraDrawerHandler.MSG_START_PREVIEW));
            synchronized (mSyncIsLooping) {
                isPreviewing = true;
            }
        }
    }

    /**
     * 停止预览
     */
    public void stopPreview() {
        if (mDrawerHandler == null) {
            return;
        }
        synchronized (mSynOperation) {
            mDrawerHandler.sendMessage(mDrawerHandler
                    .obtainMessage(CameraDrawerHandler.MSG_STOP_PREVIEW));
            synchronized (mSyncIsLooping) {
                isPreviewing = false;
            }
        }
    }

    /**
     * 改变Filter类型
     */
    public void changeFilterType(FilterType type) {
        if (mDrawerHandler == null) {
            return;
        }
        synchronized (mSynOperation) {
            mDrawerHandler.sendMessage(mDrawerHandler
                    .obtainMessage(CameraDrawerHandler.MSG_FILTER_TYPE, type));
        }
    }

    /**
     * 更新预览大小
     * @param width
     * @param height
     */
    public void updatePreview(int width, int height) {
        if (mDrawerHandler == null) {
            return;
        }
        synchronized (mSynOperation) {
            mDrawerHandler.sendMessage(mDrawerHandler
                    .obtainMessage(CameraDrawerHandler.MSG_UPDATE_PREVIEW, width, height));
        }
    }

    /**
     * 开始录制
     */
    public void startRecording() {
        if (mDrawerHandler == null) {
            return;
        }
        synchronized (mSynOperation) {
            mDrawerHandler.sendMessage(mDrawerHandler
                    .obtainMessage(CameraDrawerHandler.MSG_START_RECORDING));
        }
    }

    /**
     * 停止录制
     */
    public void stopRecording() {
        if (mDrawerHandler == null) {
            return;
        }
        synchronized (mSynOperation) {
            mDrawerHandler.sendEmptyMessage(CameraDrawerHandler.MSG_STOP_RECORDING);
            synchronized (mSyncIsLooping) {
                isRecording = false;
            }
        }
    }

    /**
     * 拍照
     */
    public void takePicture() {
        if (mDrawerHandler == null) {
            return;
        }
        synchronized (mSynOperation) {
            // 发送拍照命令
            mDrawerHandler.sendMessage(mDrawerHandler
                    .obtainMessage(CameraDrawerHandler.MSG_TAKE_PICTURE));
        }
    }
```
好了，我们来看看CameraDrawerHandler是怎样的。CameraDrawerHandler 是一个Handler，主要用来处理HandlerThread中Looper 的消息，其整体实现如下：
```
    private class CameraDrawerHandler extends Handler {

        private static final String TAG = "CameraDrawer";

        static final int MSG_SURFACE_CREATED = 0x001;
        static final int MSG_SURFACE_CHANGED = 0x002;
        static final int MSG_FRAME = 0x003;
        static final int MSG_FILTER_TYPE = 0x004;
        static final int MSG_RESET = 0x005;
        static final int MSG_SURFACE_DESTROYED = 0x006;
        static final int MSG_INIT = 0x007;
        static final int MSG_DESTROY = 0x008;

        static final int MSG_START_PREVIEW = 0x100;
        static final int MSG_STOP_PREVIEW = 0x101;
        static final int MSG_UPDATE_PREVIEW = 0x102;

        static final int MSG_START_RECORDING = 0x200;
        static final int MSG_STOP_RECORDING = 0x201;

        static final int MSG_RESET_BITRATE = 0x300;

        static final int MSG_TAKE_PICTURE = 0x400;

        // EGL共享上下文
        private EglCore mEglCore;
        // EGLSurface
        private WindowSurface mDisplaySurface;
        // CameraTexture对应的Id
        private int mTextureId;
        private SurfaceTexture mCameraTexture;
        // 矩阵
        private final float[] mMatrix = new float[16];
        // 视图宽高
        private int mViewWidth, mViewHeight;
        // 预览图片大小
        private int mImageWidth, mImageHeight;

        // 更新帧的锁
        private final Object mSyncFrameNum = new Object();
        private final Object mSyncTexture = new Object();
        private int mFrameNum = 0;
        private boolean hasNewFrame = false;
        public boolean dropNextFrame = false;
        private boolean isTakePicture = false;
        private boolean mSaveFrame = false;
        private int mSkipFrame = 0;

        private CameraFilter mCameraFilter;
        private BaseImageFilter mFilter;

        public CameraDrawerHandler(Looper looper) {
            super(looper);
        }

        @Override
        public void handleMessage(Message msg) {
            switch (msg.what) {

                // 初始化
                case MSG_INIT:
                    break;

                // 销毁
                case MSG_DESTROY:

                    break;

                // surfacecreated
                case MSG_SURFACE_CREATED:
                    onSurfaceCreated((SurfaceHolder)msg.obj);
                    break;

                // surfaceChanged
                case MSG_SURFACE_CHANGED:
                    onSurfaceChanged(msg.arg1, msg.arg2);
                    break;

                // surfaceDestroyed;
                case MSG_SURFACE_DESTROYED:
                    onSurfaceDestoryed();
                    break;

                // 帧可用（考虑同步的问题）
                case MSG_FRAME:
                    drawFrame();
                    break;

                case MSG_FILTER_TYPE:
                    setFilter((FilterType) msg.obj);
                    break;

                // 重置
                case MSG_RESET:
                    break;

                // 开始预览
                case MSG_START_PREVIEW:
                    break;

                // 停止预览
                case MSG_STOP_PREVIEW:
                    break;

                // 更新预览大小
                case MSG_UPDATE_PREVIEW:
                    synchronized (mSyncIsLooping) {
                        mViewWidth = msg.arg1;
                        mViewHeight = msg.arg2;
                    }
                    break;

                // 开始录制
                case MSG_START_RECORDING:
                    break;

                // 停止录制
                case MSG_STOP_RECORDING:
                    break;

                // 重置bitrate(录制视频时使用)
                case MSG_RESET_BITRATE:
                    break;

                // 拍照
                case MSG_TAKE_PICTURE:
                    isTakePicture = true;
                    break;

                default:
                    throw new IllegalStateException("Can not handle message what is: " + msg.what);
            }
        }
    }
```
我们看到CameraDrawerHandler 主要是用来处理各种消息的，因此这里的消息定义得非常非常多。从初始化、SurfaceHolder回调方法处理实体、帧可用、绘制滤镜、开始录制、停止录制、更新预览大小、拍照等处理实体。Handler 接受消息后，需要对这些消息进行处理。下面介绍接收到消息后的处理方法。
SurfaceHolder回调方法处理实体：
```
        private void onSurfaceCreated(SurfaceHolder holder) {
            mEglCore = new EglCore(null, EglCore.FLAG_RECORDABLE);
            mDisplaySurface = new WindowSurface(mEglCore, holder.getSurface(), false);
            mDisplaySurface.makeCurrent();
            if (mCameraFilter == null) {
                mCameraFilter = new CameraFilter();
            }
            mTextureId = createTextureOES();
            mCameraTexture = new SurfaceTexture(mTextureId);
            mCameraTexture.setOnFrameAvailableListener(CameraDrawer.this);
            CameraUtils.openFrontalCamera(CameraUtils.DESIRED_PREVIEW_FPS);
            calculateImageSize();
            mCameraFilter.onInputSizeChanged(mImageWidth, mImageHeight);
            mFilter = FilterManager.getFilter(FilterType.SATURATION);
            // 禁用深度测试和背面绘制
            GLES30.glDisable(GLES30.GL_DEPTH_TEST);
            GLES30.glDisable(GLES30.GL_CULL_FACE);
        }

        /**
         * 计算imageView 的宽高
         */
        private void calculateImageSize() {
            Camera.Size size = CameraUtils.getPreviewSize();
            Camera.CameraInfo info = CameraUtils.getCameraInfo();
            if (size != null) {
                if (info.orientation == 90 || info.orientation == 270) {
                    mImageWidth = size.height;
                    mImageHeight = size.width;
                } else {
                    mImageWidth = size.width;
                    mImageHeight = size.height;
                }
            } else {
                mImageWidth = 0;
                mImageHeight = 0;
            }
        }
        /**
         * 创建OES 类型的Texture
         * @return
         */
        private int createTextureOES() {
            return GlUtil.createTextureObject(GLES11Ext.GL_TEXTURE_EXTERNAL_OES);
        }
```
我们可以看到在onSurfaceCreated中主要是创建了EglCore 和 WindowSurface，打开相机并设置期望的帧率，禁用深度和背面绘制功能。这里的EglCore 和WidnowsSurface 来自项目Grafika( 项目地址：https://github.com/google/grafika/ )，Grafika是Google工程师业余的Demo，介绍了OpenGLES 在SurfaceView、GLSurfaceView、TextureView的各种用法，其中EglCore 和WindowSurface主要是封装了EGLDisplay、EGLContext 和 EGLSurface，使用起来比较方便，这里就直接引用了。calculateImageSize方法主要是用来计算filter应该渲染的大小，返回的是实际的预览图像的size，这里需要注意的是，SurfaceView 显示和预览图像的Size 并不一定是大小一致的。并且，这里需要根据相机的想转角度做一层处理，将宽度和高度交换。
然后我们再看看onSurfaceChanged里面的内容：
```
        private void onSurfaceChanged(int width, int height) {
            mViewWidth = width;
            mViewHeight = height;
            onFilterChanged();
            CameraUtils.startPreviewTexture(mCameraTexture);
        }

        /**
         * 滤镜或视图发生变化时调用
         */
        private void onFilterChanged() {
            mCameraFilter.onDisplayChanged(mViewWidth, mViewHeight);
            if (mFilter != null) {
                mCameraFilter.initCameraFramebuffer(mImageWidth, mImageHeight);
            } else {
                mCameraFilter.destroyFramebuffer();
            }
        }
```
在onSurfaceChanged回调中调整Filter，这里主要是处理相机的显示Size和是否需要渲染到Framebuffer 中。
接下来我们看看帧可用时，以及绘制帧的具体方法：
```
        /**
         * 绘制帧
         */
        private void drawFrame() {
            mDisplaySurface.makeCurrent();
            synchronized (mSyncFrameNum) {
                synchronized (mSyncTexture) {
                    if (mCameraTexture != null) {
                        // 如果存在新的帧，则更新帧
                        while (mFrameNum != 0) {
                            mCameraTexture.updateTexImage();
                            --mFrameNum;
                            // 是否舍弃下一帧
                            if (!dropNextFrame) {
                                hasNewFrame = true;
                            } else {
                                dropNextFrame = false;
                                hasNewFrame = false;
                            }
                        }
                    } else {
                        return;
                    }
                }
            }
            mCameraTexture.getTransformMatrix(mMatrix);
            if (mFilter == null) {
                mCameraFilter.drawFrame(mTextureId, mMatrix);
            } else {
                int id = mCameraFilter.drawToTexture(mTextureId, mMatrix);
                mFilter.drawFrame(id, mMatrix);
            }
            mDisplaySurface.swapBuffers();
        }

        /**
         * 添加新的一帧
         */
        public void addNewFrame() {
            synchronized (mSyncFrameNum) {
                ++mFrameNum;
                removeMessages(MSG_FRAME);
                sendMessageAtFrontOfQueue(obtainMessage(MSG_FRAME));
            }
        }
```
接下来我们来看看CameraFilter 类，主要是用于绘制相机预览帧的，实现如下：
```
public class CameraFilter extends BaseImageFilter {
    private static final String VERTEX_SHADER =
            "uniform mat4 uMVPMatrix;                               \n" +
            "uniform mat4 uTexMatrix;                               \n" +
            "attribute vec4 aPosition;                              \n" +
            "attribute vec4 aTextureCoord;                          \n" +
            "varying vec2 textureCoordinate;                            \n" +
            "void main() {                                          \n" +
            "    gl_Position = uMVPMatrix * aPosition;              \n" +
            "    textureCoordinate = (uTexMatrix * aTextureCoord).xy;   \n" +
            "}                                                      \n";

    private static final String FRAGMENT_SHADER_OES =
            "#extension GL_OES_EGL_image_external : require         \n" +
            "precision mediump float;                               \n" +
            "varying vec2 textureCoordinate;                            \n" +
            "uniform samplerExternalOES sTexture;                   \n" +
            "void main() {                                          \n" +
            "    gl_FragColor = texture2D(sTexture, textureCoordinate); \n" +
            "}                                                      \n";


    private int[] mFramebuffers;
    private int[] mFramebufferTextures;
    private int mFrameWidth = -1;
    private int mFrameHeight = -1;

    public CameraFilter() {
        mProgramHandle = GlUtil.createProgram(VERTEX_SHADER, FRAGMENT_SHADER_OES);
        muMVPMatrixLoc = GLES30.glGetUniformLocation(mProgramHandle, "uMVPMatrix");
        muTexMatrixLoc = GLES30.glGetUniformLocation(mProgramHandle, "uTexMatrix");
        maPositionLoc = GLES30.glGetAttribLocation(mProgramHandle, "aPosition");
        maTextureCoordLoc = GLES30.glGetAttribLocation(mProgramHandle, "aTextureCoord");
    }

    @Override
    public int getTextureType() {
        return GLES11Ext.GL_TEXTURE_EXTERNAL_OES;
    }

    @Override
    public void drawFrame(int textureId, float[] texMatrix) {
        GLES30.glUseProgram(mProgramHandle);
        GLES30.glActiveTexture(GLES30.GL_TEXTURE0);
        GLES30.glBindTexture(GLES11Ext.GL_TEXTURE_EXTERNAL_OES, textureId);
        GLES30.glUniformMatrix4fv(muMVPMatrixLoc, 1, false, GlUtil.IDENTITY_MATRIX, 0);
        GLES30.glUniformMatrix4fv(muTexMatrixLoc, 1, false, texMatrix, 0);
        runPendingOnDrawTasks();
        GLES30.glEnableVertexAttribArray(maPositionLoc);
        GLES30.glVertexAttribPointer(maPositionLoc, mCoordsPerVertex,
                GLES30.GL_FLOAT, false, mVertexStride, mVertexArray);
        GLES30.glEnableVertexAttribArray(maTextureCoordLoc);
        GLES30.glVertexAttribPointer(maTextureCoordLoc, 2,
                GLES30.GL_FLOAT, false, mTexCoordStride, mTexCoordArray);
        onDrawArraysBegin();
        GLES30.glDrawArrays(GLES30.GL_TRIANGLE_STRIP, 0, mVertexCount);
        GLES30.glDisableVertexAttribArray(maPositionLoc);
        GLES30.glDisableVertexAttribArray(maTextureCoordLoc);
        onDrawArraysAfter();
        GLES30.glBindTexture(GLES11Ext.GL_TEXTURE_EXTERNAL_OES, 0);
        GLES30.glUseProgram(0);
    }

    /**
     * 将SurfaceTexture的纹理绘制到FBO
     * @param textureId
     * @param texMatrix
     * @return 绘制完成返回绑定到Framebuffer中的Texture
     */
    public int drawToTexture(int textureId, float[] texMatrix) {
        GLES30.glBindFramebuffer(GLES30.GL_FRAMEBUFFER, mFramebuffers[0]);
        GLES30.glUseProgram(mProgramHandle);
        GLES30.glActiveTexture(GLES30.GL_TEXTURE0);
        GLES30.glBindTexture(GLES11Ext.GL_TEXTURE_EXTERNAL_OES, textureId);
        GLES30.glUniformMatrix4fv(muMVPMatrixLoc, 1, false, GlUtil.IDENTITY_MATRIX, 0);
        GLES30.glUniformMatrix4fv(muTexMatrixLoc, 1, false, texMatrix, 0);
        GLES30.glEnableVertexAttribArray(maPositionLoc);
        GLES30.glVertexAttribPointer(maPositionLoc, mCoordsPerVertex,
                GLES30.GL_FLOAT, false, mVertexStride, mVertexArray);
        GLES30.glEnableVertexAttribArray(maTextureCoordLoc);
        GLES30.glVertexAttribPointer(maTextureCoordLoc, 2,
                GLES30.GL_FLOAT, false, mTexCoordStride, getTexCoordArray());
        GLES30.glDrawArrays(GLES30.GL_TRIANGLE_STRIP, 0, mVertexCount);
        GLES30.glFinish();
        GLES30.glDisableVertexAttribArray(maPositionLoc);
        GLES30.glDisableVertexAttribArray(maTextureCoordLoc);
        GLES30.glBindTexture(GLES11Ext.GL_TEXTURE_EXTERNAL_OES, 0);
        GLES30.glUseProgram(0);
        GLES30.glBindFramebuffer(GLES30.GL_FRAMEBUFFER, 0);
        return mFramebufferTextures[0];
    }

    public void initCameraFramebuffer(int width, int height) {
        if (mFramebuffers != null && (mFrameWidth != width || mFrameHeight != height)) {
            destroyFramebuffer();
        }
        if (mFramebuffers == null) {
            mFrameWidth = width;
            mFrameHeight = height;
            mFramebuffers = new int[1];
            mFramebufferTextures = new int[1];
            GlUtil.createSampler2DFrameBuff(mFramebuffers, mFramebufferTextures, width, height);
        }
    }

    public void destroyFramebuffer() {
        if (mFramebufferTextures != null) {
            GLES30.glDeleteTextures(1, mFramebufferTextures, 0);
            mFramebufferTextures = null;
        }

        if (mFramebuffers != null) {
            GLES30.glDeleteFramebuffers(1, mFramebuffers, 0);
            mFramebuffers = null;
        }
        mImageWidth = -1;
        mImageHeight = -1;
    }
}
```
CameraFilter 是继承于BaseImageFilter类的，里面的成员变量也是来源于BaseImageFilter类。我们来看看BaseImageFilter类是如何实现的：
```
public class BaseImageFilter {

    protected static final String VERTEX_SHADER =
            "uniform mat4 uMVPMatrix;                                   \n" +
            "uniform mat4 uTexMatrix;                                   \n" +
            "attribute vec4 aPosition;                                  \n" +
            "attribute vec4 aTextureCoord;                              \n" +
            "varying vec2 textureCoordinate;                            \n" +
            "void main() {                                              \n" +
            "    gl_Position = uMVPMatrix * aPosition;                  \n" +
            "    textureCoordinate = (uTexMatrix * aTextureCoord).xy;   \n" +
            "}                                                          \n";

    private static final String FRAGMENT_SHADER_2D =
            "precision mediump float;                                   \n" +
            "varying vec2 textureCoordinate;                            \n" +
            "uniform sampler2D sTexture;                                \n" +
            "void main() {                                              \n" +
            "    gl_FragColor = texture2D(sTexture, textureCoordinate); \n" +
            "}                                                          \n";

    protected static final int SIZEOF_FLOAT = 4;

    protected static final float SquareVertices[] = {
            -1.0f, -1.0f,   // 0 bottom left
            1.0f, -1.0f,   // 1 bottom right
            -1.0f,  1.0f,   // 2 top left
            1.0f,  1.0f,   // 3 top right
    };

    protected static final float TextureVertices[] = {
            0.0f, 0.0f,     // 0 bottom left
            1.0f, 0.0f,     // 1 bottom right
            0.0f, 1.0f,     // 2 top left
            1.0f, 1.0f      // 3 top right
    };

    protected static final float TextureVertices_90[] = {
            1.0f, 0.0f,
            1.0f, 1.0f,
            0.0f, 0.0f,
            0.0f, 1.0f
    };

    protected static final float TextureVertices_180[] = {
            1.0f, 1.0f,
            0.0f, 1.0f,
            1.0f, 0.0f,
            0.0f, 0.0f
    };

    protected static final float TextureVertices_270[] = {
            0.0f, 1.0f,
            0.0f, 0.0f,
            1.0f, 1.0f,
            0.0f, 0.0f
    };

    private static final FloatBuffer FULL_RECTANGLE_BUF =
            GlUtil.createFloatBuffer(SquareVertices);

    protected FloatBuffer mVertexArray = FULL_RECTANGLE_BUF;
    protected FloatBuffer mTexCoordArray = GlUtil.createFloatBuffer(TextureVertices);
    protected int mCoordsPerVertex = 2;
    protected int mVertexStride = mCoordsPerVertex * SIZEOF_FLOAT;
    protected int mVertexCount = SquareVertices.length / mCoordsPerVertex;
    protected int mTexCoordStride = 2 * SIZEOF_FLOAT;

    protected int mProgramHandle;
    protected int muMVPMatrixLoc;
    protected int muTexMatrixLoc;
    protected int maPositionLoc;
    protected int maTextureCoordLoc;

    // 渲染的Image的宽高
    protected int mImageWidth;
    protected int mImageHeight;
    // 显示输出的宽高
    protected int mDisplayWidth;
    protected int mDisplayHeight;

    private final LinkedList<Runnable> mRunOnDraw;

    public BaseImageFilter() {
        this(VERTEX_SHADER, FRAGMENT_SHADER_2D);
    }

    public BaseImageFilter(String vertexShader, String fragmentShader) {
        mRunOnDraw = new LinkedList<>();
        mProgramHandle = GlUtil.createProgram(vertexShader, fragmentShader);
        maPositionLoc = GLES30.glGetAttribLocation(mProgramHandle, "aPosition");
        maTextureCoordLoc = GLES30.glGetAttribLocation(mProgramHandle, "aTextureCoord");
        muMVPMatrixLoc = GLES30.glGetUniformLocation(mProgramHandle, "uMVPMatrix");
        muTexMatrixLoc = GLES30.glGetUniformLocation(mProgramHandle, "uTexMatrix");
    }

    /**
     * Surface发生变化时调用
     * @param width
     * @param height
     */
    public void onInputSizeChanged(int width, int height) {
        mImageWidth = width;
        mImageHeight = height;
    }

    /**
     *  显示视图发生变化时调用
     * @param width
     * @param height
     */
    public void onDisplayChanged(int width, int height) {
        mDisplayWidth = width;
        mDisplayHeight = height;
    }

    /**
     * 绘制到FBO
     * @param textureId
     * @param texMatrix
     */
    public void drawFrame(int textureId, float[] texMatrix) {
        GLES30.glUseProgram(mProgramHandle);
        GLES30.glActiveTexture(GLES30.GL_TEXTURE0);
        GLES30.glBindTexture(GLES30.GL_TEXTURE_2D, textureId);
        GLES30.glUniformMatrix4fv(muMVPMatrixLoc, 1, false, GlUtil.IDENTITY_MATRIX, 0);
        GLES30.glUniformMatrix4fv(muTexMatrixLoc, 1, false, texMatrix, 0);
        runPendingOnDrawTasks();
        GLES30.glEnableVertexAttribArray(maPositionLoc);
        GLES30.glVertexAttribPointer(maPositionLoc, mCoordsPerVertex,
                GLES30.GL_FLOAT, false, mVertexStride, mVertexArray);
        GLES30.glEnableVertexAttribArray(maTextureCoordLoc);
        GLES30.glVertexAttribPointer(maTextureCoordLoc, 2,
                GLES30.GL_FLOAT, false, mTexCoordStride, mTexCoordArray);
        onDrawArraysBegin();
        GLES30.glDrawArrays(GLES30.GL_TRIANGLE_STRIP, 0, mVertexCount);
        GLES30.glDisableVertexAttribArray(maPositionLoc);
        GLES30.glDisableVertexAttribArray(maTextureCoordLoc);
        onDrawArraysAfter();
        GLES30.glBindTexture(GLES30.GL_TEXTURE_2D, 0);
        GLES30.glUseProgram(0);
    }

    /**
     * 获取Texture类型
     * GLES30.TEXTURE_2D / GLES11Ext.GL_TEXTURE_EXTERNAL_OES等
     */
    public int getTextureType() {
        return GLES30.GL_TEXTURE_2D;
    }

    /**
     * 调用drawArrays之前，方便添加其他属性
     */
    public void onDrawArraysBegin() {

    }

    /**
     * drawArrays调用之后，方便销毁其他属性
     */
    public void onDrawArraysAfter() {

    }

    /**
     * 释放资源
     */
    public void release() {
        GLES30.glDeleteProgram(mProgramHandle);
        mProgramHandle = -1;
    }

    public FloatBuffer getTexCoordArray() {
        FloatBuffer result = null;
        switch (CameraUtils.getPreviewOrientation()) {
            case 0:
                result = GlUtil.createFloatBuffer(TextureVertices);
                break;

            case 90:
                result = GlUtil.createFloatBuffer(TextureVertices_90);
                break;

            case 180:
                result = GlUtil.createFloatBuffer(TextureVertices_180);
                break;

            case 270:
                result = GlUtil.createFloatBuffer(TextureVertices_270);
                break;

            default:
                result = GlUtil.createFloatBuffer(TextureVertices);
        }
        return result;
    }

    ///------------------ 同一变量(uniform)设置 ------------------------///
    protected void setInteger(final int location, final int intValue) {
        runOnDraw(new Runnable() {
            @Override
            public void run() {
                GLES30.glUniform1i(location, intValue);
            }
        });
    }

    protected void setFloat(final int location, final float floatValue) {
        runOnDraw(new Runnable() {
            @Override
            public void run() {
                GLES30.glUniform1f(location, floatValue);
            }
        });
    }

    protected void setFloatVec2(final int location, final float[] arrayValue) {
        runOnDraw(new Runnable() {
            @Override
            public void run() {
                GLES30.glUniform2fv(location, 1, FloatBuffer.wrap(arrayValue));
            }
        });
    }

    protected void setFloatVec3(final int location, final float[] arrayValue) {
        runOnDraw(new Runnable() {
            @Override
            public void run() {
                GLES30.glUniform3fv(location, 1, FloatBuffer.wrap(arrayValue));
            }
        });
    }

    protected void setFloatVec4(final int location, final float[] arrayValue) {
        runOnDraw(new Runnable() {
            @Override
            public void run() {
                GLES30.glUniform4fv(location, 1, FloatBuffer.wrap(arrayValue));
            }
        });
    }

    protected void setFloatArray(final int location, final float[] arrayValue) {
        runOnDraw(new Runnable() {
            @Override
            public void run() {
                GLES30.glUniform1fv(location, arrayValue.length, FloatBuffer.wrap(arrayValue));
            }
        });
    }

    protected void setPoint(final int location, final PointF point) {
        runOnDraw(new Runnable() {

            @Override
            public void run() {
                float[] vec2 = new float[2];
                vec2[0] = point.x;
                vec2[1] = point.y;
                GLES30.glUniform2fv(location, 1, vec2, 0);
            }
        });
    }

    protected void setUniformMatrix3f(final int location, final float[] matrix) {
        runOnDraw(new Runnable() {

            @Override
            public void run() {
                GLES30.glUniformMatrix3fv(location, 1, false, matrix, 0);
            }
        });
    }

    protected void setUniformMatrix4f(final int location, final float[] matrix) {
        runOnDraw(new Runnable() {

            @Override
            public void run() {
                GLES30.glUniformMatrix4fv(location, 1, false, matrix, 0);
            }
        });
    }

    protected void runOnDraw(final Runnable runnable) {
        synchronized (mRunOnDraw) {
            mRunOnDraw.addLast(runnable);
        }
    }

    protected void runPendingOnDrawTasks() {
        while (!mRunOnDraw.isEmpty()) {
            mRunOnDraw.removeFirst().run();
        }
    }
}
```
至此，使用SurfaceView + OpenGLES 绘制Camera预览数据的整体框架搭建完成。接下来可以在此基础上添加各种滤镜，添加录制等功能了。详细实现过程可以参考本人开发的相机项目CainCamera，该项目地址如下：
https://github.com/CainKernel/CainCamera

备注： 
该项目主要是用来熟悉Camera 开发流程的，我会不定期修复开发过程中遇到的Bug，并逐步加入新的滤镜，新滤镜的加入同时也会新增文章介绍滤镜使用到的相关数学知识。
