最近本人对自己的相机项目(https://github.com/CainKernel/CainCamera) 做了优化，使得实时渲染的帧率能够得到明显的提升。在此，本人说说优化的一些思路吧。

1、为什么实时渲染跟onPreviewFrame相关？ 
其实这个跟实时渲染是没太大关系的，主要的原因是——目前的所有人脸检测的SDK都是通过byte[] 的图像数据来做人脸检测的，而glReadPixels的实现实在是太长了，一般都要100毫秒以上，通过GPU读取图像数据实在是没什么好的办法。也就是说，要使用人脸检测SDK，目前Android平台最好的办法就是通过回调的onPreviewFrame提取图像数据。如果没有人脸检测的需求，实时渲染可以通过onFrameAvailable回调直接更新渲染，效率会更高。由于目前的人脸检测SDK的影响，我们就需要保证onPrevieFrame 的帧率，onPreviewFrame 的帧率高低，直接影响了预览画面是否卡顿，录制出来的视频帧率是否足够。基本上对于相机类应用来说，是最核心的部分。

2、对比市面上的相机
目前市面上的主流美颜类相机，在实时渲染上采取的方案都差不多，大多都是通过自定义SurfaceView来实现的，预览帧率方面则差距比较大，尤其是在MTK的CPU上表现并不一致。至于为什么在MTK的CPU上表现不一致，请参考本人的文章——《关于Android Camera onPreviewFrame 预览回调帧率问题》(http://www.jianshu.com/p/a33b1eabe71c)，在这篇文章上面，采用单HandlerThread 和双HandlerThread 在高通CPU和MTK CPU上的onPreviewFrame帧率的差异，可以大体上猜测出市面上的主流相机的方案。在只做磨皮算法的情况下，主流相机的表现如下:
测试设备信息：
设备名称：红米Note2
CPU： 联发科 Helio X10（MT6795）

测试应用以及帧率如下：
美颜相机——渲染的Texture大小 960 x 540，SurfaceView的帧率 7 ~ 13帧，比较卡：
![美颜相机_960x540P.png](http://upload-images.jianshu.io/upload_images/2103804-16402e3903dec8cb.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
FaceU激萌——渲染的Texture大小1280 x 720， SufaceView的帧率15~19帧，偶尔画面不够连贯：
![FaceU_1280x720P.png](http://upload-images.jianshu.io/upload_images/2103804-4c7387071ea204d2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
天天P图——渲染的Texture大小1280 x 720， SurfaceView的帧率13 ~ 19帧，偶尔画面不够连贯：
![天天P图_1280x720P.png](http://upload-images.jianshu.io/upload_images/2103804-45ecc4e2d9b8c0d4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
无他相机 —— 渲染的Texture大小 1280 x 720， SurfaceView的帧率 17 ~ 19帧，偶尔画面不够连贯：
![无他相机_1280x720P.png](http://upload-images.jianshu.io/upload_images/2103804-59de831f6ef2b5a4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

测试结论：
从上面可以看到，在关闭所有特效，仅仅开启磨皮算法时，各个应用的帧率表现不一，大多数应用的帧率在红米Note2的表现都没有超过20帧的，底层的BufferQueueProducer产生的帧率也勉强达到20帧。这里有个例外，那就是美颜相机的帧率过低了。根据我之前那篇文章，可以猜测美颜相机是采用了双HandlerThread的方式进行渲染的，一个HandlerThread 用来控制相机的打开关闭等操作，一个HandlerThread用来控制渲染各种特效。因为美颜相机在高通的CPU上的帧率是比较正常的，跟其他的相机帧率都在同一个范围内，而在MTK的CPU上帧率却只有一半左右。大部分相机类的APP在红米Note2上的表现都是偶尔画面不够连贯。这个产生的问题主要还是红米Note2 的CPU 和 GPU确实有点落后了，渲染的效率跟不上。在Nexus 5X 、高通808CPU上，以上的各个应用最高值都是超过20帧的，并且帧率相对比较稳定，画面也并没有出现不连贯的情况。

3、怎么优化？
基于前面测试得到的结论，我们想想如何做渲染的优化？在之前的文章里面我们讨论过单双HandlerThread对MTKCPU的onPreviewFrame帧率的影响，一个HandlerThread控制Camera操作，一个HandlerThread 控制渲染的方案很明显是不行的，onPreviewFrame的帧率过低，无法满足实时性要求的，因此我们把相机的控制逻辑以及渲染逻辑都通过一个HandlerThread线程来进行控制。另外方面，渲染次数的控制。比如在录制视频时，不要重复渲染已经渲染过的特效，可以通过在特效渲染完成后，取出texture 的 id，然后单独做添加水印的渲染过程，此时我们只需要绘制一帧所需要的操作就是渲染特效 + 渲染水印，如果在录制视频时再次绘制渲染特效，则此时绘制一帧所需要的操作就是 渲染特效 + (录制视频)渲染特效 + 渲染水印，这样的话，在录制视频的时候，帧率会明显下降。

4、实现过程
基于上面的思路，我们就可以对实时渲染部分进行优化了。首先放渲染优化的结果：
测试设备信息：
设备名称：红米Note2
CPU： 联发科 Helio X10（MT6795）

测试应用及帧率如下：
CainCamera  —— 渲染的Texture大小 1280 x 720，SurfaceView的帧率稳定保证在19帧以上，偶尔画面不够连贯，比市面上的所有相机APP不连贯感要弱很多：
![CainCamera_1280x720P.png](http://upload-images.jianshu.io/upload_images/2103804-35f6e44a1fbfd109.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

优化过程：
为了方便控制和解耦，本人预览实现渲染部分分成了几个部分：
DrawerManager：Activity以及SurfaceView 等调用的管理类，用于控制绘制线程RenderThread和RenderHandler回调
RenderThread： 渲染线程，所有的预览、拍照、录制视频等操作均由RenderThread控制
RenderHandler：渲染线程回调 
RenderManager：渲染管理器，渲染实体，用于管理渲染预览还是渲染录制的视频等操作
FilterManager：用于管理不同的滤镜，用于返回一个具体的滤镜或者滤镜组
CameraUtils： Camera控制部分，用于打开关闭相机等操作的封装

接下来将会介绍一下主要的实现过程，具体的实现过程请参考本人的开源相机项目 CainCamera：
https://github.com/CainKernel/CainCamera

4.1、SurfaceView 的实现：
```
public class CameraSurfaceView extends SurfaceView implements SurfaceHolder.Callback {

    private static final String TAG = "CameraSurfaceView";

    // 触摸点击默认值，触摸差值小于该值算点击事件
    private final static float MAX_TOUCH_SIZE_FOR_CLICK = 15f;
    // 小于该时间间隔则表示双击
    private final static int MAX_DOUBLE_CLICK_INTERVAL = 200;
    // 点击时间有效时间
    private final static int MAX_CLICK_INTERVAL = 100;

    private float mTouchPreviewX = 0;
    private float mTouchPreviewY = 0;
    private float mTouchUpX = 0;
    private float mTouchUpY = 0;

    private final Object mOperation = new Object();
    private boolean mTouchWithoutSwipe = false;

    private OnTouchScroller mScroller;
    private OnClickListener mClickListener;

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
        DrawerManager.getInstance().surfaceCreated(holder);
    }

    @Override
    public void surfaceChanged(SurfaceHolder holder, int format, int width, int height) {
        DrawerManager.getInstance().surfacrChanged(width, height);
    }

    @Override
    public void surfaceDestroyed(SurfaceHolder holder) {
        DrawerManager.getInstance().surfaceDestroyed();
    }


    private boolean mIsWaitUpEvent = false;
    private boolean mIsWaitDoubleClick = false;
    @Override
    public boolean onTouchEvent(MotionEvent event) {

        switch (event.getAction()) {
            case MotionEvent.ACTION_DOWN:
                mTouchPreviewX = event.getX();
                mTouchPreviewY = event.getY();
                mIsWaitUpEvent = true;
                postDelayed(mTimerForUpEvent, MAX_CLICK_INTERVAL);
                break;

            case MotionEvent.ACTION_MOVE:
                if (Math.abs(event.getX() - mTouchPreviewX) > MAX_TOUCH_SIZE_FOR_CLICK
                        || Math.abs(mTouchPreviewY) > MAX_TOUCH_SIZE_FOR_CLICK) {
                    mIsWaitUpEvent = false;
                    removeCallbacks(mTimerForUpEvent);
                }
                break;

            case MotionEvent.ACTION_UP:
                mTouchUpX = event.getX();
                mTouchUpY = event.getY();
                mIsWaitUpEvent = false;
                removeCallbacks(mTimerForUpEvent);
                if (Math.abs(event.getX() - mTouchPreviewX) < MAX_TOUCH_SIZE_FOR_CLICK
                        && Math.abs(event.getY() - mTouchPreviewY) < MAX_TOUCH_SIZE_FOR_CLICK) {
                    synchronized (mOperation) {
                        mTouchWithoutSwipe = true;
                    }
                    processClickEvent();
                    synchronized (mOperation) {
                        mTouchWithoutSwipe = false;
                    }
                } else {
                    synchronized (mOperation) {
                        if (!mTouchWithoutSwipe) {
                            processTouchMovingEvent(event);
                        }
                    }
                }
                break;

            case MotionEvent.ACTION_CANCEL:
                mIsWaitUpEvent = false;
                removeCallbacks(mTimerForUpEvent);
                break;
        }

        return true;
    }

    /**
     * 等待松手
     */
    private Runnable mTimerForUpEvent = new Runnable() {
        @Override
        public void run() {
            if (mIsWaitUpEvent) {
                mIsWaitUpEvent = false;
            } else {

            }
        }
    };

    /**
     * 等待是否存在第二次点击事件
     */
    private Runnable mTimerForSecondClick = new Runnable() {
        @Override
        public void run() {
            if (mIsWaitDoubleClick) {
                mIsWaitDoubleClick = false;
                if (mClickListener != null) {
                    mClickListener.onClick(mTouchUpX, mTouchUpY);
                }
            }
        }
    };

    /**
     * 点击事件
     */
    private void processClickEvent() {
        if (mIsWaitDoubleClick) {
            if (mClickListener != null) {
                mClickListener.doubleClick(mTouchUpX, mTouchUpY);
            }
            mIsWaitDoubleClick = false;
            removeCallbacks(mTimerForSecondClick);
        } else {
            mIsWaitDoubleClick = true;
            postDelayed(mTimerForSecondClick, MAX_DOUBLE_CLICK_INTERVAL);
        }
    }

    /**
     * 触摸滑动事件
     * @param event
     */
    private void processTouchMovingEvent(MotionEvent event) {
        if (Math.abs(event.getX() - mTouchPreviewX)
                > Math.abs(event.getY() - mTouchPreviewY) * 1.5) {
            if (event.getX() - mTouchPreviewX < 0) {
                if (mScroller != null) {
                    mScroller.swipeBack();
                }
            } else {
                if (mScroller != null) {
                    mScroller.swipeFrontal();
                }
            }
        } else if (Math.abs(event.getY() - mTouchPreviewY)
                > Math.abs(event.getX() - mTouchPreviewX) * 1.5) {
            if (event.getY() - mTouchPreviewY < 0) {
                if (mScroller != null) {
                    if (mTouchPreviewX < getWidth() / 2) {
                        mScroller.swipeUpper(true);
                    } else {
                        mScroller.swipeUpper(false);
                    }
                }
            } else {
                if (mScroller != null) {
                    if (mTouchPreviewX < getWidth() / 2) {
                        mScroller.swipeDown(true);
                    } else {
                        mScroller.swipeDown(false);
                    }
                }
            }
        }
    }

    /**
     * 添加滑动回调
     * @param scroller
     */
    public void addScroller(OnTouchScroller scroller) {
        mScroller = scroller;
    }

    /**
     * 滑动监听器
     */
    public interface OnTouchScroller {
        void swipeBack();
        void swipeFrontal();
        void swipeUpper(boolean startInLeft);
        void swipeDown(boolean startInLeft);
    }

    /**
     * 添加点击事件回调
     * @param listener
     */
    public void addClickListener(OnClickListener listener) {
        mClickListener = listener;
    }

    /**
     * 点击事件监听器
     */
    public interface OnClickListener {
        void onClick(float x, float y);
        void doubleClick(float x, float y);
    }
}
```
以上代码实现了自定义SurfaceView，以及单击、双击操作、左滑、右滑、上滑、下滑等操作。实现这些操作主要是为了方便在预览页面进行左右滑动切换滤镜、上下滑动可以改变相机光照亮度(暂未实现)、单击改变对焦等功能。

DrawerManager 的实现如下：
```
public class DrawerManager {

    private static final String TAG = "DrawerManager";

    private static DrawerManager mInstance;

    private RenderHandler mRenderHandler;
    private RenderThread mRenderThread;

    // 操作锁
    private final Object mSynOperation = new Object();

    public static DrawerManager getInstance() {
        if (mInstance == null) {
            mInstance = new DrawerManager();
        }
        return mInstance;
    }

    private DrawerManager() {

    }

    public void surfaceCreated(SurfaceHolder holder) {
        create();
        if (mRenderHandler != null) {
            mRenderHandler.sendMessage(mRenderHandler
                    .obtainMessage(RenderHandler.MSG_SURFACE_CREATED, holder));
        }
    }

    public void surfacrChanged(int width, int height) {
        if (mRenderHandler != null) {
            mRenderHandler.sendMessage(mRenderHandler
                    .obtainMessage(RenderHandler.MSG_SURFACE_CHANGED, width, height));
        }
        startPreview();
    }

    public void surfaceDestroyed() {
        stopPreview();
        if (mRenderHandler != null) {
            mRenderHandler.sendMessage(mRenderHandler
                    .obtainMessage(RenderHandler.MSG_SURFACE_DESTROYED));
        }
        destory();
    }

    /**
     * 创建HandlerThread和Handler
     */
    synchronized private void create() {
        mRenderThread = new RenderThread("RenderThread");
        mRenderThread.start();
        mRenderHandler = new RenderHandler(mRenderThread.getLooper(), mRenderThread);
        // 绑定Handler
        mRenderThread.setRenderHandler(mRenderHandler);
    }


    /**
     * 销毁当前持有的Looper 和 Handler
     */
    synchronized private void destory() {
        // Handler不存在时，需要销毁当前线程，否则可能会出现重新打开不了的情况
        if (mRenderHandler == null) {
            if (mRenderThread != null) {
                mRenderThread.quitSafely();
                try {
                    mRenderThread.join();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                mRenderThread = null;
            }
            return;
        }
        mRenderHandler.sendEmptyMessage(RenderHandler.MSG_DESTROY);
        mRenderThread.quitSafely();
        try {
            mRenderThread.join();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        mRenderThread = null;
        mRenderHandler = null;
    }

    /**
     * 开始预览
     */
    public void startPreview() {
        if (mRenderHandler == null) {
            return;
        }
        synchronized (mSynOperation) {
            mRenderHandler.sendMessage(mRenderHandler
                    .obtainMessage(RenderHandler.MSG_START_PREVIEW));
        }
    }

    /**
     * 停止预览
     */
    public void stopPreview() {
        if (mRenderHandler == null) {
            return;
        }
        synchronized (mSynOperation) {
            mRenderHandler.sendMessage(mRenderHandler
                    .obtainMessage(RenderHandler.MSG_STOP_PREVIEW));
        }
    }

    /**
     * 设置触摸区域
     * @param rect 已经归整到(-1000, -1000)~(1000, 1000) 的区域
     */
    public void setFocusAres(Rect rect) {
        if (mRenderHandler == null) {
            return;
        }
        synchronized (mSynOperation) {
            mRenderHandler.sendMessage(mRenderHandler
                    .obtainMessage(RenderHandler.MSG_FOCUS_RECT, rect));
        }
    }

    /**
     * 改变Filter类型
     */
    public void changeFilterType(FilterType type) {
        if (mRenderHandler == null) {
            return;
        }
        synchronized (mSynOperation) {
            mRenderHandler.sendMessage(mRenderHandler
                    .obtainMessage(RenderHandler.MSG_FILTER_TYPE, type));
        }
    }

    /**
     * 改变滤镜组类型
     * @param type
     */
    public void changeFilterGroup(FilterGroupType type) {
        if (mRenderHandler == null) {
            return;
        }
        synchronized (mSynOperation) {
            mRenderHandler.sendMessage(mRenderHandler
                    .obtainMessage(RenderHandler.MSG_FILTER_GROUP, type));
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
            mRenderHandler.sendMessage(mRenderHandler
                    .obtainMessage(RenderHandler.MSG_SWITCH_CAMERA));

        }
        // 开始预览
        startPreview();
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
        if (mRenderHandler == null) {
            return;
        }
        synchronized (mSynOperation) {
            // 发送拍照命令
            mRenderHandler.sendMessage(mRenderHandler
                    .obtainMessage(RenderHandler.MSG_TAKE_PICTURE));
        }
    }
}
```
DrawerManager 用来操作渲染线程和回调的，用来控制渲染线程的打开关闭、开始预览、停止预览、改变滤镜、改变滤镜组、切换相机、开始录制、停止录制等功能，主要是给Activity等外界调用的，所有相机相关的操作逻辑、渲染逻辑通过Handler回调控制。

接下来，我们看看RenderThread 的实现：
```
public class RenderThread extends HandlerThread implements SurfaceTexture.OnFrameAvailableListener,
        Camera.PreviewCallback {

    private static final String TAG = "RenderThread";

    // 操作锁
    private final Object mSynOperation = new Object();
    // Looping锁
    private final Object mSyncIsLooping = new Object();

    private boolean isPreviewing = false;   // 是否预览状态
    private boolean isRecording = false;    // 是否录制状态

    // EGL共享上下文
    private EglCore mEglCore;
    // EGLSurface
    private WindowSurface mDisplaySurface;

    // 录制视频用的EGLSurface
    private WindowSurface mRecordWindowSurface;
    private VideoEncoderCore mVideoEncoder;

    // CameraTexture对应的Id
    private int mCameraTextureId;
    private SurfaceTexture mCameraTexture;

    // 矩阵
    private final float[] mMatrix = new float[16];

    // 视图宽高
    private int mViewWidth, mViewHeight;
    // 预览图片大小
    private int mImageWidth, mImageHeight;

    // 是否处于检测阶段
    private boolean isFaceDetecting = false;

    // 更新帧的锁
    private final Object mSyncFrameNum = new Object();
    private final Object mSyncTexture = new Object();
    // 可用帧
    private int mFrameNum = 0;
    // 拍照
    private boolean isTakePicture = false;

    // 录制比特率
    private int mRecordBitrate;
    // 录制帧率
    private static final int FRAME_RATE = 25;
    // 录制质量
    private static final int BPP = 4;

    // 关键点绘制（调试用）
    private FacePointsDrawer mFacePointsDrawer;

    // 预览回调缓存，解决previewCallback回调内存抖动问题
    private byte[] mPreviewBuffer;

    private RenderHandler mRenderHandler;

    public RenderThread(String name) {
        super(name);
    }

    public void setRenderHandler(RenderHandler handler) {
        mRenderHandler = handler;
    }

    @Override
    public void onFrameAvailable(SurfaceTexture surfaceTexture) {

    }

    private long time = 0;
    @Override
    public void onPreviewFrame(final byte[] data, Camera camera) {
        addNewFrame();
        if (mRenderHandler != null) {
            synchronized (mSynOperation) {
                if (isPreviewing || isRecording) {
                    mRenderHandler.sendMessage(mRenderHandler
                            .obtainMessage(RenderHandler.MSG_PREVIEW_CALLBACK, data));
                }
            }
        }
        if (mPreviewBuffer != null) {
            camera.addCallbackBuffer(mPreviewBuffer);
        }
        Log.d("onPreviewFrame", "update time = " + (System.currentTimeMillis() - time));
        time = System.currentTimeMillis();
    }

    void surfaceCreated(SurfaceHolder holder) {
        mEglCore = new EglCore(null, EglCore.FLAG_RECORDABLE);
        mDisplaySurface = new WindowSurface(mEglCore, holder.getSurface(), false);
        mDisplaySurface.makeCurrent();
        mCameraTextureId = GlUtil.createTextureOES();
        mCameraTexture = new SurfaceTexture(mCameraTextureId);
        mCameraTexture.setOnFrameAvailableListener(this);
        // 打开相机
        CameraUtils.openCamera(CameraUtils.DESIRED_PREVIEW_FPS);
        // 设置预览Surface
        CameraUtils.setPreviewSurface(mCameraTexture);
        calculateImageSize();

        // 渲染初始化
        RenderManager.getInstance().init();
        RenderManager.getInstance().onInputSizeChanged(mImageWidth, mImageHeight);

        // 禁用深度测试和背面绘制
        GLES30.glDisable(GLES30.GL_DEPTH_TEST);
        GLES30.glDisable(GLES30.GL_CULL_FACE);
        // 添加预览回调以及回调buffer，用于人脸检测
        initPreviewCallback();
        // 初始化人脸检测工具
        initFaceDetection();
    }

    void surfaceChanged(int width, int height) {
        mViewWidth = width;
        mViewHeight = height;
        onFilterChanged();
        RenderManager.getInstance().adjustViewSize();
        RenderManager.getInstance().updateTextureBuffer();
        RenderManager.getInstance().onDisplaySizeChanged(mViewWidth, mViewHeight);
        // 开始预览
        CameraUtils.startPreview();
        // 渲染视图变化
        RenderManager.getInstance().onDisplaySizeChanged(mViewWidth, mViewHeight);
        isPreviewing = true;
    }

    void surfaceDestoryed() {
        isPreviewing = false;
        CameraUtils.releaseCamera();
        FaceManager.getInstance().destory();
        if (mCameraTexture != null) {
            mCameraTexture.release();
            mCameraTexture = null;
        }
        if (mDisplaySurface != null) {
            mDisplaySurface.release();
            mDisplaySurface = null;
        }
        if (mEglCore != null) {
            mEglCore.release();
            mEglCore = null;
        }
        if (mFacePointsDrawer != null) {
            mFacePointsDrawer.release();
            mFacePointsDrawer = null;
        }
        RenderManager.getInstance().release();
    }

    /**
     * 初始化预览回调
     * 备注：在某些设备上，需要在setPreviewTexture之后，startPreview之前添加回调才能使得onPreviewFrame回调正常
     */
    private void initPreviewCallback() {
        Size previewSize = CameraUtils.getPreviewSize();
        int size = previewSize.getWidth() * previewSize.getHeight() * 3 / 2;
        mPreviewBuffer = new byte[size];
        CameraUtils.setPreviewCallbackWithBuffer(this, mPreviewBuffer);
    }

    /**
     * 初始化人脸检测工具
     */
    private void initFaceDetection() {
        FaceManager.getInstance().createHandleThread();
        if (ParamsManager.canFaceTrack) {
            FaceManager.getInstance().initFaceConfig(mImageWidth, mImageHeight);
            FaceManager.getInstance().getFaceDetector().setBackCamera(CameraUtils.getCameraID()
                    == Camera.CameraInfo.CAMERA_FACING_BACK);
            addDetectorCallback();
            if (ParamsManager.enableDrawingPoints) {
                mFacePointsDrawer = new FacePointsDrawer();
            }
        }
    }

    /**
     * 预览回调
     * @param data
     */
    void onPreviewCallback(byte[] data) {
        if (isFaceDetecting)
            return;
        isFaceDetecting = true;
        if (ParamsManager.canFaceTrack) {
            synchronized (mSyncIsLooping) {
                FaceManager.getInstance().faceDetecting(data, mImageHeight, mImageWidth);
            }
        }
        isFaceDetecting = false;
    }


    /**
     * 开始预览
     */
    void startPreview() {
        isPreviewing = true;
        if (mCameraTexture != null) {
            RenderManager.getInstance().updateTextureBuffer();
            CameraUtils.setPreviewSurface(mCameraTexture);
            initPreviewCallback();
            CameraUtils.startPreview();
        }

    }

    /**
     * 停止预览
     */
    void stopPreview() {
        isPreviewing = false;
        CameraUtils.stopPreview();
    }

    /**
     * 计算imageView 的宽高
     */
    private void calculateImageSize() {
        Size size = CameraUtils.getPreviewSize();
        CameraInfo info = CameraUtils.getCameraInfo();
        if (info != null) {
            if (info.getOrientation() == 90 || info.getOrientation() == 270) {
                mImageWidth = size.getHeight();
                mImageHeight = size.getWidth();
            } else {
                mImageWidth = size.getWidth();
                mImageHeight = size.getHeight();
            }
        }
    }


    /**
     * 设置对焦区域
     * @param rect
     */
    void setFocusAres(Rect rect) {
        CameraUtils.setFocusArea(rect, null);
    }

    /**
     * 滤镜或视图发生变化时调用
     */
    private void onFilterChanged() {
        RenderManager.getInstance().onFilterChanged();
    }

    /**
     * 更新filter
     * @param type Filter类型
     */
    void changeFilter(FilterType type) {
        RenderManager.getInstance().changeFilter(type);
    }

    /**
     * 切换滤镜组
     * @param type
     */
    void changeFilterGroup(FilterGroupType type) {
        synchronized (mSyncIsLooping) {
            RenderManager.getInstance().changeFilterGroup(type);
        }
    }

    /**
     * 绘制帧
     */
    void drawFrame() {
        // 如果存在新的帧，则更新帧
        synchronized (mSyncFrameNum) {
            synchronized (mSyncTexture) {
                if (mCameraTexture != null) {
                    while (mFrameNum != 0) {
                        mCameraTexture.updateTexImage();
                        --mFrameNum;
                    }
                } else {
                    return;
                }
            }
        }
        // 切换渲染上下文
        mDisplaySurface.makeCurrent();
        mCameraTexture.getTransformMatrix(mMatrix);
        RenderManager.getInstance().setTextureTransformMatirx(mMatrix);
        // 绘制
        draw();
        // 拍照状态
        if (isTakePicture) {
            isTakePicture = false;
            File file = new File(ParamsManager.ImagePath + "CainCamera_"
                    + System.currentTimeMillis() + ".jpeg");
            if (!file.getParentFile().exists()) {
                file.getParentFile().mkdirs();
            }
            try {
                mDisplaySurface.saveFrame(file);
            } catch (IOException e) {
                Log.w(TAG, "saceFrame error: " + e.toString());
            }
        }
        mDisplaySurface.swapBuffers();

        // 是否处于录制状态
        if (isRecording) {
            mRecordWindowSurface.makeCurrent();
            mVideoEncoder.drainEncoder(false);
            // draw();
            RenderManager.getInstance().drawRecordingFrame();
            mRecordWindowSurface.setPresentationTime(mCameraTexture.getTimestamp());
            mRecordWindowSurface.swapBuffers();
        }
    }

    /**
     * 添加检测回调
     */
    private void addDetectorCallback() {
        FaceManager.getInstance().getFaceDetector()
                .addDetectorCallback(new DetectorCallback() {
                    @Override
                    public void onTrackingFinish(boolean hasFaces) {
                        // 如果有人脸并且允许绘制关键点，则添加数据
                        if (hasFaces) {
                            if (ParamsManager.enableDrawingPoints && mFacePointsDrawer != null) {
                                mFacePointsDrawer.addPoints(FaceManager.getInstance()
                                        .getFaceDetector().getFacePoints());
                            }
                        } else {
                            if (ParamsManager.enableDrawingPoints && mFacePointsDrawer != null) {
                                mFacePointsDrawer.addPoints(null);
                            }
                        }
                    }
                });
    }

    /**
     * 绘制图像数据到FBO
     */
    private void draw() {
        // 绘制
        RenderManager.getInstance().drawFrame(mCameraTextureId);

        // 是否绘制点
        if (ParamsManager.enableDrawingPoints && mFacePointsDrawer != null) {
            mFacePointsDrawer.drawPoints();
        }
    }


    /**
     * 拍照
     */
    void takePicture() {
        isTakePicture = true;
    }

    /**
     * 开始录制
     */
    void startRecording() {
        File file = new File(ParamsManager.VideoPath
                + "CainCamera_" + System.currentTimeMillis() + ".mp4");
        if (!file.getParentFile().exists()) {
            file.getParentFile().mkdirs();
        }
        // 计算帧率
        mRecordBitrate = mViewWidth * mViewHeight * FRAME_RATE / BPP;
        try {
            mVideoEncoder = new VideoEncoderCore(mViewWidth, mViewHeight,
                    mRecordBitrate, file);
        } catch (IOException e) {
            e.printStackTrace();
        }
        mRecordWindowSurface = new WindowSurface(mEglCore,
                mVideoEncoder.getInputSurface(), true);
        isRecording = true;
        RenderManager.getInstance().initRecordingFilter();
    }

    /**
     * 停止录制
     */
    void stopRecording() {
        synchronized (mSyncIsLooping) {
            mVideoEncoder.drainEncoder(true);
        }
        isRecording = false;
        RenderManager.getInstance().releaseRecordingFilter();
        // 录制完成需要释放资源
        if (mVideoEncoder != null) {
            mVideoEncoder.release();
            mVideoEncoder = null;
        }
        if (mRecordWindowSurface != null) {
            mRecordWindowSurface.release();
            mRecordWindowSurface = null;
        }
    }

    /**
     * 切换相机
     */
    void switchCamera() {
        CameraUtils.switchCamera(1 - CameraUtils.getCameraID(), mCameraTexture,
                this, mPreviewBuffer);
    }

    /**
     * 添加新的一帧
     */
    private void addNewFrame() {
        synchronized (mSyncFrameNum) {
            if (isPreviewing) {
                ++mFrameNum;
                if (mRenderHandler != null) {
                    mRenderHandler.removeMessages(RenderHandler.MSG_FRAME);
                    mRenderHandler.sendMessageAtFrontOfQueue(mRenderHandler
                            .obtainMessage(RenderHandler.MSG_FRAME));
                }
            }
        }
    }
}
```
RenderThread 通过继承HandlerThread，用于控制相机打开、关闭、切换、预览回调、调用RenderManager渲染特效、渲染录制特效的操作。

RenderManager的实现如下：
```
public final class RenderManager {

    private static final String TAG = "RenderManager";

    private static RenderManager mInstance;

    private static Object mSyncObject = new Object();

    // 相机输入流滤镜
    private CameraFilter mCameraFilter;
    // 实时滤镜组
    private BaseImageFilterGroup mRealTimeFilter;

    // 当前的TextureId
    private int mCurrentTextureId;
    private DisplayFilter mRecordFilter;

    // 输入流大小
    private int mTextureWidth;
    private int mTextureHeight;
    // 显示大小
    private int mDisplayWidth;
    private int mDisplayHeight;

    private ScaleType mScaleType = ScaleType.CENTER_CROP;
    private FloatBuffer mVertexBuffer;
    private FloatBuffer mTextureBuffer;

    public static RenderManager getInstance() {
        if (mInstance == null) {
            mInstance = new RenderManager();
        }
        return mInstance;
    }

    private RenderManager() {
    }

    /**
     * 初始化
     */
    public void init() {
        // 释放之前的滤镜
        releaseFilters();
        releaseBuffers();
        initBuffers();
        // 初始化滤镜
        initFilters();
    }

    /**
     * 初始化缓冲区
     */
    private void initBuffers() {
        mVertexBuffer = ByteBuffer
                .allocateDirect(TextureRotationUtils.CubeVertices.length * 4)
                .order(ByteOrder.nativeOrder())
                .asFloatBuffer();
        mVertexBuffer.put(TextureRotationUtils.CubeVertices).position(0);
        mTextureBuffer = ByteBuffer
                .allocateDirect(TextureRotationUtils.getTextureVertices().length * 4)
                .order(ByteOrder.nativeOrder())
                .asFloatBuffer();
        mTextureBuffer.put(TextureRotationUtils.getTextureVertices()).position(0);
    }

    /**
     * 初始化滤镜
     */
    private void initFilters() {
        mCameraFilter = new CameraFilter();
        mRealTimeFilter = FilterManager.getFilterGroup();
    }

    /**
     * 调整由于surface的大小与SurfaceView大小不一致带来的显示问题
     */
    public void adjustViewSize() {
        float[] textureCoords = null;
        float[] vertexCoords = null;
        // TODO 这里可以做成镜像翻转的
        float[] textureVertices = TextureRotationUtils.getTextureVertices();
        float[] vertexVertices = TextureRotationUtils.CubeVertices;
        float ratioMax = Math.max((float) mDisplayWidth / mTextureWidth,
                (float) mDisplayHeight / mTextureHeight);
        // 新的宽高
        int imageWidth = Math.round(mTextureWidth * ratioMax);
        int imageHeight = Math.round(mTextureHeight * ratioMax);
        // 获取视图跟texture的宽高比
        float ratioWidth = (float) imageWidth / (float) mDisplayWidth;
        float ratioHeight = (float) imageHeight / (float) mDisplayHeight;
        if (mScaleType == ScaleType.CENTER_INSIDE) {
            vertexCoords = new float[] {
                    vertexVertices[0] / ratioHeight, vertexVertices[1] / ratioWidth, vertexVertices[2],
                    vertexVertices[3] / ratioHeight, vertexVertices[4] / ratioWidth, vertexVertices[5],
                    vertexVertices[6] / ratioHeight, vertexVertices[7] / ratioWidth, vertexVertices[8],
                    vertexVertices[9] / ratioHeight, vertexVertices[10] / ratioWidth, vertexVertices[11],
            };
        } else if (mScaleType == ScaleType.CENTER_CROP) {
            float distHorizontal = (1 - 1 / ratioWidth) / 2;
            float distVertical = (1 - 1 / ratioHeight) / 2;
            textureCoords = new float[] {
                    addDistance(textureVertices[0], distVertical), addDistance(textureVertices[1], distHorizontal),
                    addDistance(textureVertices[2], distVertical), addDistance(textureVertices[3], distHorizontal),
                    addDistance(textureVertices[4], distVertical), addDistance(textureVertices[5], distHorizontal),
                    addDistance(textureVertices[6], distVertical), addDistance(textureVertices[7], distHorizontal),
            };
        }
        if (vertexCoords == null) {
            vertexCoords = vertexVertices;
        }
        if (textureCoords == null) {
            textureCoords = textureVertices;
        }
        // 更新VertexBuffer 和 TextureBuffer
        mVertexBuffer.clear();
        mVertexBuffer.put(vertexCoords).position(0);
        mTextureBuffer.clear();
        mTextureBuffer.put(textureCoords).position(0);
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

    /**
     * 释放资源
     */
    public void release() {
        releaseFilters();
        releaseBuffers();
    }

    /**
     * 释放Filters资源
     */
    private void releaseFilters() {
        if (mCameraFilter != null) {
            mCameraFilter.release();
            mCameraFilter = null;
        }
        if (mRealTimeFilter != null) {
            mRealTimeFilter.release();
            mRealTimeFilter = null;
        }
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
    }


    /**
     * 更新TextureBuffer
     */
    public void updateTextureBuffer() {
        if (mCameraFilter != null) {
            mCameraFilter.updateTextureBuffer();
        }
    }

    /**
     * 渲染Texture的大小
     * @param width
     * @param height
     */
    public void onInputSizeChanged(int width, int height) {
        mTextureWidth = width;
        mTextureHeight = height;
        if (mCameraFilter != null) {
            mCameraFilter.onInputSizeChanged(width, height);
        }
        if (mRealTimeFilter != null) {
            mRealTimeFilter.onInputSizeChanged(width, height);
        }
    }

    /**
     * Surface显示的大小
     * @param width
     * @param height
     */
    public void onDisplaySizeChanged(int width, int height) {
        mDisplayWidth = width;
        mDisplayHeight = height;
        cameraFilterChanged();
        adjustViewSize();
        if (mRealTimeFilter != null) {
            mRealTimeFilter.onDisplayChanged(width, height);
        }
    }


    /**
     * 更新filter
     * @param type Filter类型
     */
    public void changeFilter(FilterType type) {
        if (mRealTimeFilter != null) {
            mRealTimeFilter.changeFilter(type);
        }
    }

    /**
     * 切换滤镜组
     * @param type
     */
    public void changeFilterGroup(FilterGroupType type) {
        synchronized (mSyncObject) {
            if (mRealTimeFilter != null) {
                mRealTimeFilter.release();
            }
            mRealTimeFilter = FilterManager.getFilterGroup(type);
            mRealTimeFilter.onInputSizeChanged(mTextureWidth, mTextureHeight);
            mRealTimeFilter.onDisplayChanged(mDisplayWidth, mDisplayHeight);
        }
    }

    /**
     * 滤镜或视图发生变化时调用
     */
    public void onFilterChanged() {
        if (mDisplayWidth != mDisplayHeight) {
            mCameraFilter.onDisplayChanged(mDisplayWidth, mDisplayHeight);
        }
        mCameraFilter.initFramebuffer(mTextureWidth, mTextureHeight);
    }


    /**
     * 设置SurfaceTexture TransformMatrix
     * @param matirx
     */
    public void setTextureTransformMatirx(float[] matirx) {
        if (mCameraFilter != null) {
            mCameraFilter.setTextureTransformMatirx(matirx);
        }
    }

    /**
     * 绘制渲染
     * @param textureId
     */
    public void drawFrame(int textureId) {
        if (mRealTimeFilter == null) {
            mCameraFilter.drawFrame(textureId);
            mCurrentTextureId = textureId;
        } else {
            int id = mCameraFilter.drawFrameBuffer(textureId);
            mRealTimeFilter.drawFrame(id, mVertexBuffer, mTextureBuffer);
            mCurrentTextureId = mRealTimeFilter.getCurrentTextureId();
        }
    }

    /**
     * 滤镜或视图发生变化时调用
     */
    private void cameraFilterChanged() {
        if (mDisplayWidth != mDisplayHeight) {
            mCameraFilter.onDisplayChanged(mDisplayWidth, mDisplayHeight);
        }
        mCameraFilter.initFramebuffer(mTextureWidth, mTextureHeight);
    }

    /**
     * 初始化录制的Filter
     */
    public void initRecordingFilter() {
        if (mRecordFilter == null) {
            mRecordFilter = new DisplayFilter();
        }
        mRecordFilter.onInputSizeChanged(mTextureWidth, mTextureHeight);
        mRecordFilter.onDisplayChanged(mDisplayWidth, mDisplayHeight);
    }

    /**
     * 释放录制的Filter资源
     */
    public void releaseRecordingFilter() {
        mRecordFilter.release();
        mRecordFilter = null;
    }

    /**
     * 渲染录制的帧
     */
    public void drawRecordingFrame() {
        mRecordFilter.drawFrame(mCurrentTextureId);
    }
}
```
RenderManager 用于控制渲染实体的，各种特效的渲染都通过RenderManager来控制，以及控制录制视频时需要绘制水印，都通过该类来进行管理。
至此，我们就可以愉快地做实时渲染以及做高效率的磨皮算法啦。

5、暂未实现的功能：
录制视频拆分线程、录音和视频合成功能。截止目前，相机项目的视频录制只接入了视频录制，录音和视频合成功能还没添加进来。本人的思路是通过多线程实现，由于MTK的CPU/驱动问题，onPreviewFrame回调帧率的影响，相机控制和渲染操作是在同一个HandlerThread 线程上实现的，但录音和音视频合成肯定不能在当前的RenderThread上实现，这样就阻塞了渲染过程，渲染核心部分在eglSwapBuffer之后的音视频合成部分并不属于渲染操作逻辑上的，因此可以跨线程来实现，但未知跨线程是否对onPreviewFrame帧率存在影响(主要是MTK CPU/驱动问题)，存疑。

