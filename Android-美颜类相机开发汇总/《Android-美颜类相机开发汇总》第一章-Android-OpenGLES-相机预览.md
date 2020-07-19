### SurfaceView + OpenGLES 预览相机
使用OpenGLES 预览相机，我们可以通过GLSurfaceView 来预览相机。GLSurfaceView封装了EGLContext。关于GLSurfaceView的源码里面，GLThread作为单独的线程处理OpenGL的绘制操作，但是这里有个问题，我们可以看看GLThread里面的循环：
```
while (true) {
    synchronized (sGLThreadManager) {
        while (true) {
            ...   // 处理是否需要刷新
            sGLThreadManager.wait();
        }
    } // end of synchronized(sGLThreadManager)
    if (event != null) {
        event.run();
        event = null;
        continue;
    }
    ...
}
```
内循环是用来判断是否需要走绘制循环。当使用RENDERMODE_WHEN_DIRTY而非RENDERMODE_CONTINUOUSLY时，如果我们不主动调用requestRender绘制的话，它会一直在内部等待。然后另外一点就是，当我们调用queueEvent方法过多的时候，会导致event事件过多，然后需要不断地循环处理event事件，最终并没有走到刷新画面的流程。也就是说，为了保证得到更高的fps，我们需要解决这个问题。还有另外一个问题就是GLSurfaceView 中的EGL环境有可能会丢失重建的情况，对后续利用SharedContext做录制处理有影响。
因此，我没有使用GLSurfaceView来做绘制操作，用另外一个Looper线程单独处理OpenGLES 的纹理资源加载、渲染等操作。放弃使用GLSurfaceView 的另外一大原因是，为了利用SharedContext实现无丢帧录制视频的功能，GLSurfaceView 有可能会在中途释放并重新创建EGLContext，导致SharedContext失效，录制失败的情况。关于这个的话，可以参考[grafika](https://github.com/google/grafika)，里面有issue讨论过这个问题。
关于SurfaceView + OpenGLES 预览相机，可以参考本人的文章：
[Android Camera SurfaceView OpenGLES 预览](https://www.jianshu.com/p/e4643b141644)
这篇文章是很久之前写的，现在CainCamera开源项目已经发生了比较大的改变。这里还是重新介绍一遍吧。不过这次应该是最后一次大改动了，相机部分的功能基本已经完成，只剩一些小功能没有实现而已，而且暂时也不会再更新相机部分的功能了。

* 渲染线程 —— HandlerThread 
通过HandlerThread 创建EGLContext绑定的渲染线程，如下：
```
class RenderThread extends HandlerThread implements SurfaceTexture.OnFrameAvailableListener,
        Camera.PreviewCallback {

    private static final String TAG = "RenderThread";
    private static final boolean VERBOSE = false;

    // 操作锁
    private final Object mSynOperation = new Object();
    // 更新帧的锁
    private final Object mSyncFrameNum = new Object();
    private final Object mSyncFence = new Object();

    private boolean isPreviewing = false;       // 是否预览状态
    private boolean isRecording = false;        // 是否录制状态
    private boolean isRecordingPause = false;   // 是否处于暂停录制状态

    // EGL共享上下文
    private EglCore mEglCore;
    // 预览用的EGLSurface
    private WindowSurface mDisplaySurface;

    private int mInputTexture;
    private int mCurrentTexture;
    private SurfaceTexture mSurfaceTexture;

    // 矩阵
    private final float[] mMatrix = new float[16];

    // 预览回调
    private byte[] mPreviewBuffer;
    // 输入图像大小
    private int mTextureWidth, mTextureHeight;

    // 可用帧
    private int mFrameNum = 0;

    // 渲染Handler回调
    private RenderHandler mRenderHandler;

    // 计算帧率
    private FrameRateMeter mFrameRateMeter;

    // 上下文
    private Context mContext;

    // 正在拍照
    private volatile boolean mTakingPicture;
    // 预览参数
    private CameraParam mCameraParam;

    // 渲染管理器
    private RenderManager mRenderManager;

    public RenderThread(Context context, String name) {
        super(name);
        mContext = context;
        mCameraParam = CameraParam.getInstance();
        mRenderManager = RenderManager.getInstance();
        mFrameRateMeter = new FrameRateMeter();
    }

    /**
     * 设置预览Handler回调
     * @param handler
     */
    public void setRenderHandler(RenderHandler handler) {
        mRenderHandler = handler;
    }

    @Override
    public void onFrameAvailable(SurfaceTexture surfaceTexture) {

    }

    private long time = 0;
    @Override
    public void onPreviewFrame(byte[] data, Camera camera) {
        synchronized (mSynOperation) {
            if (isPreviewing || isRecording) {
                mRenderHandler.sendMessage(mRenderHandler
                        .obtainMessage(RenderHandler.MSG_PREVIEW_CALLBACK, data));
            }
        }
        if (mPreviewBuffer != null) {
            camera.addCallbackBuffer(mPreviewBuffer);
        }
        // 计算fps
        if (mRenderHandler != null && mCameraParam.showFps) {
            mRenderHandler.sendEmptyMessage(RenderHandler.MSG_CALCULATE_FPS);
        }
        if (VERBOSE) {
            Log.d("onPreviewFrame", "update time = " + (System.currentTimeMillis() - time));
            time = System.currentTimeMillis();
        }
    }

    /**
     * 预览回调
     * @param data
     */
    void onPreviewCallback(byte[] data) {
        if (mCameraParam.cameraCallback != null) {
            mCameraParam.cameraCallback.onPreviewCallback(data);
        }
    }

    /**
     * Surface创建
     * @param holder
     */
    void surfaceCreated(SurfaceHolder holder) {
        mEglCore = new EglCore(null, EglCore.FLAG_RECORDABLE);
        mDisplaySurface = new WindowSurface(mEglCore, holder.getSurface(), false);
        mDisplaySurface.makeCurrent();

        GLES30.glDisable(GLES30.GL_DEPTH_TEST);
        GLES30.glDisable(GLES30.GL_CULL_FACE);

        // 渲染器初始化
        mRenderManager.init(mContext);

        mInputTexture = OpenGLUtils.createOESTexture();
        mSurfaceTexture = new SurfaceTexture(mInputTexture);
        mSurfaceTexture.setOnFrameAvailableListener(this);

        // 打开相机
        openCamera();

    }

    /**
     * Surface改变
     * @param width
     * @param height
     */
    void surfaceChanged(int width, int height) {
        mRenderManager.setDisplaySize(width, height);
        startPreview();
    }

    /**
     * Surface销毁
     */
    void surfaceDestroyed() {
        mTakingPicture = false;
        mRenderManager.release();
        releaseCamera();
        if (mSurfaceTexture != null) {
            mSurfaceTexture.release();
            mSurfaceTexture = null;
        }
        if (mDisplaySurface != null) {
            mDisplaySurface.release();
            mDisplaySurface = null;
        }
        if (mEglCore != null) {
            mEglCore.release();
            mEglCore = null;
        }
    }

    /**
     * 绘制帧
     */
    void drawFrame() {
        // 如果存在新的帧，则更新帧
        synchronized (mSyncFrameNum) {
            synchronized (mSyncFence) {
                if (mSurfaceTexture != null) {
                    while (mFrameNum != 0) {
                        mSurfaceTexture.updateTexImage();
                        --mFrameNum;
                    }
                } else {
                    return;
                }
            }
        }

        // 切换渲染上下文
        mDisplaySurface.makeCurrent();
        mSurfaceTexture.getTransformMatrix(mMatrix);

        // 绘制渲染
        mCurrentTexture = mRenderManager.drawFrame(mInputTexture, mMatrix);

        // 是否绘制人脸关键点
        mRenderManager.drawFacePoint(mCurrentTexture);

        // 显示到屏幕
        mDisplaySurface.swapBuffers();

        // 执行拍照
        if (mCameraParam.isTakePicture && !mTakingPicture) {
            synchronized (mSyncFence) {
                mTakingPicture = true;
                mRenderHandler.sendEmptyMessage(RenderHandler.MSG_TAKE_PICTURE);
            }
        }

        // 是否处于录制状态
        if (isRecording && !isRecordingPause) {
            HardcodeEncoder.getInstance().frameAvailable();
            HardcodeEncoder.getInstance()
                    .drawRecorderFrame(mCurrentTexture, mSurfaceTexture.getTimestamp());
        }
    }

    /**
     * 拍照
     */
    void takePicture() {
        synchronized (mSyncFence) {
            ByteBuffer buffer = mDisplaySurface.getCurrentFrame();
            mCameraParam.captureCallback.onCapture(buffer,
                    mDisplaySurface.getWidth(), mDisplaySurface.getHeight());
            mTakingPicture = false;
            mCameraParam.isTakePicture = false;
        }
    }

    /**
     * 计算fps
     */
    void calculateFps() {
        // 帧率回调
        if ((mCameraParam).fpsCallback != null) {
            mFrameRateMeter.drawFrameCount();
            (mCameraParam).fpsCallback.onFpsCallback(mFrameRateMeter.getFPS());
        }
    }

    /**
     * 计算imageView 的宽高
     */
    private void calculateImageSize() {
        if (mCameraParam.orientation == 90 || mCameraParam.orientation == 270) {
            mTextureWidth = mCameraParam.previewHeight;
            mTextureHeight = mCameraParam.previewWidth;
        } else {
            mTextureWidth = mCameraParam.previewWidth;
            mTextureHeight = mCameraParam.previewHeight;
        }
        mRenderManager.setTextureSize(mTextureWidth, mTextureHeight);
    }

    /**
     * 切换边框模糊
     * @param enableEdgeBlur
     */
    void changeEdgeBlurFilter(boolean enableEdgeBlur) {
        synchronized (mSynOperation) {
            mRenderManager.changeEdgeBlurFilter(enableEdgeBlur);
        }
    }

    /**
     * 切换动态滤镜
     * @param color
     */
    void changeDynamicFilter(DynamicColor color) {
        synchronized (mSynOperation) {
            mRenderManager.changeDynamicFilter(color);
        }
    }

    /**
     * 切换动态彩妆
     * @param makeup
     */
    void changeDynamicMakeup(DynamicMakeup makeup) {
        synchronized (mSynOperation) {
            mRenderManager.changeDynamicMakeup(makeup);
        }
    }

    /**
     * 切换动态资源
     * @param color
     */
    void changeDynamicResource(DynamicColor color) {
        synchronized (mSynOperation) {
            mRenderManager.changeDynamicResource(color);
        }
    }

    /**
     * 切换动态资源
     * @param sticker
     */
    void changeDynamicResource(DynamicSticker sticker) {
        synchronized (mSynOperation) {
            mRenderManager.changeDynamicResource(sticker);
        }
    }

    /**
     * 开始录制
     */
    void startRecording() {
        if (mEglCore != null) {
            // 设置渲染Texture 的宽高
            HardcodeEncoder.getInstance().setTextureSize(mTextureWidth, mTextureHeight);
            // 这里将EGLContext传递到录制线程共享。
            // 由于EGLContext是当前线程手动创建，也就是OpenGLES的main thread
            // 这里需要传自己手动创建的EglContext
            HardcodeEncoder.getInstance().startRecording(mContext, mEglCore.getEGLContext());
        }
        isRecording = true;
    }

    /**
     * 停止录制
     */
    void stopRecording() {
        HardcodeEncoder.getInstance().stopRecording();
        isRecording = false;
    }

    /**
     * 请求刷新
     */
    public void requestRender() {
        synchronized (mSyncFrameNum) {
            if (isPreviewing) {
                ++mFrameNum;
                if (mRenderHandler != null) {
                    mRenderHandler.removeMessages(RenderHandler.MSG_RENDER);
                    mRenderHandler.sendMessage(mRenderHandler
                            .obtainMessage(RenderHandler.MSG_RENDER));
                }
            }
        }
    }


    // --------------------------------- 相机操作逻辑 ----------------------------------------------
    /**
     * 打开相机
     */
    void openCamera() {
        releaseCamera();
        CameraEngine.getInstance().openCamera(mContext);
        CameraEngine.getInstance().setPreviewSurface(mSurfaceTexture);
        calculateImageSize();
        mPreviewBuffer = new byte[mTextureWidth * mTextureHeight * 3/ 2];
        CameraEngine.getInstance().setPreviewCallbackWithBuffer(this, mPreviewBuffer);
        // 相机打开回调
        if (mCameraParam.cameraCallback != null) {
            mCameraParam.cameraCallback.onCameraOpened();
        }
    }

    /**
     * 切换相机
     */
    void switchCamera() {
        mCameraParam.backCamera = !mCameraParam.backCamera;
        if (mCameraParam.backCamera) {
            mCameraParam.cameraId = Camera.CameraInfo.CAMERA_FACING_BACK;
        } else {
            mCameraParam.cameraId = Camera.CameraInfo.CAMERA_FACING_FRONT;
        }
        openCamera();
        startPreview();
    }

    /**
     * 开始预览
     */
    private void startPreview() {
        CameraEngine.getInstance().startPreview();
        isPreviewing = true;
    }

    /**
     * 释放相机
     */
    private void releaseCamera() {
        isPreviewing = false;
        CameraEngine.getInstance().releaseCamera();
    }
}
```
相机的操作也放在该线程。这样相机打开关闭操作、渲染前处理等操作都不会影响到UI的响应了。CameraEngine则是相机引擎单例，用于控制相机操作的。RenderManager 则是渲染操作的单例，如果你不想用SurfaceView，也可以单独将RenderManager提取出来，放到GLSurfaceView中。

* 渲染管理器 —— RenderManager
RenderManager 的代码如下：
```
public final class RenderManager {

    private static class RenderManagerHolder {
        public static RenderManager instance = new RenderManager();
    }

    private RenderManager() {
        mCameraParam = CameraParam.getInstance();
    }

    public static RenderManager getInstance() {
        return RenderManagerHolder.instance;
    }

    // 滤镜列表
    private SparseArray<GLImageFilter> mFilterArrays = new SparseArray<GLImageFilter>();

    // 坐标缓冲
    private ScaleType mScaleType = ScaleType.CENTER_CROP;
    private FloatBuffer mVertexBuffer;
    private FloatBuffer mTextureBuffer;
    // 用于显示裁剪的纹理顶点缓冲
    private FloatBuffer mDisplayVertexBuffer;
    private FloatBuffer mDisplayTextureBuffer;

    // 视图宽高
    private int mViewWidth, mViewHeight;
    // 输入图像大小
    private int mTextureWidth, mTextureHeight;

    // 相机参数
    private CameraParam mCameraParam;
    // 上下文
    private Context mContext;

    /**
     * 初始化
     */
    public void init(Context context) {
        initBuffers();
        initFilters(context);
        mContext = context;
    }

    /**
     * 释放资源
     */
    public void release() {
        releaseBuffers();
        releaseFilters();
        mContext = null;
    }

    /**
     * 释放滤镜
     */
    private void releaseFilters() {
        for (int i = 0; i < mFilterArrays.size(); i++) {
            if (mFilterArrays.get(i) != null) {
                mFilterArrays.get(i).release();
            }
        }
        mFilterArrays.clear();
    }

    /**
     * 释放缓冲区
     */
    private void releaseBuffers() {
        if (mVertexBuffer != null) {
            mVertexBuffer.clear();
            mVertexBuffer = null;
        }
        if (mTextureBuffer != null) {
            mTextureBuffer.clear();
            mTextureBuffer = null;
        }
        if (mDisplayVertexBuffer != null) {
            mDisplayVertexBuffer.clear();
            mDisplayVertexBuffer = null;
        }
        if (mDisplayTextureBuffer != null) {
            mDisplayTextureBuffer.clear();
            mDisplayTextureBuffer = null;
        }
    }

    /**
     * 初始化缓冲区
     */
    private void initBuffers() {
        releaseBuffers();
        mDisplayVertexBuffer = OpenGLUtils.createFloatBuffer(TextureRotationUtils.CubeVertices);
        mDisplayTextureBuffer = OpenGLUtils.createFloatBuffer(TextureRotationUtils.TextureVertices);
        mVertexBuffer = OpenGLUtils.createFloatBuffer(TextureRotationUtils.CubeVertices);
        mTextureBuffer = OpenGLUtils.createFloatBuffer(TextureRotationUtils.TextureVertices);
    }

    /**
     * 初始化滤镜
     * @param context
     */
    private void initFilters(Context context) {
        releaseFilters();
        // 相机输入滤镜
        mFilterArrays.put(RenderIndex.CameraIndex, new GLImageOESInputFilter(context));
        // 美颜滤镜
        mFilterArrays.put(RenderIndex.BeautyIndex, new GLImageBeautyFilter(context));
        // 彩妆滤镜
        mFilterArrays.put(RenderIndex.MakeupIndex, new GLImageMakeupFilter(context, null));
        // 美型滤镜
        mFilterArrays.put(RenderIndex.FaceAdjustIndex, new GLImageFaceReshapeFilter(context));
        // LUT/颜色滤镜
        mFilterArrays.put(RenderIndex.FilterIndex, null);
        // 贴纸资源滤镜
        mFilterArrays.put(RenderIndex.ResourceIndex, null);
        // 景深滤镜
        mFilterArrays.put(RenderIndex.DepthBlurIndex, new GLImageDepthBlurFilter(context));
        // 暗角滤镜
        mFilterArrays.put(RenderIndex.VignetteIndex, new GLImageVignetteFilter(context));
        // 显示输出
        mFilterArrays.put(RenderIndex.DisplayIndex, new GLImageFilter(context));
        // 人脸关键点调试
        mFilterArrays.put(RenderIndex.FacePointIndex, new GLImageFacePointsFilter(context));
    }

    /**
     * 是否切换边框模糊
     * @param enableEdgeBlur
     */
    public synchronized void changeEdgeBlurFilter(boolean enableEdgeBlur) {
        if (enableEdgeBlur) {
            mFilterArrays.get(RenderIndex.DisplayIndex).release();
            GLImageFrameEdgeBlurFilter filter = new GLImageFrameEdgeBlurFilter(mContext);
            filter.onInputSizeChanged(mTextureWidth, mTextureHeight);
            filter.onDisplaySizeChanged(mViewWidth, mViewHeight);
            mFilterArrays.put(RenderIndex.DisplayIndex, filter);
        } else {
            mFilterArrays.get(RenderIndex.DisplayIndex).release();
            GLImageFilter filter = new GLImageFilter(mContext);
            filter.onInputSizeChanged(mTextureWidth, mTextureHeight);
            filter.onDisplaySizeChanged(mViewWidth, mViewHeight);
            mFilterArrays.put(RenderIndex.DisplayIndex, filter);
        }
    }

    /**
     * 切换动态滤镜
     * @param color
     */
    public synchronized void changeDynamicFilter(DynamicColor color) {
        if (mFilterArrays.get(RenderIndex.FilterIndex) != null) {
            mFilterArrays.get(RenderIndex.FilterIndex).release();
            mFilterArrays.put(RenderIndex.FilterIndex, null);
        }
        if (color == null) {
            return;
        }
        GLImageDynamicColorFilter filter = new GLImageDynamicColorFilter(mContext, color);
        filter.onInputSizeChanged(mTextureWidth, mTextureHeight);
        filter.initFrameBuffer(mTextureWidth, mTextureHeight);
        filter.onDisplaySizeChanged(mViewWidth, mViewHeight);
        mFilterArrays.put(RenderIndex.FilterIndex, filter);
    }

    /**
     * 切换动态滤镜
     * @param dynamicMakeup
     */
    public synchronized void changeDynamicMakeup(DynamicMakeup dynamicMakeup) {
        if (mFilterArrays.get(RenderIndex.MakeupIndex) != null) {
            ((GLImageMakeupFilter)mFilterArrays.get(RenderIndex.MakeupIndex)).changeMakeupData(dynamicMakeup);
        } else {
            GLImageMakeupFilter filter = new GLImageMakeupFilter(mContext, dynamicMakeup);
            filter.onInputSizeChanged(mTextureWidth, mTextureHeight);
            filter.initFrameBuffer(mTextureWidth, mTextureHeight);
            filter.onDisplaySizeChanged(mViewWidth, mViewHeight);
            mFilterArrays.put(RenderIndex.MakeupIndex, filter);
        }
    }

    /**
     * 切换动态资源
     * @param color
     */
    public synchronized void changeDynamicResource(DynamicColor color) {
        if (mFilterArrays.get(RenderIndex.ResourceIndex) != null) {
            mFilterArrays.get(RenderIndex.ResourceIndex).release();
            mFilterArrays.put(RenderIndex.ResourceIndex, null);
        }
        if (color == null) {
            return;
        }
        GLImageDynamicColorFilter filter = new GLImageDynamicColorFilter(mContext, color);
        filter.onInputSizeChanged(mTextureWidth, mTextureHeight);
        filter.initFrameBuffer(mTextureWidth, mTextureHeight);
        filter.onDisplaySizeChanged(mViewWidth, mViewHeight);
        mFilterArrays.put(RenderIndex.ResourceIndex, filter);
    }

    /**
     * 切换动态资源
     * @param sticker
     */
    public synchronized void changeDynamicResource(DynamicSticker sticker) {
        // 释放旧滤镜
        if (mFilterArrays.get(RenderIndex.ResourceIndex) != null) {
            mFilterArrays.get(RenderIndex.ResourceIndex).release();
            mFilterArrays.put(RenderIndex.ResourceIndex, null);
        }
        if (sticker == null) {
            return;
        }
        GLImageDynamicStickerFilter filter = new GLImageDynamicStickerFilter(mContext, sticker);
        // 设置输入输入大小，初始化fbo等
        filter.onInputSizeChanged(mTextureWidth, mTextureHeight);
        filter.initFrameBuffer(mTextureWidth, mTextureHeight);
        filter.onDisplaySizeChanged(mViewWidth, mViewHeight);
        mFilterArrays.put(RenderIndex.ResourceIndex, filter);
    }

    /**
     * 绘制纹理
     * @param inputTexture
     * @param mMatrix
     * @return
     */
    public int drawFrame(int inputTexture, float[] mMatrix) {
        int currentTexture = inputTexture;
        if (mFilterArrays.get(RenderIndex.CameraIndex) == null
                || mFilterArrays.get(RenderIndex.DisplayIndex) == null) {
            return currentTexture;
        }
        if (mFilterArrays.get(RenderIndex.CameraIndex) instanceof GLImageOESInputFilter) {
            ((GLImageOESInputFilter)mFilterArrays.get(RenderIndex.CameraIndex)).setTextureTransformMatrix(mMatrix);
        }
        currentTexture = mFilterArrays.get(RenderIndex.CameraIndex)
                .drawFrameBuffer(currentTexture, mVertexBuffer, mTextureBuffer);
        // 如果处于对比状态，不做处理
        if (!mCameraParam.showCompare) {
            // 美颜滤镜
            if (mFilterArrays.get(RenderIndex.BeautyIndex) != null) {
                if (mFilterArrays.get(RenderIndex.BeautyIndex) instanceof IBeautify
                        && mCameraParam.beauty != null) {
                    ((IBeautify) mFilterArrays.get(RenderIndex.BeautyIndex)).onBeauty(mCameraParam.beauty);
                }
                currentTexture = mFilterArrays.get(RenderIndex.BeautyIndex).drawFrameBuffer(currentTexture, mVertexBuffer, mTextureBuffer);
            }

            // 彩妆滤镜
            if (mFilterArrays.get(RenderIndex.MakeupIndex) != null) {
                currentTexture = mFilterArrays.get(RenderIndex.MakeupIndex).drawFrameBuffer(currentTexture, mVertexBuffer, mTextureBuffer);
            }

            // 美型滤镜
            if (mFilterArrays.get(RenderIndex.FaceAdjustIndex) != null) {
                if (mFilterArrays.get(RenderIndex.FaceAdjustIndex) instanceof IBeautify) {
                    ((IBeautify) mFilterArrays.get(RenderIndex.FaceAdjustIndex)).onBeauty(mCameraParam.beauty);
                }
                currentTexture = mFilterArrays.get(RenderIndex.FaceAdjustIndex).drawFrameBuffer(currentTexture, mVertexBuffer, mTextureBuffer);
            }

            // 绘制颜色滤镜
            if (mFilterArrays.get(RenderIndex.FilterIndex) != null) {
                currentTexture = mFilterArrays.get(RenderIndex.FilterIndex).drawFrameBuffer(currentTexture, mVertexBuffer, mTextureBuffer);
            }

            // 资源滤镜，可以是贴纸、滤镜甚至是彩妆类型
            if (mFilterArrays.get(RenderIndex.ResourceIndex) != null) {
                currentTexture = mFilterArrays.get(RenderIndex.ResourceIndex).drawFrameBuffer(currentTexture, mVertexBuffer, mTextureBuffer);
            }

            // 景深
            if (mFilterArrays.get(RenderIndex.DepthBlurIndex) != null) {
                mFilterArrays.get(RenderIndex.DepthBlurIndex).setFilterEnable(mCameraParam.enableDepthBlur);
                currentTexture = mFilterArrays.get(RenderIndex.DepthBlurIndex).drawFrameBuffer(currentTexture, mVertexBuffer, mTextureBuffer);
            }

            // 暗角
            if (mFilterArrays.get(RenderIndex.VignetteIndex) != null) {
                mFilterArrays.get(RenderIndex.VignetteIndex).setFilterEnable(mCameraParam.enableVignette);
                currentTexture = mFilterArrays.get(RenderIndex.VignetteIndex).drawFrameBuffer(currentTexture, mVertexBuffer, mTextureBuffer);
            }
        }

        // 显示输出，需要调整视口大小
        mFilterArrays.get(RenderIndex.DisplayIndex).drawFrame(currentTexture, mDisplayVertexBuffer, mDisplayTextureBuffer);

        return currentTexture;
    }

    /**
     * 绘制调试用的人脸关键点
     * @param mCurrentTexture
     */
    public void drawFacePoint(int mCurrentTexture) {
        if (mFilterArrays.get(RenderIndex.FacePointIndex) != null) {
            if (mCameraParam.drawFacePoints && LandmarkEngine.getInstance().hasFace()) {
                mFilterArrays.get(RenderIndex.FacePointIndex).drawFrame(mCurrentTexture, mDisplayVertexBuffer, mDisplayTextureBuffer);
            }
        }
    }

    /**
     * 设置输入纹理大小
     * @param width
     * @param height
     */
    public void setTextureSize(int width, int height) {
        mTextureWidth = width;
        mTextureHeight = height;
    }

    /**
     * 设置纹理显示大小
     * @param width
     * @param height
     */
    public void setDisplaySize(int width, int height) {
        mViewWidth = width;
        mViewHeight = height;
        adjustCoordinateSize();
        onFilterChanged();
    }

    /**
     * 调整滤镜
     */
    private void onFilterChanged() {
        for (int i = 0; i < mFilterArrays.size(); i++) {
            if (mFilterArrays.get(i) != null) {
                mFilterArrays.get(i).onInputSizeChanged(mTextureWidth, mTextureHeight);
                // 到显示之前都需要创建FBO，这里限定是防止创建多余的FBO，节省GPU资源
                if (i < RenderIndex.DisplayIndex) {
                    mFilterArrays.get(i).initFrameBuffer(mTextureWidth, mTextureHeight);
                }
                mFilterArrays.get(i).onDisplaySizeChanged(mViewWidth, mViewHeight);
            }
        }
    }

    /**
     * 调整由于surface的大小与SurfaceView大小不一致带来的显示问题
     */
    private void adjustCoordinateSize() {
        float[] textureCoord = null;
        float[] vertexCoord = null;
        float[] textureVertices = TextureRotationUtils.TextureVertices;
        float[] vertexVertices = TextureRotationUtils.CubeVertices;
        float ratioMax = Math.max((float) mViewWidth / mTextureWidth,
                (float) mViewHeight / mTextureHeight);
        // 新的宽高
        int imageWidth = Math.round(mTextureWidth * ratioMax);
        int imageHeight = Math.round(mTextureHeight * ratioMax);
        // 获取视图跟texture的宽高比
        float ratioWidth = (float) imageWidth / (float) mViewWidth;
        float ratioHeight = (float) imageHeight / (float) mViewHeight;
        if (mScaleType == ScaleType.CENTER_INSIDE) {
            vertexCoord = new float[] {
                    vertexVertices[0] / ratioHeight, vertexVertices[1] / ratioWidth, vertexVertices[2],
                    vertexVertices[3] / ratioHeight, vertexVertices[4] / ratioWidth, vertexVertices[5],
                    vertexVertices[6] / ratioHeight, vertexVertices[7] / ratioWidth, vertexVertices[8],
                    vertexVertices[9] / ratioHeight, vertexVertices[10] / ratioWidth, vertexVertices[11],
            };
        } else if (mScaleType == ScaleType.CENTER_CROP) {
            float distHorizontal = (1 - 1 / ratioWidth) / 2;
            float distVertical = (1 - 1 / ratioHeight) / 2;
            textureCoord = new float[] {
                    addDistance(textureVertices[0], distVertical), addDistance(textureVertices[1], distHorizontal),
                    addDistance(textureVertices[2], distVertical), addDistance(textureVertices[3], distHorizontal),
                    addDistance(textureVertices[4], distVertical), addDistance(textureVertices[5], distHorizontal),
                    addDistance(textureVertices[6], distVertical), addDistance(textureVertices[7], distHorizontal),
            };
        }
        if (vertexCoord == null) {
            vertexCoord = vertexVertices;
        }
        if (textureCoord == null) {
            textureCoord = textureVertices;
        }
        // 更新VertexBuffer 和 TextureBuffer
        mDisplayVertexBuffer.clear();
        mDisplayVertexBuffer.put(vertexCoord).position(0);
        mDisplayTextureBuffer.clear();
        mDisplayTextureBuffer.put(textureCoord).position(0);
    }

    /**
     * 计算距离
     * @param coordinate
     * @param distance
     * @return
     */
    private float addDistance(float coordinate, float distance) {
        return coordinate == 0.0f ? distance : 1 - distance;
    }
}
```
这里由于渲染层数是有限并且是固定的，因此使用SparseArray来存储渲染的滤镜列表，这比用Hashmap的效率要高一点。其中CameraParam是存储相机参数的单例，UI层可以通过改变CameraParam的数据调节滤镜渲染的流程。

* 预览渲染器 —— PreviewRenderer 
为了方便使用，我们将RenderThread封装到预览渲染器单例中：
```
public final class PreviewRenderer {

    private PreviewRenderer() {
        mCameraParam = CameraParam.getInstance();
    }

    private static class RenderHolder {
        private static PreviewRenderer instance = new PreviewRenderer();
    }

    public static PreviewRenderer getInstance() {
        return RenderHolder.instance;
    }

    // 相机渲染参数
    private CameraParam mCameraParam;

    // 渲染Handler
    private RenderHandler mRenderHandler;
    // 渲染线程
    private RenderThread mPreviewRenderThread;
    // 操作锁
    private final Object mSynOperation = new Object();

    private WeakReference<SurfaceView> mWeakSurfaceView;

    /**
     * 设置相机回调
     * @param callback
     * @return
     */
    public RenderBuilder setCameraCallback(OnCameraCallback callback) {
        return new RenderBuilder(this, callback);
    }

    /**
     * 初始化渲染器
     */
    void initRenderer(Context context) {
        synchronized (mSynOperation) {
            mPreviewRenderThread = new RenderThread(context, "RenderThread");
            mPreviewRenderThread.start();
            mRenderHandler = new RenderHandler(mPreviewRenderThread);
            // 绑定Handler
            mPreviewRenderThread.setRenderHandler(mRenderHandler);
        }
    }

    /**
     * 销毁渲染器
     */
    public void destroyRenderer() {
        synchronized (mSynOperation) {
            if (mWeakSurfaceView != null) {
                mWeakSurfaceView.clear();
                mWeakSurfaceView = null;
            }
            if (mRenderHandler != null) {
                mRenderHandler.removeCallbacksAndMessages(null);
                mRenderHandler = null;
            }
            if (mPreviewRenderThread != null) {
                mPreviewRenderThread.quitSafely();
                try {
                    mPreviewRenderThread.join();
                } catch (InterruptedException e) {

                }
                mPreviewRenderThread = null;
            }
        }
    }

    /**
     * 绑定需要渲染的SurfaceView
     * @param surfaceView
     */
    public void setSurfaceView(SurfaceView surfaceView) {
        mWeakSurfaceView = new WeakReference<>(surfaceView);
        surfaceView.getHolder().addCallback(mSurfaceCallback);
    }

    /**
     * Surface回调
     */
    private SurfaceHolder.Callback mSurfaceCallback = new SurfaceHolder.Callback() {
        @Override
        public void surfaceCreated(SurfaceHolder holder) {
            if (mRenderHandler != null) {
                mRenderHandler.sendMessage(mRenderHandler
                        .obtainMessage(RenderHandler.MSG_SURFACE_CREATED, holder));
            }
        }

        @Override
        public void surfaceChanged(SurfaceHolder holder, int format, int width, int height) {
            surfaceSizeChanged(width, height);
        }

        @Override
        public void surfaceDestroyed(SurfaceHolder holder) {
            if (mRenderHandler != null) {
                mRenderHandler.sendMessage(mRenderHandler
                        .obtainMessage(RenderHandler.MSG_SURFACE_DESTROYED));
            }
        }
    };

    /**
     * Surface大小发生变化
     * @param width
     * @param height
     */
    public void surfaceSizeChanged(int width, int height) {
        if (mRenderHandler != null) {
            mRenderHandler.sendMessage(mRenderHandler
                    .obtainMessage(RenderHandler.MSG_SURFACE_CHANGED, width, height));
        }
    }

    /**
     * 请求渲染
     */
    public void requestRender() {
        if (mPreviewRenderThread != null) {
            mPreviewRenderThread.requestRender();
        }
    }

    /**
     * 切换边框模糊功能
     * @param enableEdgeBlur
     */
    public void changeEdgeBlurFilter(boolean enableEdgeBlur) {
        if (mRenderHandler == null) {
            return;
        }
        synchronized (mSynOperation) {
            mRenderHandler.sendMessage(mRenderHandler
                    .obtainMessage(RenderHandler.MSG_CHANGE_EDGE_BLUR, enableEdgeBlur));
        }
    }

    /**
     * 切换滤镜
     * @param color
     */
    public void changeDynamicFilter(DynamicColor color) {
        if (mRenderHandler == null) {
            return;
        }
        synchronized (mSynOperation) {
            mRenderHandler.sendMessage(mRenderHandler
                    .obtainMessage(RenderHandler.MSG_CHANGE_DYNAMIC_COLOR, color));
        }
    }

    /**
     * 切换彩妆
     * @param makeup
     */
    public void changeDynamicMakeup(DynamicMakeup makeup) {
        if (mRenderHandler == null) {
            return;
        }
        synchronized (mSynOperation) {
            mRenderHandler.sendMessage(mRenderHandler
                    .obtainMessage(RenderHandler.MSG_CHANGE_DYNAMIC_MAKEUP, makeup));
        }
    }

    /**
     * 切换动态资源
     * @param color
     */
    public void changeDynamicResource(DynamicColor color) {
        if (mRenderHandler == null) {
            return;
        }
        synchronized (mSynOperation) {
            mRenderHandler.sendMessage(mRenderHandler
                    .obtainMessage(RenderHandler.MSG_CHANGE_DYNAMIC_RESOURCE, color));
        }
    }

    /**
     * 切换动态资源
     * @param sticker
     */
    public void changeDynamicResource(DynamicSticker sticker) {
        if (mRenderHandler == null) {
            return;
        }
        synchronized (mSynOperation) {
            mRenderHandler.sendMessage(mRenderHandler
                    .obtainMessage(RenderHandler.MSG_CHANGE_DYNAMIC_RESOURCE, sticker));
        }
    }

    /**
     * 开始录制
     */
    public void startRecording() {
        if (mRenderHandler == null) {
            return;
        }
        synchronized (mSynOperation) {
            mRenderHandler.sendMessage(mRenderHandler
                    .obtainMessage(RenderHandler.MSG_START_RECORDING));
        }
    }

    /**
     * 停止录制
     */
    public void stopRecording() {
        if (mRenderHandler == null) {
            return;
        }
        synchronized (mSynOperation) {
            mRenderHandler.sendEmptyMessage(RenderHandler.MSG_STOP_RECORDING);
        }
    }

    /**
     * 拍照
     */
    public void takePicture() {
        synchronized (mSynOperation) {
            if (!mCameraParam.isTakePicture) {
                mCameraParam.isTakePicture = true;
            }
        }
    }

    /**
     * 切换相机
     */
    public void switchCamera() {
        if (mRenderHandler == null) {
            return;
        }
        synchronized (mSynOperation) {
            mRenderHandler.sendEmptyMessage(RenderHandler.MSG_SWITCH_CAMERA);
        }
    }

    /**
     * 重新打开相机
     */
    public void reopenCamera() {
        if (mRenderHandler == null) {
            return;
        }
        synchronized (mSynOperation) {
            mRenderHandler.sendEmptyMessage(RenderHandler.MSG_REOPEN_CAMERA);
        }
    }

    /**
     * 是否需要进行对比
     * @param enable
     */
    public void enableCompare(boolean enable) {
        synchronized (mSynOperation) {
            mCameraParam.showCompare = enable;
        }
    }
}
```
这样，我们就可以在预览页面里面，通过PreviewRenderer 来控制OpenGLES 渲染参数、相机操作逻辑、以及切换滤镜、贴纸等操作逻辑了。

* 预览页面Builder封装
为了方便进入预览页面时，传入预览的宽高比、是否显示调试关键点等参数，我们通过Builder 模式来控制：
```
public final class PreviewBuilder {

    private PreviewEngine mPreviewEngine;
    private CameraParam mCameraParam;

    public PreviewBuilder(PreviewEngine engine, AspectRatio ratio) {
        mPreviewEngine = engine;
        mCameraParam = CameraParam.getInstance();
        mCameraParam.setAspectRatio(ratio);
    }

    /**
     * 是否显示人脸关键点
     * @param show
     * @return
     */
    public PreviewBuilder showFacePoints(boolean show) {
        mCameraParam.drawFacePoints = show;
        return this;
    }

    /**
     * 是否显示fps
     * @param show
     * @return
     */
    public PreviewBuilder showFps(boolean show) {
        mCameraParam.showFps = show;
        return this;
    }

    /**
     * 期望预览帧率
     * @param fps
     * @return
     */
    public PreviewBuilder expectFps(int fps) {
        mCameraParam.expectFps = fps;
        return this;
    }

    /**
     * 期望宽度
     * @param width
     * @return
     */
    public PreviewBuilder expectWidth(int width) {
        mCameraParam.expectWidth = width;
        return this;
    }

    /**
     * 期望高度
     * @param height
     * @return
     */
    public PreviewBuilder expectHeight(int height) {
        mCameraParam.expectHeight = height;
        return this;
    }

    /**
     * 是否高清拍照
     * @param highDefinition
     * @return
     */
    public PreviewBuilder highDefinition(boolean highDefinition) {
        mCameraParam.highDefinition = highDefinition;
        return this;
    }

    /**
     * 是否打开后置摄像头
     * @param backCamera
     * @return
     */
    public PreviewBuilder backCamera(boolean backCamera) {
        mCameraParam.backCamera = backCamera;
        if (mCameraParam.backCamera) {
            mCameraParam.cameraId = Camera.CameraInfo.CAMERA_FACING_BACK;
        }
        return this;
    }

    /**
     * 对焦权重
     * @param weight
     * @return
     */
    public PreviewBuilder focusWeight(int weight) {
        mCameraParam.setFocusWeight(weight);
        return this;
    }

    /**
     * 是否允许录制
     * @param recordable
     * @return
     */
    public PreviewBuilder recordable(boolean recordable) {
        mCameraParam.recordable = recordable;
        return this;
    }

    /**
     * 录制时间
     * @param recordTime
     * @return
     */
    public PreviewBuilder recordTime(int recordTime) {
        mCameraParam.recordTime = recordTime;
        return this;
    }

    /**
     * 是否录制音频
     * @param recordAudio
     * @return
     */
    public PreviewBuilder recordAudio(boolean recordAudio) {
        mCameraParam.recordAudio = recordAudio;
        return this;
    }

    /**
     * 延时拍摄
     * @param takeDelay
     * @return
     */
    public PreviewBuilder takeDelay(boolean takeDelay) {
        mCameraParam.takeDelay = takeDelay;
        return this;
    }

    /**
     * 是否开启夜光补偿
     * @param luminousEnhancement
     * @return
     */
    public PreviewBuilder luminousEnhancement(boolean luminousEnhancement) {
        mCameraParam.luminousEnhancement = luminousEnhancement;
        return this;
    }

    /**
     * 设置拍照监听器
     * @param listener
     * @return
     */
    public PreviewBuilder setPreviewCaptureListener(OnPreviewCaptureListener listener) {
        mCameraParam.captureListener = listener;
        return this;
    }

    /**
     *
     * @param listener
     * @return
     */
    public PreviewBuilder setGalleryListener(OnGallerySelectedListener listener) {
        mCameraParam.gallerySelectedListener = listener;
        return this;
    }

    /**
     * 打开预览
     * @param requestCode
     */
    public void startPreviewForResult(int requestCode) {
        Activity activity = mPreviewEngine.getActivity();
        if (activity == null) {
            return;
        }
        Intent intent = new Intent(activity, CameraActivity.class);
        Fragment fragment = mPreviewEngine.getFragment();
        if (fragment != null) {
            fragment.startActivityForResult(intent, requestCode);
        } else {
            activity.startActivityForResult(intent, requestCode);
        }
    }

    /**
     * 打开预览
     */
    public void startPreview() {
        Activity activity = mPreviewEngine.getActivity();
        if (activity == null) {
            return;
        }
        Intent intent = new Intent(activity, CameraActivity.class);
        Fragment fragment = mPreviewEngine.getFragment();
        if (fragment != null) {
            fragment.startActivity(intent);
        } else {
            activity.startActivity(intent);
        }
    }
}
```
然后我们暴露一个预览引擎对象：
```
public final class PreviewEngine {

    private WeakReference<Activity> mWeakActivity;
    private WeakReference<Fragment> mWeakFragment;

    private PreviewEngine(Activity activity) {
        this(activity, null);
    }

    private PreviewEngine(Fragment fragment) {
        this(fragment.getActivity(), fragment);
    }

    private PreviewEngine(Activity activity, Fragment fragment) {
        mWeakActivity = new WeakReference<>(activity);
        mWeakFragment = new WeakReference<>(fragment);
    }

    public static PreviewEngine from(Activity activity) {
        return new PreviewEngine(activity);
    }

    public static PreviewEngine from(Fragment fragment) {
        return new PreviewEngine(fragment);
    }

    /**
     * 设置长宽比
     * @param ratio
     * @return
     */
    public PreviewBuilder setCameraRatio(AspectRatio ratio) {
        return new PreviewBuilder(this, ratio);
    }

    public Activity getActivity() {
        return mWeakActivity.get();
    }

    public Fragment getFragment() {
        return mWeakFragment.get();
    }
}
```
* 打开预览页面的调用方式
这样，我们就可以在使用的时候打开预览页面时传入不同的预览参数配置，使用如下所示：
```
PreviewEngine.from(this)
                .setCameraRatio(AspectRatio.Ratio_16_9)
                .showFacePoints(false)
                .showFps(true)
                .setGalleryListener(new OnGallerySelectedListener() {
                    @Override
                    public void onGalleryClickListener(GalleryType type) {
                        scanMedia(type == GalleryType.ALL);
                    }
                })
                .setPreviewCaptureListener(new OnPreviewCaptureListener() {
                    @Override
                    public void onMediaSelectedListener(String path, GalleryType type) {
                        if (type == GalleryType.PICTURE) {
                            Intent intent = new Intent(MainActivity.this, ImageEditActivity.class);
                            intent.putExtra(ImageEditActivity.PATH, path);
                            startActivity(intent);
                        } else if (type == GalleryType.VIDEO) {
                            Intent intent = new Intent(MainActivity.this, VideoEditActivity.class);
                            intent.putExtra(VideoEditActivity.PATH, path);
                            startActivity(intent);
                        }
                    }
                })
                .startPreview();
```
预览整体流程，这里就介绍完了。
详细实现过程请参照本人的项目：
[CainCamera](https://github.com/CainKernel/CainCamera)
