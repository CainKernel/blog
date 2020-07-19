录制视频功能在现在的很多应用上都存有一席之地，在直播类、美颜类应用上更是不可或缺的的一部分功能。在Android中录制视频有软硬编码两种方式。软编码就是利用CPU对视频帧进行编码，编码效率上肯定不如硬编码的，但软编码支持的编码格式较多，在直播类APP中，软编码能更好地应对网络抖动等状况。硬编码的效率高，编码格式支持有限。取舍问题，一般情况下得根据应用的类型进行分类。对于美颜类APP来说，录制的视频一般都是经过了GPU渲染后的，如果使用软编码，则需要将数据从GPU读取出来，读取方式有glReadPixels 和PBO两种方式，glReadPixels由于同步产生的bottleneck问题，在一些处理器上取得一帧的数据达到了100ms以上，这种方式肯定是不可行的。对于PBO来说，内存对齐和分辨率来也会对其效率产生影响。因此对于软编码来说，取得视频帧的效率是最核心的问题。业界一般都采用CPU和GPU共享内存方式来解决的，跟iOS的架构方案类似，这里不做细谈。硬编码不需要从GPU中读取数据就能直接完成编码，在Android中硬编码录制方案一般是采用MediaCodec来录制视频帧的。我们接下来要谈的就是如何利用MediaCodec + MediaMuxer 硬编码实现短视频分段录制以及如何尽可能地实现无丢帧录制
0、参考文章
关于如何使用MediaCodec录制视频的原理，可以参考以下文章：[Android在MediaMuxer和MediaCodec录制视频示例 - audio+video](http://blog.csdn.net/luoyouren/article/details/52135476)，或者参考这篇文章[Android MediaCodec编解码详解及demo](http://www.jianshu.com/p/e6c683d6dbbe) 介绍的一些Demo进行学习, 以及官网关于[MediaCodec](https://developer.android.google.cn/reference/android/media/MediaCodec.html)的资料。关于如何使用MediaCodec进行录制音视频，这里不介绍。本文章会默认你已经回使用MediaCodec进行视频录制，介绍的是在录制过程如何对预览帧率和录制视频帧率进行优化。

1、参考项目
使用MediaCodec硬编码录制音视频并混合的具体实现，可以参考开源项目：
[AudioVideoRecordingSample](https://github.com/saki4510t/AudioVideoRecordingSample)
如果不做其他渲染操作的话，该项目还是比较有参考意义的。在该项目采用的录制方案中，录制线程是跟渲染线程是在同一个Looper 里面进行的。这里会产生一个问题，如果录制的视频分辨率比较高，而你渲染的东西又比较多，比如你做了磨皮、美白、瘦脸、贴纸等功能的渲染后，在同一个Looper里面做音视频录制和混合的话，会导致预览帧率降低，预览产生不连贯感甚至卡顿现象发生，录制出来的视频有时可能会出现卡顿、帧率过快等奇怪的现象。而且，该方案在处理AudioRecord上存在一些细节性的Bug，那就是在分段视频录制时，过快操作会导致AudioRecord native_stop等状态出错，导致崩溃发生。因此，直接使用该方案对于短视频分段录制来说，并不是非常合适，需要做一些修改，如何修改，我们将会在后面的实现部分做讲解。首先我们来分析一下目前各大商业类相机在短视频录制的方案吧。

2、短视频分段录制的商业方案
短视频录制这个功能，目前市面上的美颜类相机实现得比较好的不是很多。截止本文章发布(2017年12月7日)，市面上主流的美颜类相机录制方案如下：
美颜相机： 长按拍照按钮开始录制，不能分段录制，录制有最短时长限制
Camera360：长按拍照按钮开始录制， 不能分段录制，录制有最短时长限制
B612咔叽： 长按拍照按钮录制，不能分段录制，有最短时长限制
天天P图： 长按拍照按钮录制，不能分段录制，有最短时长限制
FaceU激萌：支持分段录制，一段视频最小时间间隔为1秒钟，小于1秒钟无法操作
无他相机：支持分段录制，一段视频最小时间间隔为1秒钟，快速点击录制停止(500毫秒)时，会导致视频失效，保留之前的录制时长，如果视频时长过短，则提示视频太短。并且录制过程中，预览帧率下降。
可以看到，目前市面上还没有分段录制小于1秒钟的短视频方案，并且在仔细操作的时候发现，在切换状态时，存在预览画面停顿、录制时预览帧率降低等现象。实际上，我们可以从以上的相机中猜测，可以发现，短视频录制问题上，支持分段录制的相机应该都使用了MediaCodec + MediaMuxer 硬编码的方式进行录制的。这里算一下时间，MediaCodec在初始化时大约需要消耗200毫秒，销毁需要的时长也差不多。那么能否做到500毫秒录制一段视频？

3、优化思路
顺着上面的思路，我们思考一下，既然渲染操作和录制编码操作在同一个Looper中进行可能会导致帧率降低、预览画面卡一下等现象。那么，有没有办法将其拆分出去呢？理论上是可以做到的，这时候我们需要做并行化处理。录制编码的初始化，销毁等操作并不需要放在渲染线程所在的Looper 里面执行，可以放到另外一个Looper线程中处理，这样就能避免因MediaCodec 和 MediaMuxer的初始化所消耗的时长(200毫秒左右)而导致Looper 被阻塞，导致预览画面在此期间卡顿。另外一个问题就是MediaCodec编码和MediaMuxer复用器混合音视频时对RenderThread造成阻塞，导致预览帧率下降、预览卡顿等问题。 也就是说，我们如果能够实现多线程渲染，将音视频的录制编码以及混合部分拆解出来放到另外一个Looper线程中执行的话，以上的问题就不存在了。那么有没有办法实现呢？

答案是有的。由于MediaCodec在创建的时候会产生一个Surface，我们可以通过跨Looper线程的方式对Surface进行操作。既然是通过对Surface操作的，别说跨Looper线程执行，跨进程执行都可以实现。跨进程执行，这里不做详细的介绍，这里简单提一句，跨进程执行可以通过IBinder传递Surface，用Service来实现。跨Looper线程执行，需要注意一点是，一般录制的时候会添加水印，由于现在是通过硬编码的方式实现的，添加水印需要用到OpenGLES，因此，这里就得考虑OpenGLES的多线程渲染问题了。目前，关于OpenGLES多线程渲染的文章以及开源项目并不多，基本上只能参考Google的开源项目[Grafika](https://github.com/google/grafika)里面的内容。
Grafika中的Show + capture camera 例子就是使用OpenGLES多线程渲染实现录制的。在CameraCaptureActivity类里面，使用了GLSurfaceView，然后在绘制阶段，通过将GLSurfaceView 当前的EGLContext 共享给TextureMovieEncoder，将录制和预览渲染分离。下面是onDrawFrame 的实现源码：
```
    @Override
    public void onDrawFrame(GL10 unused) {
        if (VERBOSE) Log.d(TAG, "onDrawFrame tex=" + mTextureId);
        boolean showBox = false;

        // Latch the latest frame.  If there isn't anything new, we'll just re-use whatever
        // was there before.
        mSurfaceTexture.updateTexImage();

        // If the recording state is changing, take care of it here.  Ideally we wouldn't
        // be doing all this in onDrawFrame(), but the EGLContext sharing with GLSurfaceView
        // makes it hard to do elsewhere.
        // 开始录屏
        if (mRecordingEnabled) {
            switch (mRecordingStatus) {
                case RECORDING_OFF:
                    Log.d(TAG, "START recording");
                    mVideoEncoder.startRecording(new TextureMovieEncoder.EncoderConfig(
                            mOutputFile, 640, 480, 1000000, EGL14.eglGetCurrentContext()));
                    mRecordingStatus = RECORDING_ON;
                    break;
                case RECORDING_RESUMED:
                    Log.d(TAG, "RESUME recording");
                    // 更新EGL的Context，这里是渲染时使用了多线程的形式
                    mVideoEncoder.updateSharedContext(EGL14.eglGetCurrentContext());
                    mRecordingStatus = RECORDING_ON;
                    break;
                case RECORDING_ON:
                    break;
                default:
                    throw new RuntimeException("unknown status " + mRecordingStatus);
            }
        } else {
            // 录制状态判断
            switch (mRecordingStatus) {
                case RECORDING_ON:
                case RECORDING_RESUMED:
                    // 停止录制
                    Log.d(TAG, "STOP recording");
                    mVideoEncoder.stopRecording();
                    mRecordingStatus = RECORDING_OFF;
                    break;
                case RECORDING_OFF:
                    // yay
                    break;
                default:
                    throw new RuntimeException("unknown status " + mRecordingStatus);
            }
        }

        // Set the video encoder's texture name.  We only need to do this once, but in the
        // current implementation it has to happen after the video encoder is started, so
        // we just do it here.
        //
        // TODO: be less lame.
        mVideoEncoder.setTextureId(mTextureId);

        // Tell the video encoder thread that a new frame is available.
        // This will be ignored if we're not actually recording.
        mVideoEncoder.frameAvailable(mSurfaceTexture);

        if (mIncomingWidth <= 0 || mIncomingHeight <= 0) {
            // Texture size isn't set yet.  This is only used for the filters, but to be
            // safe we can just skip drawing while we wait for the various races to resolve.
            // (This seems to happen if you toggle the screen off/on with power button.)
            Log.i(TAG, "Drawing before incoming texture size set; skipping");
            return;
        }
        // 是否需要更新
        // Update the filter, if necessary.
        if (mCurrentFilter != mNewFilter) {
            updateFilter();
        }
        if (mIncomingSizeUpdated) {
            mFullScreen.getProgram().setTexSize(mIncomingWidth, mIncomingHeight);
            mIncomingSizeUpdated = false;
        }

        // Draw the video frame.
        mSurfaceTexture.getTransformMatrix(mSTMatrix);
        mFullScreen.drawFrame(mTextureId, mSTMatrix);

        // Draw a flashing box if we're recording.  This only appears on screen.
        showBox = (mRecordingStatus == RECORDING_ON);
        if (showBox && (++mFrameCount & 0x04) == 0) {
            drawBox();
        }
    }
```
我们可以发现，在onDrawFrame中通过 
```
mVideoEncoder.updateSharedContext(EGL14.eglGetCurrentContext());
```
将GLSurfaceView 的EGLContext 共享给了TextureMovieEncoder。那么TextureMovieEncoder 在接受到EGLContext 之后做了哪些操作？请看下面的代码：
```
private void handleUpdateSharedContext(EGLContext newSharedContext) {
        Log.d(TAG, "handleUpdatedSharedContext " + newSharedContext);

        // Release the EGLSurface and EGLContext.
        mInputWindowSurface.releaseEglSurface();
        mFullScreen.release(false);
        mEglCore.release();

        // Create a new EGLContext and recreate the window surface.
        mEglCore = new EglCore(newSharedContext, EglCore.FLAG_RECORDABLE);
        mInputWindowSurface.recreate(mEglCore);
        mInputWindowSurface.makeCurrent();

        // Create new programs and such for the new context.
        mFullScreen = new FullFrameRect(
                new Texture2dProgram(Texture2dProgram.ProgramType.TEXTURE_EXT));
    }
```
TextureMovieEncoder 在接收到EGLContext 后，重新创建一个EglCore类和一个WindowSurface，这两个是用来做渲染绘制用的。TextureMovieEncoder自身是一个Runnable，在run方法中自定义了一个Looper，并绑定EncoderHandle，如下所示：
```
    @Override
    public void run() {
        // Establish a Looper for this thread, and define a Handler for it.
        Looper.prepare();
        synchronized (mReadyFence) {
            mHandler = new EncoderHandler(this);
            mReady = true;
            mReadyFence.notify();
        }
        Looper.loop();

        Log.d(TAG, "Encoder thread exiting");
        synchronized (mReadyFence) {
            mReady = mRunning = false;
            mHandler = null;
        }
    }
```
通过上面的分析，Grafika项目中的Show + capture camera 例子其实是通过共享EGLContext的方式实现了OpenGLES多线程渲染。既然谷歌的开源项目有这样的写法。那么，我们就参考这样的方式来实现自己的多Looper线程共享EGLContext的方式实现MediaCodec + MediaMuxer 硬编码录制视频吧。

4、录制渲染Looper拆分与音视频录制混合
好了，前面我们讲到了渲染和音视频录制的拆分思路，以及分析了谷歌的开源项目Grafika中的OpenGLES多线程渲染实现方式。接下来我们来谈谈我们应该怎么实现录制渲染Looper线程拆分与音视频录制混合吧。首先，我们先从[AudioVideoRecordingSample](https://github.com/saki4510t/AudioVideoRecordingSample) 说起，AudioVideoRecordingSample 项目基本上是一套完整的录制过程，但是使用的Thread 并没有新开一个Looper，此时Thread 绑定的Looper 依旧是渲染线程的Looper 。这样的话，在录制阶段如果要对视频做其他的处理，则会造成帧率下降。为了解决这个问题，我们需要新开一个Looper线程，然后通过共享EGLContext 来将预览渲染部分和录制渲染部分拆分预览渲染的Looper线程RenderThread 和 录制渲染的Looper线程RecordThread。另外就是，AudioVideoRecordingSample 另外新开了一个线程给AudioRecord录音器进行录音，然后从AudioRecord中提取录音数据给音频的MediaCodec不断地喂数据。MediaMuxer 将音频MediaCodec 和视频 MediaCodec混合起来。这里的AudioRecordThread需要做一些细微的修改，保证录音器的状态跟录制状态一致，以避免录音状态出错。正题的架构如下图所示：
![无丢帧录制流程](http://upload-images.jianshu.io/upload_images/2103804-3e7c6a3471edf27f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


好了，废话不多数，我们来看看代码是如何实现的吧。首先是录制渲染Looper线程拆分与OpenGLES 多线程渲染以及共享EGLContext上下文如何实现。

首先是RecordManager录制管理器的实现。在RecordManager 录制管理类中，包含了一个自定义的带Looper 的RecordThread 以及绑定RecordThread的RecordHandler回调，录制管理类通过调用RecordThread 将视频录制的一些参数传递给EncoderManager编码管理器，同时，也将OpenGLES的EGLContext共享上下文传递给了EncoderManager编码管理器。这里的共享方式跟Grafika项目的共享方式是一致的，共享的EGLContext 是属于RenderThread所绑定的OpenGLES的EGLContext：
```
public final class RecordManager {

    private static final String TAG = "RecordManager";
    private static final boolean VERBOSE = false;

    public static final int RECORD_WIDTH = 540;
    public static final int RECORD_HEIGHT = 960;

    private static RecordManager mInstance;

    // 初始化录制器
    static final int MSG_INIT_RECORDER = 0;
    // 开始录制
    static final int MSG_START_RECORDING = 1;
    // 帧可用
    static final int MSG_FRAME_AVAILABLE = 2;
    // 渲染帧
    static final int MSG_DRAW_FRAME = 3;
    // 停止录制
    static final int MSG_STOP_RECORDING = 4;
    // 暂停录制
    static final int MSG_PAUSE_RECORDING = 5;
    // 继续录制
    static final int MSG_CONTINUE_RECORDING = 6;
    // 设置帧率
    static final int MSG_FRAME_RATE = 7;
    // 是否允许录制高清视频
    static final int MSG_HIGHTDEFINITION = 8;
    // 是否允许录制
    static final int MSG_ENABLE_AUDIO = 9;
    // 退出
    static final int MSG_QUIT = 10;
    // 设置渲染Texture的宽高
    static final int MSG_SET_TEXTURE_SIZE = 11;
    // 设置预览大小
    static final int MSG_SET_DISPLAY_SIZE = 12;

    // 录制线程
    private RecordThread mRecordThread;

    private String mOutputPath;

    public static RecordManager getInstance() {
        if (mInstance == null) {
            mInstance = new RecordManager();
        }
        return mInstance;
    }

    private RecordManager() {}

    /**
     * 初始化录制线程
     */
    public void initThread() {
        mRecordThread = new RecordThread();
        mRecordThread.start();
        mRecordThread.waitUntilReady();
    }

    /**
     * 初始化录制器，此时耗时大约200ms左右，不能放在跟渲染线程同一个Looper里面
     * @param width
     * @param height
     */
    public void initRecorder(int width, int height) {
        initRecorder(width, height, null);
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
     * 设置预览大小
     * @param width
     * @param height
     */
    public void setDisplaySize(int width, int height) {
        Handler handler = mRecordThread.getHandler();
        if (handler != null) {
            handler.sendMessage(handler.obtainMessage(MSG_SET_DISPLAY_SIZE, width, height));
        }
    }

    /**
     * 开始录制
     * @param sharedContext EGLContext上下文包装类
     */
    public void startRecording(EGLContext sharedContext) {
        Handler handler = mRecordThread.getHandler();
        if (handler != null) {
            handler.sendMessage(handler.obtainMessage(MSG_START_RECORDING, sharedContext));
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
            handler.sendMessage(handler.obtainMessage(MSG_QUIT));
        }
        mRecordThread = null;
    }


    /**
     * 暂停录制
     */
    public void pauseRecording() {
        Handler handler = mRecordThread.getHandler();
        if (handler != null) {
            handler.sendMessage(handler.obtainMessage(MSG_PAUSE_RECORDING));
        }
    }

    /**
     * 继续录制
     */
    public void continueRecording() {
        Handler handler = mRecordThread.getHandler();
        if (handler != null) {
            handler.sendMessage(handler.obtainMessage(MSG_CONTINUE_RECORDING));
        }
    }


    /**
     * 设置帧率
     * @param frameRate
     */
    public void setFrameRate(int frameRate) {
        Handler handler = mRecordThread.getHandler();
        if (handler != null) {
            handler.sendMessage(handler.obtainMessage(MSG_FRAME_RATE, frameRate));
        }
    }


    /**
     * 是否允许录制高清视频
     * @param enable
     */
    public void enableHighDefinition(boolean enable) {
        Handler handler = mRecordThread.getHandler();
        if (handler != null) {
            handler.sendMessage(handler.obtainMessage(MSG_HIGHTDEFINITION, enable));
        }
    }

    /**
     * 是否允许录音
     * @param enable
     */
    public void setEnableAudioRecording(boolean enable) {
        Handler handler = mRecordThread.getHandler();
        if (handler != null) {
            handler.sendMessage(handler.obtainMessage(MSG_ENABLE_AUDIO, enable));
        }
    }

    /**
     * 获取输出路径
     * @return
     */
    public String getOutputPath() {
        return mOutputPath;
    }

    /**
     * 设置输出路径
     * @param path
     */
    public void setOutputPath(String path) {
        mOutputPath = path;
        EncoderManager.getInstance().setOutputPath(path);
    }

    /**
     * 录制线程
     */
    private static class RecordThread extends Thread {

        // 录制线程Handler回调
        private RecordHandler mHandler;

        private Object mReadyFence = new Object();
        private boolean mReady;


        @Override
        public void run() {
            Looper.prepare();
            synchronized (mReadyFence) {
                mHandler = new RecordHandler(this);
                mReady = true;
                mReadyFence.notify();
            }
            Looper.loop();
            if (VERBOSE) {
                Log.d(TAG, "Record thread exiting");
            }

            synchronized (mReadyFence) {
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
                EncoderManager.getInstance().initRecorder(width, height, listener);
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
                EncoderManager.getInstance().setTextureSize(width, height);
            }
        }

        /**
         * 设置预览大小
         * @param width
         * @param height
         */
        void setDisplaySize(int width, int height) {
            if (VERBOSE) {
                Log.d(TAG, "setDisplaySize");
            }
            synchronized (mReadyFence) {
                EncoderManager.getInstance().setDisplaySize(width, height);
            }
        }

        /**
         * 开始录制
         * @param eglContext EGLContext上下文包装类
         */
        void startRecording(EGLContext eglContext) {
            if (VERBOSE) {
                Log.d(TAG, " start recording");
            }
            synchronized (mReadyFence) {
                EncoderManager.getInstance().startRecording(eglContext);
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
                EncoderManager.getInstance().frameAvailable();
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
                EncoderManager.getInstance().drawRecorderFrame(currentTexture, timeStamp);
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
                EncoderManager.getInstance().stopRecording();
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
                EncoderManager.getInstance().pauseRecording();
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
                EncoderManager.getInstance().continueRecording();
            }
        }


        /**
         * 设置帧率
         * @param frameRate
         */
        void setFrameRate(int frameRate) {
            if (VERBOSE) {
                Log.d(TAG, "set frame rate: " + frameRate);
            }
            synchronized (mReadyFence) {
                EncoderManager.getInstance().setFrameRate(frameRate);
            }
        }


        /**
         * 是否允许录制高清视频
         * @param enable
         */
        void enableHighDefinition(boolean enable) {
            if (VERBOSE) {
                Log.d(TAG, "enable highDefinition ? " + enable);
            }

            synchronized (mReadyFence) {
                EncoderManager.getInstance().enableHighDefinition(enable);
            }
        }

        /**
         * 是否允许录音
         * @param enable
         */
        void setEnableAudioRecording(boolean enable) {
            if (VERBOSE) {
                Log.d(TAG, "enable audio recording ? " + enable);
            }
            synchronized (mReadyFence) {
                EncoderManager.getInstance().setEnableAudioRecording(enable);
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

                // 开始录制
                case MSG_START_RECORDING:
                    thread.startRecording((EGLContext) msg.obj);
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

                // 设置帧率
                case MSG_FRAME_RATE:
                    thread.setFrameRate((Integer) msg.obj);
                    break;

                // 是否允许高清录制
                case MSG_HIGHTDEFINITION:
                    thread.enableHighDefinition((Boolean) msg.obj);
                    break;

                // 是否允许录音
                case MSG_ENABLE_AUDIO:
                    thread.setEnableAudioRecording((Boolean) msg.obj);
                    break;

                // 退出线程
                case MSG_QUIT:
                    Looper.myLooper().quit();
                    break;

                // 设置渲染Texture的宽高
                case MSG_SET_TEXTURE_SIZE:
                    thread.setTextureSize(msg.arg1, msg.arg2);
                    break;

                // 设置预览的大小
                case MSG_SET_DISPLAY_SIZE:
                    thread.setDisplaySize(msg.arg1, msg.arg2);
                    break;

                default:
                    throw new RuntimeException("Unhandled msg what = " + what);
            }
        }
    }

}
```
接下来我们看看EncoderManager编码管理器中的实现。编码管理器主要用于管理MediaMuxer、MediaCodec、录制渲染所需要的WindowSurface、EglCore等的初始化以及销毁等操作。根据RecordManager传递过来的OpenGLES共享上下文EGLContext，我们需要重新创建一个EglCore 和 录制用的WindowSurface，录制渲染部分需要用到，在drawRecorderFrame中，我们通过WindowSurface切换到当前线程的共享上下文状态下，做录制渲染绘制工作。这里通过handler将渲染部分的Texture发送给当前线程进行绘制，这里有个细节需要注意的是，由于OpenGLES 不是线程安全的，多线程渲染是通过TLS(Thread Local Storage)机制实现的，因此这里的Texture不能跟RenderManager共用，必须通过handler发送给录制的HandlerThread中存储起来，这样在录制线程渲染完之前，RenderManager可以渲染不同的Texture，如果共用，那么这里会产生录制一闪一闪的情况。
```
public class EncoderManager {

    private static final String TAG = "EncoderManager";

    private static EncoderManager mInstance;

    // 录制比特率
    private int mRecordBitrate;
    // 录制帧率
    private int mFrameRate = 25;
    // 像素资料量
    private int mBPP = 4;

    // 是否允许高清视频
    private boolean mEnableHD = false;
    // 码率乘高清值
    private int HDValue = 16;

    // 渲染Texture的宽度
    private int mTextureWidth;
    // 渲染Texture的高度
    private int mTextureHeight;
    // 视频宽度
    private int mVideoWidth;
    // 视频高度
    private int mVideoHeight;
    // 显示宽度
    private int mDisplayWidth;
    // 显示高度
    private int mDisplayHeight;
    // 缩放方式
    private ScaleType mScaleType = ScaleType.CENTER_CROP;

    private EglCore mEglCore;
    // 录制视频用的EGLSurface
    private WindowSurface mRecordWindowSurface;
    // 录制的Filter
    private DisplayFilter mRecordFilter;

    // 复用器管理器
    private MediaMuxerWrapper mMuxerManager;

    // 录制文件路径
    private String mRecorderOutputPath = null;

    // 是否允许录音
    private boolean isEnableAudioRecording = true;

    // 是否处于录制状态
    private boolean isRecording = false;

    public static EncoderManager getInstance() {
        if (mInstance == null) {
            mInstance = new EncoderManager();
        }
        return mInstance;
    }

    private EncoderManager() {

    }

    /**
     * 初始化录制器，此时耗时大约280ms左右
     * 如果放在渲染线程里执行，会导致一开始录制出来的视频开头严重掉帧
     * @param width
     * @param height
     */
    synchronized public void initRecorder(int width, int height) {
        initRecorder(width, height, null);
    }

    /**
     * 初始化录制器，耗时大约208ms左右
     * 如果放在渲染线程里面执行，会导致一开始录制出来的视频开头严重掉帧
     * @param width
     * @param height
     * @param listener
     */
    synchronized public void initRecorder(int width, int height,
                                          MediaEncoder.MediaEncoderListener listener) {
        mVideoWidth = width;
        mVideoHeight = height;
        // 如果路径为空，则生成默认的路径
        if (mRecorderOutputPath == null || mRecorderOutputPath.isEmpty()) {
            mRecorderOutputPath = ParamsManager.VideoPath
                    + "CainCamera_" + System.currentTimeMillis() + ".mp4";
            Log.d(TAG, "the outpath is empty, auto-created path is : " + mRecorderOutputPath);
        }
        File file = new File(mRecorderOutputPath);
        if (!file.getParentFile().exists()) {
            file.getParentFile().mkdirs();
        }
        // 计算帧率
        mRecordBitrate = width * height * mFrameRate / mBPP;
        if (mEnableHD) {
            mRecordBitrate *= HDValue;
        }
        try {

            mMuxerManager = new MediaMuxerWrapper(file.getAbsolutePath());
            new MediaVideoEncoder(mMuxerManager, listener, mVideoWidth, mVideoHeight);
            if (isEnableAudioRecording) {
                new MediaAudioEncoder(mMuxerManager, listener);
            }

            mMuxerManager.prepare();
        } catch (IOException e) {
            Log.e(TAG, "startRecording:", e);
        }
    }

    /**
     * 设置渲染Texture的宽高
     * @param width
     * @param height
     */
    public void setTextureSize(int width, int height) {
        mTextureWidth = width;
        mTextureHeight = height;
    }

    /**
     * 设置预览大小
     * @param width
     * @param height
     */
    public void setDisplaySize(int width, int height) {
        mDisplayWidth = width;
        mDisplayHeight = height;
    }

    /**
     * 调整视口大小
     */
    private void updateViewport() {
        float[] mvpMatrix = GlUtil.IDENTITY_MATRIX;
        if (mVideoWidth == 0 || mVideoHeight == 0) {
            mVideoWidth = mTextureWidth;
            mVideoHeight = mTextureHeight;
        }
        final double scale_x = mDisplayWidth / mVideoWidth;
        final double scale_y = mDisplayHeight / mVideoHeight;
        final double scale = (mScaleType == ScaleType.CENTER_CROP)
                ? Math.max(scale_x,  scale_y) : Math.min(scale_x, scale_y);
        final double width = scale * mVideoWidth;
        final double height = scale * mVideoHeight;
        Matrix.scaleM(mvpMatrix, 0, (float)(width / mDisplayWidth),
                (float)(height / mDisplayHeight), 1.0f);
        if (mRecordFilter != null) {
            mRecordFilter.setMVPMatrix(mvpMatrix);
        }
    }

    /**
     * 开始录制，共享EglContext实现多线程录制
     */
    public synchronized void startRecording(EGLContext eglContext) {
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
        initRecordingFilter();
        updateViewport();
        if (mMuxerManager != null) {
            mMuxerManager.startRecording();
        }
        isRecording = true;
    }

    /**
     * 帧可用时调用
     */
    public void frameAvailable() {
        if (mMuxerManager != null && mMuxerManager.getVideoEncoder() != null && isRecording) {
            mMuxerManager.getVideoEncoder().frameAvailableSoon();
        }
    }

    /**
     * 发送渲染指令
     * @param currentTexture 当前Texture
     * @param timeStamp 时间戳
     */
    public void drawRecorderFrame(int currentTexture, long timeStamp) {
        if (mRecordWindowSurface != null) {
            mRecordWindowSurface.makeCurrent();
            drawRecordingFrame(currentTexture);
            mRecordWindowSurface.setPresentationTime(timeStamp);
            mRecordWindowSurface.swapBuffers();
        }
    }

    /**
     * 停止录制
     */
    public synchronized void stopRecording() {
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
    }

    /**
     * 暂停录制
     */
    public synchronized void pauseRecording() {
        if (mMuxerManager != null && isRecording) {
            mMuxerManager.pauseRecording();
        }
    }

    /**
     * 继续录制
     */
    public synchronized void continueRecording() {
        if (mMuxerManager != null && isRecording) {
            mMuxerManager.continueRecording();
        }
    }

    /**
     * 初始化录制的Filter
     * TODO 录制视频大小跟渲染大小、显示大小拆分成不同的大小
     */
    private void initRecordingFilter() {
        if (mRecordFilter == null) {
            mRecordFilter = new DisplayFilter();
        }
        mRecordFilter.onInputSizeChanged(mTextureWidth, mTextureHeight);
        mRecordFilter.onDisplayChanged(mVideoWidth, mVideoHeight);
    }

    /**
     * 渲染录制的帧
     */
    public void drawRecordingFrame(int textureId) {
        if (mRecordFilter != null) {
            GLES30.glViewport(0, 0, mVideoWidth, mVideoHeight);
            mRecordFilter.drawFrame(textureId);
        }
    }

    /**
     * 释放录制的Filter资源
     */
    public void releaseRecordingFilter() {
        if (mRecordFilter != null) {
            mRecordFilter.release();
            mRecordFilter = null;
        }
    }

    /**
     * 销毁资源
     */
    public void release() {
        releaseRecordingFilter();
        if (mEglCore != null) {
            mEglCore.release();
            mEglCore = null;
        }
        if (mRecordWindowSurface != null) {
            mRecordWindowSurface.release();
            mRecordWindowSurface = null;
        }
    }

    /**
     * 设置视频帧率
     * @param frameRate
     */
    public void setFrameRate(int frameRate) {
        mFrameRate = frameRate;
    }

    /**
     * 是否允许录制高清视频
     * @param enable
     */
    public void enableHighDefinition(boolean enable) {
        mEnableHD = enable;
    }


    /**
     * 是否允许录音
     * @param enable
     */
    public void setEnableAudioRecording(boolean enable) {
        isEnableAudioRecording = enable;
    }

    /**
     * 设置输出路径
     * @param path
     * @return
     */
    public void setOutputPath(String path) {
        mRecorderOutputPath = path;
    }
}

```
这里我使用了[AudioVideoRecordingSample](https://github.com/saki4510t/AudioVideoRecordingSample) 开源项目的代码，并且在此基础上做了一些修改，以适应我的项目。详情请参考本人的开源相机项目：
 [CainCamera](https://github.com/CainKernel/CainCamera)
到这里，我们实现了短视频的分段录制。由于使用了OpenGLES多线程渲染，录制时也没有出现帧率降低的情况。快速点击录制和停止，你会发现，录制出来的视频甚至可以做到只有一帧的效果，我在Nexus 5X上快读点击录制播放，在9秒内共录制了15个视频，平均一个视频600毫秒，这里还是手动操作的。点击预览，使用ExoPlayer 将录制的多段视频播放出来，你会发现，视频并没有出现丢帧的情况。分段录制功能基本完美实现。

备注：截止本文章发布时，项目中的录制按钮还存在一些小Bug，导致录制功能并不是非常完美。但这并不影响分段录制功能。另外就是，多段视频合成功能还没有实现。后面等我做完多段视频合成功能后，我会再写一篇文章介绍如何高效地对视频进行合成、通过视频生成GIF，并且讨论GIF生成以及八叉树色彩量化、扩散等方面的优化。（2017年12月14日更新，目前视频合成部分已经实现，还剩下GIF部分没有实现）
