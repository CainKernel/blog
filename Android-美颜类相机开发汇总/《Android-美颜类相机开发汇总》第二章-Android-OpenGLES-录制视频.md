### 视频录制
之前写过一篇关于利用MediaCodec 和 SharedContext 实现无丢帧录制的文章：
[OpenGLES + MediaCodec 短视频分段录制实现与无丢帧录制优化](https://www.jianshu.com/p/9dc03b01bae3)
本项目也是通过这种方式实现的，但由于项目发生了比较大的改变，这里重新介绍一下。

* 编码器线程封装
HardcodeEncoder 是硬编码编码器封装，通过传递EGLContext到另外一个Looper线程来实现SharedContext功能。这里要注意一点，SharedContext 都是基于EGLContext的，如果你创建的EGLContext 被释放了，SharedContext就失效了，这会导致视频录制失败。因此，在释放EGLContext之前，必须停止录制或者等待视频录制完成。
```
public final class HardcodeEncoder {

    private static final String TAG = "HardcodeEncoder";
    private static final boolean VERBOSE = false;

    private final Object mReadyFence = new Object();

    // 初始化录制器
    static final int MSG_INIT_RECORDER = 0;
    // 帧可用
    static final int MSG_FRAME_AVAILABLE = 1;
    // 渲染帧
    static final int MSG_DRAW_FRAME = 2;
    // 停止录制
    static final int MSG_STOP_RECORDING = 3;
    // 暂停录制
    static final int MSG_PAUSE_RECORDING = 4;
    // 继续录制
    static final int MSG_CONTINUE_RECORDING = 5;
    // 是否允许录制
    static final int MSG_ENABLE_AUDIO = 6;
    // 退出
    static final int MSG_QUIT = 7;
    // 设置渲染纹理尺寸
    static final int MSG_SET_TEXTURE_SIZE = 8;

    // 输出路径
    private String mOutputPath;

    // 录制线程
    private RecordThread mRecordThread;

    private static class HardcodeEncoderHolder {
        public static HardcodeEncoder instance = new HardcodeEncoder();
    }

    private HardcodeEncoder() {}

    public static HardcodeEncoder getInstance() {
        return HardcodeEncoderHolder.instance;
    }

    /**
     * 准备录制器
     */
    public HardcodeEncoder preparedRecorder() {
        synchronized (mReadyFence) {
            if (mRecordThread == null) {
                mRecordThread = new RecordThread(this);
                mRecordThread.start();
                mRecordThread.waitUntilReady();
            }
        }
        return this;
    }

    /**
     * 销毁录制器
     */
    public void destroyRecorder() {
        synchronized (mReadyFence) {
            if (mRecordThread != null) {
                Handler handler = mRecordThread.getHandler();
                if (handler != null) {
                    handler.sendMessage(handler.obtainMessage(MSG_QUIT));
                }
                mRecordThread = null;
            }
        }
    }

    /**
     * 初始化录制器，此时耗时大约200ms左右，不能放在跟渲染线程同一个Looper里面
     * @param width
     * @param height
     * @param listener
     */
    public void initRecorder(int width, int height, MediaEncoder.MediaEncoderListener listener) {
        Handler handler = mRecordThread.getHandler();
        if (handler != null) {
            handler.sendMessage(handler.obtainMessage(MSG_INIT_RECORDER, width, height, listener));
        }
    }

    /**
     * 设置渲染Texture的宽高
     * @param width
     * @param height
     */
    public void setTextureSize(int width, int height) {
        Handler handler = mRecordThread.getHandler();
        if (handler != null) {
            handler.sendMessage(handler.obtainMessage(MSG_SET_TEXTURE_SIZE, width, height));
        }
    }

    /**
     * 开始录制
     * @param sharedContext EGLContext上下文包装类
     */
    public void startRecording(final Context context, final EGLContext sharedContext) {
        Handler handler = mRecordThread.getHandler();
        if (handler != null) {
            handler.post(new Runnable() {
                @Override
                public void run() {
                    mRecordThread.startRecording(context, (EGLContext) sharedContext);
                }
            });
        }
    }


    /**
     * 帧可用
     */
    public void frameAvailable() {
        Handler handler = mRecordThread.getHandler();
        if (handler != null) {
            handler.sendMessage(handler.obtainMessage(MSG_FRAME_AVAILABLE));
        }
    }

    /**
     * 发送渲染指令
     * @param texture 当前Texture
     * @param timeStamp 时间戳
     */
    public void drawRecorderFrame(int texture, long timeStamp) {
        Handler handler = mRecordThread.getHandler();
        if (handler != null) {
            handler.sendMessage(handler
                    .obtainMessage(MSG_DRAW_FRAME, texture, 0 /* unused */, timeStamp));
        }
    }

    /**
     * 停止录制
     */
    public void stopRecording() {
        if (mRecordThread == null) {
            return;
        }
        Handler handler = mRecordThread.getHandler();
        if (handler != null) {
            handler.sendMessage(handler.obtainMessage(MSG_STOP_RECORDING));
        }
    }


    /**
     * 暂停录制
     */
    public void pauseRecord() {
        Handler handler = mRecordThread.getHandler();
        if (handler != null) {
            handler.sendMessage(handler.obtainMessage(MSG_PAUSE_RECORDING));
        }
    }

    /**
     * 继续录制
     */
    public void continueRecord() {
        Handler handler = mRecordThread.getHandler();
        if (handler != null) {
            handler.sendMessage(handler.obtainMessage(MSG_CONTINUE_RECORDING));
        }
    }

    /**
     * 是否允许录音
     * @param enable
     */
    public HardcodeEncoder enableAudioRecord(boolean enable) {
        Handler handler = mRecordThread.getHandler();
        if (handler != null) {
            handler.sendMessage(handler.obtainMessage(MSG_ENABLE_AUDIO, enable));
        }
        return this;
    }

    /**
     * 设置输出路径
     * @param path
     */
    public HardcodeEncoder setOutputPath(String path) {
        mOutputPath = path;
        return this;
    }

    /**
     * 获取输出路径
     * @return
     */
    public String getOutputPath() {
        return mOutputPath;
    }

    /**
     * 录制线程
     */
    private static class RecordThread extends Thread {

        private final Object mReadyFence = new Object();
        private boolean mReady;

        // 输入纹理大小
        private int mTextureWidth, mTextureHeight;
        // 录制视频大小
        private int mVideoWidth, mVideoHeight;

        private EglCore mEglCore;
        // 录制视频用的EGLSurface
        private WindowSurface mRecordWindowSurface;
        // 录制的Filter
        private GLImageFilter mRecordFilter;
        private FloatBuffer mVertexBuffer;
        private FloatBuffer mTextureBuffer;
        // 复用器管理器
        private MediaMuxerWrapper mMuxerManager;

        // 是否允许录音
        private boolean enableAudio = true;

        // 是否处于录制状态
        private boolean isRecording = false;

        // MediaCodec 初始化和释放所需要的时间总和
        // 根据试验的结果，大部分手机在初始化和释放阶段的时间总和都要300ms ~ 800ms左右
        // 第一次初始化普遍时间比较长，新出的红米5(2G内存)在第一次初始化1624ms，后面则是600ms左右
        private long mProcessTime = 0;

        // 录制线程Handler回调
        private RecordHandler mHandler;

        private WeakReference<HardcodeEncoder> mWeakRecorder;

        RecordThread(HardcodeEncoder manager) {
            mWeakRecorder = new WeakReference<>(manager);
        }

        @Override
        public void run() {
            Looper.prepare();
            synchronized (mReadyFence) {
                mVertexBuffer = OpenGLUtils.createFloatBuffer(TextureRotationUtils.CubeVertices);
                mTextureBuffer = OpenGLUtils.createFloatBuffer(TextureRotationUtils.TextureVertices);
                mHandler = new RecordHandler(this);
                mReady = true;
                mReadyFence.notify();
            }
            Looper.loop();
            if (VERBOSE) {
                Log.d(TAG, "Record thread exiting");
            }

            synchronized (mReadyFence) {
                release();
                mReady = false;
                mHandler = null;
            }
        }

        /**
         * 等待线程结束
         */
        void waitUntilReady() {
            synchronized (mReadyFence) {
                while (!mReady) {
                    try {
                        mReadyFence.wait();
                    } catch (InterruptedException ie) {

                    }
                }
            }
        }

        /**
         * 初始化录制器
         * @param width
         * @param height
         * @param listener
         */
        void initRecorder(int width, int height, MediaEncoder.MediaEncoderListener listener) {
            if (VERBOSE) {
                Log.d(TAG, "init recorder");
            }

            synchronized (mReadyFence) {
                long time = System.currentTimeMillis();
                mVideoWidth = width;
                mVideoHeight = height;

                String filePath = mWeakRecorder.get().getOutputPath();

                // 如果路径为空，则生成默认的路径
                if (TextUtils.isEmpty(filePath)) {
                    throw new IllegalArgumentException("filePath Must no be empty");
                }
                File file = new File(filePath);
                if (!file.getParentFile().exists()) {
                    file.getParentFile().mkdirs();
                }
                try {
                    mMuxerManager = new MediaMuxerWrapper(file.getAbsolutePath());
                    new MediaVideoEncoder(mMuxerManager, listener, mVideoWidth, mVideoHeight);
                    if (enableAudio) {
                        new MediaAudioEncoder(mMuxerManager, listener);
                    }

                    mMuxerManager.prepare();
                } catch (IOException e) {
                    Log.e(TAG, "startRecording:", e);
                }
                mProcessTime += (System.currentTimeMillis() - time);
            }
        }

        /**
         * 设置渲染Texture的宽高
         * @param width
         * @param height
         */
        void setTextureSize(int width, int height) {
            if (VERBOSE) {
                Log.d(TAG, "setTextureSize");
            }
            synchronized (mReadyFence) {
                mTextureWidth = width;
                mTextureHeight = height;
            }
        }

        /**
         * 开始录制
         * @param eglContext EGLContext上下文包装类
         */
        void startRecording(Context context, EGLContext eglContext) {
            if (VERBOSE) {
                Log.d(TAG, " start recording");
            }
            synchronized (mReadyFence) {
                if (mMuxerManager.getVideoEncoder() == null) {
                    return;
                }
                // 释放之前的Egl
                if (mRecordWindowSurface != null) {
                    mRecordWindowSurface.releaseEglSurface();
                }
                if (mEglCore != null) {
                    mEglCore.release();
                }
                // 重新创建一个EglContext 和 Window Surface
                mEglCore = new EglCore(eglContext, EglCore.FLAG_RECORDABLE);
                if (mRecordWindowSurface != null) {
                    mRecordWindowSurface.recreate(mEglCore);
                } else {
                    mRecordWindowSurface = new WindowSurface(mEglCore,
                            ((MediaVideoEncoder) mMuxerManager.getVideoEncoder()).getInputSurface(),
                            true);
                }
                mRecordWindowSurface.makeCurrent();
                initRecordingFilter(context);
                if (mMuxerManager != null) {
                    mMuxerManager.startRecording();
                }
                isRecording = true;
            }
        }


        /**
         * 帧可用
         */
        void frameAvailable() {
            if (VERBOSE) {
                Log.d(TAG, "frame available");
            }
            synchronized (mReadyFence) {
                if (mMuxerManager != null && mMuxerManager.getVideoEncoder() != null && isRecording) {
                    mMuxerManager.getVideoEncoder().frameAvailableSoon();
                }
            }
        }

        /**
         * 发送渲染指令
         * @param currentTexture 当前Texture
         * @param timeStamp 时间戳
         */
        void drawRecordingFrame(int currentTexture, long timeStamp) {
            if (VERBOSE) {
                Log.d(TAG, "draw recording frame");
            }
            synchronized (mReadyFence) {
                if (mRecordWindowSurface != null) {
                    mRecordWindowSurface.makeCurrent();
                    drawRecordingFrame(currentTexture);
                    mRecordWindowSurface.setPresentationTime(timeStamp);
                    mRecordWindowSurface.swapBuffers();
                }
            }
        }

        /**
         * 停止录制
         */
        void stopRecording() {
            if (VERBOSE) {
                Log.d(TAG, "stop recording");
            }
            synchronized (mReadyFence) {
                long time = System.currentTimeMillis();
                isRecording = false;
                if (mMuxerManager != null) {
                    mMuxerManager.stopRecording();
                    mMuxerManager = null;
                }
                if (mRecordWindowSurface != null) {
                    mRecordWindowSurface.release();
                    mRecordWindowSurface = null;
                }
                // 释放资源
                releaseRecordingFilter();

                if (VERBOSE) {
                    mProcessTime += (System.currentTimeMillis() - time);
                    Log.d(TAG, "sum of init and release time: " + mProcessTime + "ms");
                    mProcessTime = 0;
                }
            }
        }


        /**
         * 暂停录制
         */
        void pauseRecording() {
            if (VERBOSE) {
                Log.d(TAG, "pause recording");
            }
            synchronized (mReadyFence) {
                if (mMuxerManager != null && isRecording) {
                    mMuxerManager.pauseRecording();
                }
            }
        }

        /**
         * 继续录制
         */
        void continueRecording() {
            if (VERBOSE) {
                Log.d(TAG, "continue recording");
            }
            synchronized (mReadyFence) {
                if (mMuxerManager != null && isRecording) {
                    mMuxerManager.continueRecording();
                }
            }
        }

        /**
         * 初始化录制的Filter
         * TODO 录制视频大小跟渲染大小、显示大小拆分成不同的大小
         */
        private void initRecordingFilter(Context context) {
            if (mRecordFilter == null) {
                mRecordFilter = new GLImageFilter(context);
            }
            mRecordFilter.onInputSizeChanged(mTextureWidth, mTextureHeight);
            mRecordFilter.onDisplaySizeChanged(mVideoWidth, mVideoHeight);
        }

        /**
         * 渲染录制的帧
         */
        private void drawRecordingFrame(int textureId) {
            if (mRecordFilter != null) {
                GLES30.glViewport(0, 0, mVideoWidth, mVideoHeight);
                mRecordFilter.drawFrame(textureId, mVertexBuffer, mTextureBuffer);
            }
        }

        /**
         * 释放录制的Filter资源
         */
        private void releaseRecordingFilter() {
            if (mRecordFilter != null) {
                mRecordFilter.release();
                mRecordFilter = null;
            }
        }

        /**
         * 销毁资源
         */
        public void release() {
            // 停止录制
            stopRecording();
            if (mEglCore != null) {
                mEglCore.release();
                mEglCore = null;
            }
            if (mRecordWindowSurface != null) {
                mRecordWindowSurface.release();
                mRecordWindowSurface = null;
            }
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
         * 是否允许录音
         * @param enable
         */
        void enableAudioRecording(boolean enable) {
            if (VERBOSE) {
                Log.d(TAG, "enable audio recording ? " + enable);
            }
            synchronized (mReadyFence) {
                enableAudio = enable;
            }
        }

        /**
         * 获取Handler
         */
        public RecordHandler getHandler() {
            return mHandler;
        }
    }

    /**
     * 录制线程Handler回调
     */
    private static class RecordHandler extends Handler {

        private WeakReference<RecordThread> mWeakRecordThread;

        public RecordHandler(RecordThread thread) {
            mWeakRecordThread = new WeakReference<RecordThread>(thread);
        }

        @Override
        public void handleMessage(Message msg) {
            int what = msg.what;
            RecordThread thread = mWeakRecordThread.get();
            if (thread == null) {
                Log.w(TAG, "RecordHandler.handleMessage: encoder is null");
                return;
            }

            switch (what) {
                // 初始化录制器
                case MSG_INIT_RECORDER:
                    thread.initRecorder(msg.arg1, msg.arg2,
                            (MediaEncoder.MediaEncoderListener) msg.obj);
                    break;

                // 帧可用
                case MSG_FRAME_AVAILABLE:
                    thread.frameAvailable();
                    break;

                // 渲染帧
                case MSG_DRAW_FRAME:
                    thread.drawRecordingFrame(msg.arg1, (Long) msg.obj);
                    break;

                // 停止录制
                case MSG_STOP_RECORDING:
                    thread.stopRecording();
                    break;

                // 暂停录制
                case MSG_PAUSE_RECORDING:
                    thread.pauseRecording();
                    break;

                // 继续录制
                case MSG_CONTINUE_RECORDING:
                    thread.continueRecording();
                    break;

                // 是否允许录音
                case MSG_ENABLE_AUDIO:
                    thread.enableAudioRecording((Boolean) msg.obj);
                    break;

                // 退出线程
                case MSG_QUIT:
                    removeCallbacksAndMessages(null);
                    Looper.myLooper().quit();
                    break;

                // 设置渲染Texture的宽高
                case MSG_SET_TEXTURE_SIZE:
                    thread.setTextureSize(msg.arg1, msg.arg2);
                    break;

                default:
                    throw new RuntimeException("Unhandled msg what = " + what);
            }
        }
    }

}
```
* 计时器封装
计时器部分是通过Handler封装实现的。实现如下：
```
public abstract class RecordTimer {

    //  倒计时(毫秒)
    private final long mMillisInFuture;

    // 回调时间(毫秒)
    private final long mCountdownInterval;

    // 停止的时间
    private long mStopTimeInFuture;

    // 计算回调次数
    private int mTickCounter;
    // 记录开始时间
    private long mStartTime;

    public RecordTimer(long millisInFuture, long countDownInterval) {
        mMillisInFuture = millisInFuture;
        mCountdownInterval = countDownInterval;
        mTickCounter = 0;
    }

    /**
     * 取消计时
     */
    public final void cancel() {
        mHandler.removeMessages(MSG);
    }

    /**
     * 开始计时
     * @return
     */
    public synchronized final RecordTimer start() {
        if (mMillisInFuture <= 0) {
            onFinish();
            return this;
        }

        mStartTime = SystemClock.elapsedRealtime();
        mStopTimeInFuture = mStartTime + mMillisInFuture;

        mHandler.sendMessage(mHandler.obtainMessage(MSG));
        return this;
    }

    /**
     * 计时间隔回调
     * @param millisUntilFinished
     */
    public abstract void onTick(long millisUntilFinished);

    /**
     * 完成计时
     */
    public abstract void onFinish();

    private static final int MSG = 1;

    // handler回调
    @SuppressWarnings("HandlerLeak")
    private Handler mHandler = new Handler() {

        @Override
        public void handleMessage(Message msg) {

            synchronized (RecordTimer.this) {
                // 获取精确倒计时
                final long millisLeft = mStopTimeInFuture
                        - SystemClock.elapsedRealtime();

                if (millisLeft <= 0) {
                    onFinish();
                } else if (millisLeft < mCountdownInterval) {
                    // no tick, just delay until done
                    sendMessageDelayed(obtainMessage(MSG), millisLeft);
                } else {
                    long lastTickStart = SystemClock.elapsedRealtime();
                    onTick(millisLeft);
                    // 计算延时
                    long now = SystemClock.elapsedRealtime();
                    long extraDelay = now - mStartTime - mTickCounter
                            * mCountdownInterval;
                    mTickCounter++;
                    long delay = lastTickStart + mCountdownInterval - now
                            - extraDelay;
                    // 如果回调时间超过了延时技术，则跳过下一次回调
                    while (delay < 0) {
                        delay += mCountdownInterval;
                    }
                    // 下一次的回调延时
                    sendMessageDelayed(obtainMessage(MSG), delay);
                }
            }
        }
    };
}
```
* 录制器封装：
```
public final class PreviewRecorder {
    // 十秒时长
    public static final int DURATION_TEN_SECOND = 10 * 1000;
    // 三分钟时长
    public static final int DURATION_THREE_MINUTE = 180 * 1000;

    // 录制类型，默认是录制视频
    private RecordType mRecordType;
    // 输出路径
    private String mOutputPath;
    // 录制视频宽度
    private int mRecordWidth;
    // 录制视频高度
    private int mRecordHeight;
    // 是否允许录音
    private boolean mRecordAudio;
    // 是否处于录制状态
    private boolean isRecording;
    // MediaEncoder准备好的数量
    private int mPreparedCount = 0;
    // 开始MediaEncoder的数量
    private int mStartedCount = 0;
    // 释放MediaEncoder的数量
    private int mReleaseCount = 0;

    // 精确倒计时
    private RecordTimer mRecordTimer;
    // 倒计时数值
    private long mMaxMillisSeconds = DURATION_TEN_SECOND;
    // 50毫秒读取一次
    private long mCountDownInterval = 50;

    // 当前走过的时长，有可能跟视频的长度不一致
    // 在最后一秒内点击录制，倒计时走完，但录制的视频立即停止
    // 这样的话，最后一段显示的视频长度跟当前走过的时长并不相等
    private long mCurrentDuration = 0;
    // 记录最后一秒点击时剩余的时长
    private long mLastSecondLeftTime = 0;
    // 记录是否最后点击停止
    private boolean mLastSecondStop = false;
    // 是否需要处理最后一秒的情况
    private boolean mProcessLasSecond = true;
    // 计时器是否停止
    private boolean mTimerFinish = false;

    // 分段视频列表
    private LinkedList<RecordItem> mVideoList = new LinkedList<RecordItem>();

    // 录制监听器
    private OnRecordListener mRecordListener;

    private static class RecordEngineHolder {
        public static PreviewRecorder instance = new PreviewRecorder();
    }

    private PreviewRecorder() {
        reset();
    }

    public static PreviewRecorder getInstance() {
        return RecordEngineHolder.instance;
    }

    /**
     * 重置参数
     */
    public void reset() {
        mRecordType = RecordType.Video;
        mOutputPath = null;
        mRecordWidth = 0;
        mRecordHeight = 0;
        mRecordAudio = false;
        isRecording = false;
        mPreparedCount = 0;
        mStartedCount = 0;
        mReleaseCount = 0;
    }

    /**
     * 设置录制类型
     * @param type
     * @return
     */
    public PreviewRecorder setRecordType(RecordType type) {
        mRecordType = type;
        return this;
    }

    /**
     * 设置输出路径
     * @param path
     * @return
     */
    public PreviewRecorder setOutputPath(String path) {
        mOutputPath = path;
        return this;
    }

    /**
     * 设置录制大小
     * @param width
     * @param height
     * @return
     */
    public PreviewRecorder setRecordSize(int width, int height) {
        mRecordWidth = width;
        mRecordHeight = height;
        return this;
    }

    /**
     * 是否允许录音
     * @param enable
     * @return
     */
    public PreviewRecorder enableAudio(boolean enable) {
        mRecordAudio = enable;
        return this;
    }

    /**
     *  开始录制
     */
    public void startRecord() {
        if (mRecordWidth <= 0 || mRecordHeight <= 0) {
            throw new IllegalArgumentException("Video's width or height must not be zero");
        }

        if (TextUtils.isEmpty(mOutputPath)) {
            throw new IllegalArgumentException("Video output path must not be empty");
        }

        // 初始化录制器
        HardcodeEncoder.getInstance()
                .preparedRecorder()
                .setOutputPath(mOutputPath)
                .enableAudioRecord(mRecordAudio)
                .initRecorder(mRecordWidth, mRecordHeight, mEncoderListener);

        // 初始化计时器
        initTimer();
    }

    /**
     * 停止录制
     */
    public void stopRecord() {
        stopRecord(true);
    }

    /**
     * 停止录制
     * @param stopTimer 是否停止计时器
     */
    public void stopRecord(boolean stopTimer) {
        // 停止录制
        PreviewRenderer.getInstance().stopRecording();
        // 停止倒计时
        if (stopTimer) {
            stopTimer();
        }
    }

    /**
     * 销毁录制器
     */
    public void destroyRecorder() {
        HardcodeEncoder.getInstance().destroyRecorder();
    }

    /**
     * 删除一段已记录的时长
     */
    public void deleteRecordDuration() {
        resetDuration();
        resetLastSecondStop();
    }

    /**
     * 取消录制
     */
    public void cancelRecording() {
        HardcodeEncoder.getInstance().stopRecording();
        cancelTimerWithoutSaving();
    }

    /**
     * 设置录制监听器
     * @param listener
     * @return
     */
    public PreviewRecorder setOnRecordListener(OnRecordListener listener) {
        mRecordListener = listener;
        return this;
    }

    /**
     * 判断是否正在录制
     * @return
     */
    public boolean isRecording() {
        return isRecording;
    }

    /**
     * 获取录制的总时长
     * @return
     */
    public int getDuration() {
        int duration = 0;
        if (mVideoList != null) {
            for (RecordItem recordItem : mVideoList) {
                duration += recordItem.getDuration();
            }
        }
        return duration;
    }

    /**
     * 添加分段视频
     * @param path      视频路径
     * @param duration  视频时长
     */
    private void addSubVideo(String path, int duration) {
        if (mVideoList == null) {
            mVideoList = new LinkedList<RecordItem>();
        }
        RecordItem recordItem = new RecordItem();
        recordItem.mediaPath = path;
        recordItem.duration = duration;
        mVideoList.add(recordItem);
    }

    /**
     * 移除当前分段视频
     */
    public void removeLastSubVideo() {
        RecordItem recordItem = mVideoList.get(mVideoList.size() - 1);
        mVideoList.remove(recordItem);
        if (recordItem != null) {
            recordItem.delete();
            mVideoList.remove(recordItem);
        }
    }

    /**
     * 删除所有分段视频
     */
    public void removeAllSubVideo() {
        if (mVideoList != null) {
            for (RecordItem part : mVideoList) {
                part.delete();
            }
            mVideoList.clear();
        }
    }

    /**
     * 获取分段视频路径
     * @return
     */
    public List<String> getSubVideoPathList() {
        if (mVideoList == null || mVideoList.isEmpty()) {
            return new ArrayList<String>();
        }
        List<String> mediaPaths = new ArrayList<String>();
        for (int i = 0; i < mVideoList.size(); i++) {
            mediaPaths.add(i, mVideoList.get(i).getMediaPath());
        }
        return mediaPaths;
    }

    /**
     * 获取分段视频数量
     * @return
     */
    public int getNumberOfSubVideo() {
        return mVideoList.size();
    }


    // --------------------------------------- 计时器操作 ------------------------------------------
    /**
     * 初始化倒计时
     */
    private void initTimer() {

        cancelCountDown();

        mRecordTimer = new RecordTimer(mMaxMillisSeconds, mCountDownInterval) {
            @Override
            public void onTick(long millisUntilFinished) {
                if (!mTimerFinish) {
                    // 获取视频总时长
                    int previousDuration = getDuration();
                    // 获取当前分段视频走过的时间
                    mCurrentDuration = mMaxMillisSeconds - millisUntilFinished;
                    // 如果总时长够设定的最大时长，则需要停止计时
                    if (previousDuration + mCurrentDuration >= mMaxMillisSeconds) {
                        mCurrentDuration = mMaxMillisSeconds - previousDuration;
                        mTimerFinish = true;
                    }
                    // 计时回调
                    if (mRecordListener != null) {
                        mRecordListener.onRecordProgressChanged(getVisibleDuration());
                    }
                    // 是否需要结束计时器
                    if (mTimerFinish) {
                        mTimerHandler.sendEmptyMessage(0);
                    }
                }
            }

            @Override
            public void onFinish() {
                mTimerFinish = true;
                if (mRecordListener != null) {
                    mRecordListener.onRecordProgressChanged(getVisibleDuration(true));
                }
            }
        };
    }

    /**
     * 开始倒计时
     */
    private void startTimer() {
        if (mRecordTimer != null) {
            mRecordTimer.start();
        }
    }

    /**
     * 停止倒计时
     */
    private void stopTimer() {
        // 重置最后一秒停止标志
        mLastSecondStop = false;
        // 判断是否需要处理最后一秒的情况
        if (mProcessLasSecond) {
            // 如果在下一次计时器回调之前剩余时间小于1秒，则表示是最后一秒内点击了停止
            if (getAvailableTime() + mCountDownInterval < 1000) {
                mLastSecondStop = true;
                mLastSecondLeftTime = getAvailableTime();
            }
        }
        // 如果不是最后一秒，则立即停止
        if (!mLastSecondStop) {
            cancelCountDown();
        }
    }

    /**
     * 取消倒计时，不保存走过的时长、停止标志、剩余时间等
     */
    private void cancelTimerWithoutSaving() {
        cancelCountDown();
        resetDuration();
        resetLastSecondStop();
        mLastSecondLeftTime = 0;
    }

    /**
     * 取消倒计时
     */
    private void cancelCountDown() {
        if (mRecordTimer != null) {
            mRecordTimer.cancel();
            mRecordTimer = null;
        }
        // 复位结束标志
        mTimerFinish = false;
    }

    /**
     * 重置当前走过的时长
     */
    private void resetDuration() {
        mCurrentDuration = 0;
    }

    /**
     * 重置最后一秒停止标志
     */
    private void resetLastSecondStop() {
        mLastSecondStop = false;
    }

    /**
     * 设置总时长
     * @param type
     */
    public void setMilliSeconds(CountDownType type) {
        if (type == CountDownType.TenSecond) {
            mMaxMillisSeconds = DURATION_TEN_SECOND;
        } else if (type == CountDownType.ThreeMinute) {
            mMaxMillisSeconds = DURATION_THREE_MINUTE;
        }
    }

    /**
     * 获取总时长
     * @return
     */
    public long getMaxMilliSeconds() {
        return mMaxMillisSeconds;
    }

    /**
     * 设置刷新间隔
     * @param interval
     */
    public void setCountDownInterval(long interval) {
        mCountDownInterval = interval;
    }

    /**
     * 获取刷新间隔
     * @return
     */
    public long getCountDownInterval() {
        return mCountDownInterval;
    }

    /**
     * 获取剩余时间
     * @return
     */
    private long getAvailableTime() {
        return mMaxMillisSeconds - getDuration() - mCurrentDuration;
    }

    /**
     * 获取当前实际时长 (跟显示的时长不一定不一样)
     * @return
     */
    private long getRealDuration() {
        // 如果是最后一秒内点击，则计时器走过的时长要比视频录制的时长短一些，需要减去多余的时长
        if (mLastSecondLeftTime > 0) {
            long realTime = mCurrentDuration - mLastSecondLeftTime;
            mLastSecondLeftTime = 0;
            return realTime;
        }
        return mCurrentDuration;
    }

    /**
     * 获取显示的时长
     */
    public long getVisibleDuration() {
        return getVisibleDuration( false);
    }

    /**
     * 获取显示的时长
     * @param finish    是否完成
     * @return
     */
    private long getVisibleDuration(boolean finish) {
        if (finish) {
            return mMaxMillisSeconds;
        } else {
            long time = getDuration() + mCurrentDuration;
            if (time > mMaxMillisSeconds) {
                time = mMaxMillisSeconds;
            }
            return time;
        }
    }

    /**
     * 获取显示的时间文本
     * @return
     */
    public String getVisibleDurationString() {
        return StringUtils.generateMillisTime((int) getVisibleDuration());
    }

    /**
     * 是否最后一秒内停止了
     * @return
     */
    public boolean isLastSecondStop() {
        return mLastSecondStop;
    }

    /**
     * 是否处理最后一秒的情况(不再停止，但记录时长)
     * @param enable
     */
    public void setProcessLastSecond(boolean enable) {
        mProcessLasSecond = enable;
    }

    // 倒计时Handler
    @SuppressWarnings("HandlerLeak")
    private Handler mTimerHandler = new Handler() {
        @Override
        public void handleMessage(Message msg) {
            cancelCountDown();
        }
    };

    /**
     * 录制编码监听器
     */
    private MediaEncoder.MediaEncoderListener mEncoderListener = new MediaEncoder.MediaEncoderListener() {

        @Override
        public void onPrepared(MediaEncoder encoder) {
            mPreparedCount++;
            // 不允许录音、允许录制音频并且准备好两个MediaEncoder，就可以开始录制了，如果是GIF，则没有音频
            if (!mRecordAudio || (mRecordAudio && mPreparedCount == 2) || mRecordType == RecordType.Gif) { // 录制GIF，没有音频
                // 准备完成，开始录制
                PreviewRenderer.getInstance().startRecording();
                // 重置
                mPreparedCount = 0;
            }
        }

        @Override
        public void onStarted(MediaEncoder encoder) {
            mStartedCount++;
            // 不允许音频录制、允许录制音频并且开始了两个MediaEncoder，就处于录制状态了，如果是GIF，则没有音频
            if (!mRecordAudio || (mRecordAudio && mStartedCount == 2) || mRecordType == RecordType.Gif) {
                isRecording = true;
                // 重置状态
                mStartedCount = 0;
                // 开始倒计时
                startTimer();
                // 录制开始回调
                if (mRecordListener != null) {
                    mRecordListener.onRecordStarted();
                }

            }
        }

        @Override
        public void onStopped(MediaEncoder encoder) {
        }

        @Override
        public void onReleased(MediaEncoder encoder) { // 复用器释放完成
            mReleaseCount++;
            // 不允许音频录制、允许录制音频并且释放了两个MediaEncoder，就完全释放掉了，如果是GIF，则没有音频
            if (!mRecordAudio || (mRecordAudio && mReleaseCount == 2) || mRecordType == RecordType.Gif) {
                // 录制完成跳转预览页面
                String outputPath = HardcodeEncoder.getInstance().getOutputPath();
                // 添加分段视频，存在时长为0的情况，也就是取消倒计时但不保存时长的情况
                if (getRealDuration() > 0) {
                    addSubVideo(outputPath, (int) getRealDuration());
                } else { // 移除多余的视频
                    FileUtils.deleteFile(outputPath);
                }
                // 重置当前走过的时长
                resetDuration();
                // 处于非录制状态
                isRecording = false;
                // 重置释放状态
                mReleaseCount = 0;

                // 录制完成回调
                if (mRecordListener != null) {
                    mRecordListener.onRecordFinish();
                }

            }
        }
    };

    /**
     * 合并视频
     */
    public void combineVideo(final String path, final VideoCombiner.CombineListener listener) {
        new Thread(new Runnable() {
            @Override
            public void run() {
                new VideoCombiner(getSubVideoPathList(), path, listener).combineVideo();
            }
        }).start();
    }

    /**
     * 设置录制类型
     */
    public enum CountDownType {
        TenSecond,
        ThreeMinute,
    }

    /**
     * 录制类型
     */
    public enum RecordType {
        Gif,
        Video
    }
}
```
这里封装了录制器录制开始、录制结束、删除分段视频等基本功能。
使用如下：
```
// 开始录制
PreviewRecorder.getInstance()
                    .setRecordType(mCameraParam.mGalleryType == GalleryType.VIDEO ? PreviewRecorder.RecordType.Video : PreviewRecorder.RecordType.Gif)
                    .setOutputPath(PathConstraints.getVideoCachePath(mActivity))
                    .enableAudio(enableAudio)
                    .setRecordSize(width, height)
                    .setOnRecordListener(mRecordListener)
                    .startRecord();

// 停止录制
PreviewRecorder.getInstance().stopRecord();
```
我们只需要传递录制类型、视频存放路径、是否录制音频、录制视频的宽高、录制回调等参数，然后调用startRecord()方法开始录制。停止录制则只需要调用stopRecord()即可。
至此，我们就实现了通过SharedContext + MediaCodec的方式来录制视频的功能。

详细实现过程请参照本人的项目：
[CainCamera](https://github.com/CainKernel/CainCamera)
