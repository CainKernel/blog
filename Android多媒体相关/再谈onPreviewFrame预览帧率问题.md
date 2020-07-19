在之前，本人写了一篇文章（[关于Android Camera onPreviewFrame 预览回调帧率问题](http://www.jianshu.com/p/a33b1eabe71c)），说了关于高通和MTK CPU在单双HandlerThread控制Camera和Rendering上的差异。我觉得有必要详细说明一下onPreviewFrame在不同情况下，可能会对帧率产生影响的问题。对此，本人重新梳理了一下，详细讨论一下该如何确保onPreviewFrame回调帧率。
首先，我们来复现一下双HandlerThread分别控制相机和渲染的方式导致(大)部分MTK 的CPU 的onPreviewFrame回调帧率大幅度降低的情况：
首先是Camera控制类，用于管理相机的打开关闭等操作：
```
public final class CameraManager {

    private static CameraManager mInstance;

    // 相机默认宽高，相机的宽度和高度跟屏幕坐标不一样，手机屏幕的宽度和高度是反过来的。
    public final int DEFAULT_WIDTH = 1280;
    public final int DEFAULT_HEIGHT = 720;
    // 期望fps
    public final int DESIRED_PREVIEW_FPS = 30;

    // 这里反过来是因为相机的分辨率跟屏幕的分辨率宽高刚好反过来
    public final float Ratio_4_3 = 0.75f;
    public final float Ratio_1_1 = 1.0f;
    public final float Ratio_16_9 = 0.5625f;

    private int mCameraID = Camera.CameraInfo.CAMERA_FACING_FRONT;
    private Camera mCamera;
    private int mCameraPreviewFps;
    private int mOrientation = 0;

    // 当前的宽高比
    private float mCurrentRatio = Ratio_16_9;

    /**
     * 获取单例
     * @return
     */
    public static CameraManager getInstance() {
        if (mInstance == null) {
            mInstance = new CameraManager();
        }
        return mInstance;
    }

    private CameraManager() {}


    /**
     * 打开相机
     */
    public void openCamera() {
        openCamera(DESIRED_PREVIEW_FPS);
    }

    /**
     * 打开相机，默认打开前置相机
     * @param expectFps
     */
    public void openCamera(int expectFps) {
        openCamera(mCameraID, expectFps);
    }

    /**
     * 根据ID打开相机
     * @param cameraID
     * @param expectFps
     */
    public void openCamera(int cameraID, int expectFps) {
        openCamera(cameraID, expectFps, DEFAULT_WIDTH, DEFAULT_HEIGHT);
    }

    /**
     *  打开相机
     * @param cameraID
     * @param expectFps
     * @param expectWidth
     * @param expectHeight
     */
    public void openCamera(int cameraID, int expectFps, int expectWidth, int expectHeight) {
        if (mCamera != null) {
            throw new RuntimeException("camera already initialized!");
        }
        mCamera = Camera.open(cameraID);
        if (mCamera == null) {
            throw new RuntimeException("Unable to open camera");
        }
        mCameraID = cameraID;
        Camera.Parameters parameters = mCamera.getParameters();
        mCameraPreviewFps = chooseFixedPreviewFps(parameters, expectFps * 1000);
        parameters.setRecordingHint(true);
        mCamera.setParameters(parameters);
        setPreviewSize(mCamera, expectWidth, expectHeight);
        setPictureSize(mCamera, expectWidth, expectHeight);
        mCamera.setDisplayOrientation(mOrientation);
    }

    /**
     * 重新打开相机
     */
    public void reopenCamera() {
        releaseCamera();
        openCamera(mCameraID, DESIRED_PREVIEW_FPS);
    }

    /**
     * 重新打开相机
     * @param expectFps
     */
    public void reopenCamera(int expectFps) {
        releaseCamera();
        openCamera(mCameraID, expectFps);
    }

    /**
     * 重新打开相机
     * @param expectFps
     * @param expectWidth
     * @param expectHeight
     */
    public void reopenCamera(int expectFps, int expectWidth, int expectHeight) {
        releaseCamera();
        openCamera(mCameraID, expectFps, expectWidth, expectHeight);
    }


    /**
     * 设置预览的Surface
     * @param texture
     */
    public void setPreviewSurface(SurfaceTexture texture) {
        if (mCamera != null) {
            try {
                mCamera.setPreviewTexture(texture);
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }

    /**
     * 设置预览的Surface
     * @param holder
     */
    public void setPreviewSurface(SurfaceHolder holder) {
        if (mCamera != null) {
            try {
                mCamera.setPreviewDisplay(holder);
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }

    /**
     * 添加预览回调 备注：预览回调需要在setPreviewSurface之后调用
     * @param callback
     * @param previewBuffer
     */
    public void setPreviewCallbackWithBuffer(Camera.PreviewCallback callback, byte[] previewBuffer) {
        if (mCamera != null) {
            mCamera.setPreviewCallbackWithBuffer(callback);
            mCamera.addCallbackBuffer(previewBuffer);
        }
    }

    /**
     * 开始预览
     */
    public void startPreview() {
        if (mCamera != null) {
            mCamera.startPreview();
        }
    }

    /**
     * 停止预览
     */
    public void stopPreview() {
        if (mCamera != null) {
            mCamera.stopPreview();
        }
    }

    /**
     * 切换相机
     * @param cameraID 相机Id
     */
    public void switchCamera(int cameraID) {
        switchCamera(cameraID, DESIRED_PREVIEW_FPS);
    }

    /**
     * 切换相机
     * @param cameraId 相机Id
     * @param expectFps 期望帧率
     */
    public void switchCamera(int cameraId, int expectFps) {
        switchCamera(cameraId, expectFps, DEFAULT_WIDTH, DEFAULT_HEIGHT);
    }

    /**
     * 切换相机
     * @param cameraId      相机Id
     * @param expectFps     期望帧率
     * @param expectWidth   期望宽度
     * @param expectHeight  期望高度
     */
    public void switchCamera(int cameraId, int expectFps, int expectWidth, int expectHeight) {
        if (mCameraID == cameraId) {
            return;
        }
        mCameraID = cameraId;
        releaseCamera();
        openCamera(cameraId, expectFps, expectWidth, expectHeight);
    }

    /**
     * 切换相机并预览
     * @param cameraId 相机Id
     * @param holder    SurfaceHolder
     * @param callback  回调
     * @param buffer    缓冲
     */
    public void switchCameraAndPreview(int cameraId, SurfaceHolder holder,
                                       Camera.PreviewCallback callback, byte[] buffer) {
        switchCamera(cameraId);
        setPreviewSurface(holder);
        setPreviewCallbackWithBuffer(callback, buffer);
        startPreview();
    }

    /**
     * 切换相机并预览
     * @param cameraId 相机Id
     * @param texture    SurfaceTexture
     * @param callback  回调
     * @param buffer    缓冲
     */
    public void switchCameraAndPreview(int cameraId, SurfaceTexture texture,
                                       Camera.PreviewCallback callback, byte[] buffer) {
        switchCamera(cameraId);
        setPreviewSurface(texture);
        setPreviewCallbackWithBuffer(callback, buffer);
        startPreview();
    }

    /**
     * 释放相机
     */
    public void releaseCamera() {
        if (mCamera != null) {
            mCamera.setPreviewCallbackWithBuffer(null);
            mCamera.stopPreview();
            mCamera.release();
            mCamera = null;
        }
    }

    /**
     * 拍照
     */
    public void takePicture(Camera.ShutterCallback shutterCallback,
                                   Camera.PictureCallback rawCallback,
                                   Camera.PictureCallback pictureCallback) {
        if (mCamera != null) {
            mCamera.takePicture(shutterCallback, rawCallback, pictureCallback);
        }
    }

    /**
     * 设置预览大小
     * @param camera
     * @param expectWidth
     * @param expectHeight
     */
    private void setPreviewSize(Camera camera, int expectWidth, int expectHeight) {
        Camera.Parameters parameters = camera.getParameters();
        Camera.Size size = calculatePerfectSize(parameters.getSupportedPreviewSizes(),
                expectWidth, expectHeight);
        parameters.setPreviewSize(size.width, size.height);
        camera.setParameters(parameters);
        Log.d("setPreviewSize", "width = " + size.width + ", height = " + size.height);
    }

    /**
     * 设置拍摄的照片大小
     * @param camera
     * @param expectWidth
     * @param expectHeight
     */
    private void setPictureSize(Camera camera, int expectWidth, int expectHeight) {
        Camera.Parameters parameters = camera.getParameters();
        Camera.Size size = calculatePerfectSize(parameters.getSupportedPictureSizes(),
                expectWidth, expectHeight);
        parameters.setPictureSize(size.width, size.height);
        camera.setParameters(parameters);
        Log.d("setPictureSize", "width = " + size.width + ", height = " + size.height);
    }


    /**
     * 设置预览角度，setDisplayOrientation本身只能改变预览的角度
     * previewFrameCallback以及拍摄出来的照片是不会发生改变的，拍摄出来的照片角度依旧不正常的
     * 拍摄的照片需要自行处理
     * 这里Nexus5X的相机简直没法吐槽，后置摄像头倒置了，切换摄像头之后就出现问题了。
     * @param activity
     */
    public int calculateCameraPreviewOrientation(Activity activity) {
        Camera.CameraInfo info = new Camera.CameraInfo();
        Camera.getCameraInfo(mCameraID, info);
        int rotation = activity.getWindowManager().getDefaultDisplay()
                .getRotation();
        int degrees = 0;
        switch (rotation) {
            case Surface.ROTATION_0:
                degrees = 0;
                break;
            case Surface.ROTATION_90:
                degrees = 90;
                break;
            case Surface.ROTATION_180:
                degrees = 180;
                break;
            case Surface.ROTATION_270:
                degrees = 270;
                break;
        }

        int result;
        if (info.facing == Camera.CameraInfo.CAMERA_FACING_FRONT) {
            result = (info.orientation + degrees) % 360;
            result = (360 - result) % 360;
        } else {
            result = (info.orientation - degrees + 360) % 360;
        }
        mOrientation = result;
        return result;
    }

    //-------------------------------- setter and getter start -------------------------------------

    /**
     * 获取当前的Camera ID
     * @return
     */
    public int getCameraID() {
        return mCameraID;
    }

    /**
     * 获取照片大小
     * @return
     */
    public Size getPictureSize() {
        if (mCamera != null) {
            Camera.Size size = mCamera.getParameters().getPictureSize();
            Size result = new Size(size.width, size.height);
            return result;
        }
        return new Size(0, 0);
    }

    /**
     * 获取预览大小
     * @return
     */
    public Size getPreviewSize() {
        if (mCamera != null) {
            Camera.Size size = mCamera.getParameters().getPreviewSize();
            Size result = new Size(size.width, size.height);
            return result;
        }
        return new Size(0, 0);
    }

    /**
     * 获取相机信息
     * @return
     */
    public CameraInfo getCameraInfo() {
        if (mCamera != null) {
            Camera.CameraInfo info = new Camera.CameraInfo();
            Camera.getCameraInfo(mCameraID, info);
            CameraInfo result = new CameraInfo(info.facing, info.orientation);
            return result;
        }
        return null;
    }

    /**
     * 获取当前预览的角度
     * @return
     */
    public int getPreviewOrientation() {
        return mOrientation;
    }

    /**
     * 获取FPS（千秒值）
     * @return
     */
    public int getCameraPreviewThousandFps() {
        return mCameraPreviewFps;
    }

    /**
     * 获取当前的宽高比
     * @return
     */
    public float getCurrentRatio() {
        return mCurrentRatio;
    }


    //---------------------------------- setter and getter end -------------------------------------

    /**
     * 计算最完美的Size
     * @param sizes
     * @param expectWidth
     * @param expectHeight
     * @return
     */
    private Camera.Size calculatePerfectSize(List<Camera.Size> sizes, int expectWidth,
                                                    int expectHeight) {
        sortList(sizes); // 根据宽度进行排序

        // 根据当前期望的宽高判定
        List<Camera.Size> bigEnough = new ArrayList<>();
        List<Camera.Size> noBigEnough = new ArrayList<>();
        for (Camera.Size size : sizes) {
            if (size.height * expectWidth / expectHeight == size.width) {
                if (size.width >= expectWidth && size.height >= expectHeight) {
                    bigEnough.add(size);
                } else {
                    noBigEnough.add(size);
                }
            }
        }
        if (bigEnough.size() > 0) {
            return Collections.min(bigEnough, new CompareAreaSize());
        } else if (noBigEnough.size() > 0) {
            return Collections.max(noBigEnough, new CompareAreaSize());
        } else { // 如果不存在满足要求的数值，则辗转计算宽高最接近的值
            Camera.Size result = sizes.get(0);
            boolean widthOrHeight = false; // 判断存在宽或高相等的Size
            // 辗转计算宽高最接近的值
            for (Camera.Size size : sizes) {
                // 如果宽高相等，则直接返回
                if (size.width == expectWidth && size.height == expectHeight
                        && ((float) size.height / (float) size.width) == mCurrentRatio) {
                    result = size;
                    break;
                }
                // 仅仅是宽度相等，计算高度最接近的size
                if (size.width == expectWidth) {
                    widthOrHeight = true;
                    if (Math.abs(result.height - expectHeight)
                            > Math.abs(size.height - expectHeight)
                            && ((float) size.height / (float) size.width) == mCurrentRatio) {
                        result = size;
                        break;
                    }
                }
                // 高度相等，则计算宽度最接近的Size
                else if (size.height == expectHeight) {
                    widthOrHeight = true;
                    if (Math.abs(result.width - expectWidth)
                            > Math.abs(size.width - expectWidth)
                            && ((float) size.height / (float) size.width) == mCurrentRatio) {
                        result = size;
                        break;
                    }
                }
                // 如果之前的查找不存在宽或高相等的情况，则计算宽度和高度都最接近的期望值的Size
                else if (!widthOrHeight) {
                    if (Math.abs(result.width - expectWidth)
                            > Math.abs(size.width - expectWidth)
                            && Math.abs(result.height - expectHeight)
                            > Math.abs(size.height - expectHeight)
                            && ((float) size.height / (float) size.width) == mCurrentRatio) {
                        result = size;
                    }
                }
            }
            return result;
        }
    }

    /**
     * 分辨率由大到小排序
     * @param list
     */
    private void sortList(List<Camera.Size> list) {
        Collections.sort(list, new CompareAreaSize());
    }

    /**
     * 比较器
     */
    private class CompareAreaSize implements Comparator<Camera.Size> {
        @Override
        public int compare(Camera.Size pre, Camera.Size after) {
            return Long.signum((long) pre.width * pre.height -
                    (long) after.width * after.height);
        }
    }

    /**
     * 选择合适的FPS
     * @param parameters
     * @param expectedThoudandFps 期望的FPS
     * @return
     */
    private int chooseFixedPreviewFps(Camera.Parameters parameters, int expectedThoudandFps) {
        List<int[]> supportedFps = parameters.getSupportedPreviewFpsRange();
        for (int[] entry : supportedFps) {
            if (entry[0] == entry[1] && entry[0] == expectedThoudandFps) {
                parameters.setPreviewFpsRange(entry[0], entry[1]);
                return entry[0];
            }
        }
        int[] temp = new int[2];
        int guess;
        parameters.getPreviewFpsRange(temp);
        if (temp[0] == temp[1]) {
            guess = temp[0];
        } else {
            guess = temp[1] / 2;
        }
        return guess;
    }


}
```
接着，我们来看看控制Camera相机的HandlerThread线程，如下：
```
public class CameraHandlerThread extends HandlerThread {

    private static final String TAG = "CameraHandlerThread";

    private final Handler mHandler;

    public CameraHandlerThread() {
        super(TAG);
        start();
        mHandler = new Handler(getLooper());
    }

    public CameraHandlerThread(String name) {
        super(name);
        start();
        mHandler = new Handler(getLooper());
    }

    /**
     * 销毁线程
     */
    public void destoryThread() {
        releaseCamera();
        mHandler.removeCallbacksAndMessages(null);
        quitSafely();
    }


    /**
     * 检查handler是否可用
     */
    private void checkHandleAvailable() {
        if (mHandler == null) {
            throw new NullPointerException("Handler is not available!");
        }
    }

    /**
     * 等待操作完成
     */
    private void waitUntilReady() {
        try {
            wait();
        } catch (InterruptedException e) {
            Log.w(TAG, "wait was interrupted");
        }
    }

    /**
     * 打开相机
     */
    synchronized public void openCamera() {
        checkHandleAvailable();
        mHandler.post(new Runnable() {
            @Override
            public void run() {
                internalOpenCamera();
                notifyCameraOpened();
            }
        });
        waitUntilReady();
    }

    /**
     * 打开相机
     * @param expectFps 期望帧率
     */
    synchronized public void openCamera(final int expectFps) {
        checkHandleAvailable();
        mHandler.post(new Runnable() {
            @Override
            public void run() {
                internalOpenCamera(expectFps);
                notifyCameraOpened();
            }
        });
        waitUntilReady();
    }

    /**
     * 打开相机
     * @param cameraId  相机Id
     * @param expectFps 期望帧率
     */
    synchronized public void openCamera(final int cameraId, final int expectFps) {
        checkHandleAvailable();
        mHandler.post(new Runnable() {
            @Override
            public void run() {
                internalOpenCamera(cameraId, expectFps);
                notifyCameraOpened();
            }
        });
        waitUntilReady();
    }

    /**
     * 打开相机
     * @param cameraId      相机Id
     * @param expectFps     期望帧率
     * @param expectWidth   期望宽度
     * @param expectHeight  期望高度
     */
    synchronized public void openCamera(final int cameraId, final int expectFps,
                                        final int expectWidth, final int expectHeight) {
        checkHandleAvailable();
        mHandler.post(new Runnable() {
            @Override
            public void run() {
                internalOpenCamera(cameraId, expectFps, expectWidth, expectHeight);
                notifyCameraOpened();
            }
        });
        waitUntilReady();
    }


    /**
     * 通知相机已打开，主要的作用是，如果在打开之后要立即获得mCamera实例，则需要添加wait()-notify()
     * wait() - notify() 不是必须的
     */
    synchronized private void notifyCameraOpened() {
        notify();
    }


    /**
     * 重新打开相机
     */
    synchronized public void reopenCamera() {
        checkHandleAvailable();
        mHandler.post(new Runnable() {
            @Override
            public void run() {
                internalReopenCamera();
                notifyCameraOpened();
            }
        });
        waitUntilReady();
    }

    /**
     * 重新打开相机
     * @param expectFps 期望帧率
     */
    synchronized public void reopenCamera(final int expectFps) {
        checkHandleAvailable();
        mHandler.post(new Runnable() {
            @Override
            public void run() {
                internalReopenCamera(expectFps);
                notifyCameraOpened();
            }
        });
        waitUntilReady();
    }


    /**
     * 重新打开相机
     * @param expectFps     期望帧率
     * @param expectWidth   期望宽度
     * @param expectHeight  期望高度
     */
    synchronized public void reopenCamera(final int expectFps,
                                          final int expectWidth, final int expectHeight) {
        checkHandleAvailable();
        mHandler.post(new Runnable() {
            @Override
            public void run() {
                internalReopenCamera(expectFps, expectWidth, expectHeight);
                notifyCameraOpened();
            }
        });
        waitUntilReady();
    }

    /**
     * 设置预览Surface
     * @param holder SurfaceHolder
     */
    public void setPreviewSurface(final SurfaceHolder holder) {
        checkHandleAvailable();
        mHandler.post(new Runnable() {
            @Override
            public void run() {
                internalPreviewSurface(holder);
            }
        });
    }

    /**
     * 设置预览Surface
     * @param texture   SurfaceTexture
     */
    public void setPreviewSurface(final SurfaceTexture texture) {
        checkHandleAvailable();
        mHandler.post(new Runnable() {
            @Override
            public void run() {
                internalPreviewSurface(texture);
            }
        });
    }

    /**
     * 设置预览回调
     * @param callback  回调
     * @param buffer    缓冲
     */
    public void setPreviewCallbackWithBuffer(final Camera.PreviewCallback callback,
                                             final byte[] buffer) {
        checkHandleAvailable();
        mHandler.post(new Runnable() {
            @Override
            public void run() {
                internalPreviewCallbackWithBuffer(callback, buffer);
            }
        });
    }

    /**
     * 开始预览
     */
    public void startPreview() {
        checkHandleAvailable();
        mHandler.post(new Runnable() {
            @Override
            public void run() {
                internalStartPreview();
            }
        });
    }

    /**
     * 停止预览
     */
    public void stopPreview() {
        checkHandleAvailable();
        mHandler.post(new Runnable() {
            @Override
            public void run() {
                internalStopPreview();
            }
        });
    }

    /**
     * 切换相机
     * @param cameraId 相机Id
     */
    public void switchCamera(final int cameraId) {
        checkHandleAvailable();
        mHandler.post(new Runnable() {
            @Override
            public void run() {
                internalSwitchCamera(cameraId);
            }
        });
    }

    /**
     * 切换相机
     * @param cameraId  相机Id
     * @param expectFps 期望帧率
     */
    public void switchCamera(final int cameraId, final int expectFps) {
        checkHandleAvailable();
        mHandler.post(new Runnable() {
            @Override
            public void run() {
                internalSwitchCamera(cameraId, expectFps);
            }
        });
    }

    /**
     * 切换相机
     * @param cameraId      相机Id
     * @param expectFps     期望帧率
     * @param expectWidth   期望宽度
     * @param expectHeight  期望高度
     */
    public void switchCamera(final int cameraId, final int expectFps, final int expectWidth, final int expectHeight) {
        checkHandleAvailable();
        mHandler.post(new Runnable() {
            @Override
            public void run() {
                internalSwitchCamera(cameraId, expectFps, expectWidth, expectHeight);
            }
        });
    }

    /**
     * 切换相机
     * @param cameraId
     * @param holder
     * @param callback
     * @param buffer
     */
    public void switchCameraAndPreview(final int cameraId, final SurfaceHolder holder,
                             final Camera.PreviewCallback callback, final byte[] buffer) {
        checkHandleAvailable();
        mHandler.post(new Runnable() {
            @Override
            public void run() {
                internalSwitchCameraAndPreview(cameraId, holder, callback, buffer);
            }
        });
    }

    /**
     * 切换相机
     * @param cameraId
     * @param texture
     * @param callback
     * @param buffer
     */
    public void switchCameraAndPreview(final int cameraId, final SurfaceTexture texture,
                             final Camera.PreviewCallback callback, final byte[] buffer) {
        checkHandleAvailable();
        mHandler.post(new Runnable() {
            @Override
            public void run() {
                internalSwitchCameraAndPreview(cameraId, texture, callback, buffer);
            }
        });
    }



    /**
     * 释放相机
     */
    synchronized public void releaseCamera() {
        checkHandleAvailable();
        mHandler.post(new Runnable() {
            @Override
            public void run() {
                internalReleaseCamera();
                notifyCameraReleased();
            }
        });
        waitUntilReady();
    }

    /**
     * 通知销毁成功
     */
    synchronized private void notifyCameraReleased() {
        notify();
    }

    /**
     * 拍照
     * @param shutterCallback
     * @param rawCallback
     * @param pictureCallback
     */
    public void takePicture(final Camera.ShutterCallback shutterCallback,
                            final Camera.PictureCallback rawCallback,
                            final Camera.PictureCallback pictureCallback) {
        checkHandleAvailable();
        mHandler.post(new Runnable() {
            @Override
            public void run() {
                internalTakePicture(shutterCallback, rawCallback, pictureCallback);
            }
        });
    }

    /**
     * 计算预览角度
     * @param activity
     */
    synchronized public void calculatePreviewOrientation(final Activity activity) {
        checkHandleAvailable();
        mHandler.post(new Runnable() {
            @Override
            public void run() {
                internalCalculatePreviewOrientation(activity);
                notifyPreviewOrientationCalculated();
            }
        });

        try {
            wait();
        } catch (InterruptedException e) {
            Log.w(TAG, "wait was interrupted");
        }
    }

    /**
     * 通知计算预览角度完成
     */
    synchronized private void notifyPreviewOrientationCalculated() {
        notify();
    }



    // ------------------------------- 内部方法 -----------------------------
    /**
     * 打开相机
     */
    private void internalOpenCamera() {
        internalOpenCamera(CameraManager.getInstance().DESIRED_PREVIEW_FPS);
    }

    /**
     * 打开相机
     * @param expectFps 期望的帧率
     */
    private void internalOpenCamera(int expectFps) {
        CameraManager.getInstance().openCamera(expectFps);
    }

    /**
     * 打开相机
     * @param cameraId 相机Id
     * @param expectFps 期望帧率
     */
    private void internalOpenCamera(int cameraId, int expectFps) {
        CameraManager.getInstance().openCamera(cameraId, expectFps);
    }

    /**
     * 打开相机
     * @param cameraId      相机帧率
     * @param expectFps     期望帧率
     * @param expectWidth   期望宽度
     * @param expectHeight  期望高度
     */
    private void internalOpenCamera(int cameraId, int expectFps,
                                    int expectWidth, int expectHeight) {
        CameraManager.getInstance().openCamera(cameraId, expectFps, expectWidth, expectHeight);
    }

    /**
     * 重新打开相机
     */
    private void internalReopenCamera() {
        CameraManager.getInstance().reopenCamera();
    }

    /**
     * 重新打开相机
     * @param expectFps 期望帧率
     */
    private void internalReopenCamera(int expectFps) {
        CameraManager.getInstance().reopenCamera();
    }

    /**
     * 重新打开相机
     * @param expectFps 期望帧率
     * @param expectWidth   期望宽度
     * @param expectHeight  期望高度
     */
    public void internalReopenCamera(int expectFps, int expectWidth, int expectHeight) {
        CameraManager.getInstance().reopenCamera(expectFps, expectWidth, expectHeight);
    }

    /**
     * 预览Surface
     * @param holder
     */
    private void internalPreviewSurface(SurfaceHolder holder) {
        CameraManager.getInstance().setPreviewSurface(holder);
    }

    /**
     * 预览Surface
     * @param texture
     */
    private void internalPreviewSurface(SurfaceTexture texture) {
        CameraManager.getInstance().setPreviewSurface(texture);
    }

    /**
     * 设置预览回调
     * @param callback  预览回调
     * @param buffer    预览回调
     */
    private void internalPreviewCallbackWithBuffer(Camera.PreviewCallback callback, byte[] buffer) {
        CameraManager.getInstance().setPreviewCallbackWithBuffer(callback, buffer);
    }

    /**
     * 开始预览
     */
    private void internalStartPreview() {
        CameraManager.getInstance().startPreview();
    }

    /**
     * 停止预览
     */
    private void internalStopPreview() {
        CameraManager.getInstance().stopPreview();
    }

    /**
     * 切换相机
     * @param cameraId  相机的Id
     */
    private void internalSwitchCamera(int cameraId) {
        CameraManager.getInstance().switchCamera(cameraId);
    }

    /**
     * 切换相机
     * @param cameraId 相机Id
     * @param expectFps 期望帧率
     */
    private void internalSwitchCamera(int cameraId, int expectFps) {
        CameraManager.getInstance().switchCamera(cameraId, expectFps);
    }

    /**
     * 切换相机
     * @param cameraId      相机Id
     * @param expectFps     期望帧率
     * @param expectWidth   期望宽度
     * @param expectHeight  期望高度
     */
    private void internalSwitchCamera(int cameraId, int expectFps,
                                      int expectWidth, int expectHeight) {
        CameraManager.getInstance().switchCamera(cameraId, expectFps, expectWidth, expectHeight);
    }

    /**
     * 切换相机并预览
     * @param cameraId  相机的Id
     * @param holder    绑定的SurfaceHolder
     * @param callback  预览回调
     * @param buffer    缓冲buffer
     */
    private void internalSwitchCameraAndPreview(int cameraId, SurfaceHolder holder,
                                                Camera.PreviewCallback callback, byte[] buffer) {
        CameraManager.getInstance().switchCameraAndPreview(cameraId, holder, callback, buffer);
    }

    /**
     * 切换相机并预览
     * @param cameraId  相机Id
     * @param texture   绑定的SurfaceTexture
     * @param callback  预览回调
     * @param buffer    缓冲buffer
     */
    private void internalSwitchCameraAndPreview(int cameraId, SurfaceTexture texture,
                                      Camera.PreviewCallback callback, byte[] buffer) {
        CameraManager.getInstance().switchCameraAndPreview(cameraId, texture, callback, buffer);
    }

    /**
     * 释放相机
     */
    private void internalReleaseCamera() {
        CameraManager.getInstance().releaseCamera();
    }

    /**
     * 拍照
     * @param shutterCallback
     * @param rawCallback
     * @param pictureCallback
     */
    private void internalTakePicture(Camera.ShutterCallback shutterCallback,
                                     Camera.PictureCallback rawCallback,
                                     Camera.PictureCallback pictureCallback) {
        CameraManager.getInstance().takePicture(shutterCallback, rawCallback, pictureCallback);
    }

    /**
     * 计算预览角度
     * @param activity
     */
    private void internalCalculatePreviewOrientation(Activity activity) {
        CameraManager.getInstance().calculateCameraPreviewOrientation(activity);
    }



    // ------------------------------------- setter and getter -------------------------------------

    /**
     * 获取回调
     * @return
     */
    public Handler getHandler() {
        return mHandler;
    }

    /**
     * 获取相机Id
     * @return
     */
    public int getCameraId() {
        return CameraManager.getInstance().getCameraID();
    }

    /**
     * 获取照片的大小
     * @return
     */
    public Size getPictureSize() {
        return CameraManager.getInstance().getPictureSize();
    }

    /**
     * 获取当前预览的大小
     * @return
     */
    public Size getPreviewSize() {
        return CameraManager.getInstance().getPreviewSize();
    }

    /**
     * 获取相机信息
     * @return
     */
    public CameraInfo getCameraInfo() {
        return CameraManager.getInstance().getCameraInfo();
    }

    /**
     * 获取当前预览角度
     * @return
     */
    public int getPreviewOrientation() {
        return CameraManager.getInstance().getPreviewOrientation();
    }

    /**
     * 获取帧率(FPS 千秒值)
     * @return
     */
    public int getCameraPreviewThousandFps() {
        return CameraManager.getInstance().getCameraPreviewThousandFps();
    }

    /**
     * 获取当前的长宽比
     * @return
     */
    public float getCurrentRatio() {
        return CameraManager.getInstance().getCurrentRatio();
    }

}
```
然后，我们来看看渲染管理器RenderManager：
```
public final class RenderManager {

    private static final String TAG = "RenderManager";

    private static RenderManager mInstance;

    private static Object mSyncObject = new Object();

    // 是否允许绘制人脸关键点
    private boolean enableDrawPoints = false;

    // 相机输入流滤镜
    private CameraFilter mCameraFilter;
    // 实时滤镜组
    private BaseImageFilterGroup mRealTimeFilter;
    // 关键点绘制（调试用）
    private FacePointsDrawer mFacePointsDrawer;
    // 顶点和UV坐标缓冲
    private FloatBuffer mVertexBuffer;
    private FloatBuffer mTextureBuffer;

    // 输入流大小
    private int mTextureWidth;
    private int mTextureHeight;
    // 显示大小
    private int mDisplayWidth;
    private int mDisplayHeight;

    // 显示的缩放裁剪类型
    private ScaleType mScaleType = ScaleType.CENTER_CROP;



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
        // 释放之前的滤镜和缓冲
        releaseBuffers();
        releaseFilters();
        // 初始化滤镜和缓冲
        initBuffers();
        initFilters();
    }

    /**
     * 初始化滤镜
     */
    private void initFilters() {
        mCameraFilter = new CameraFilter();
        mFacePointsDrawer = new FacePointsDrawer();
//        mRealTimeFilter = FilterManager.getFilterGroup();
    }

    /**
     * 初始化缓冲
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
        if (mFacePointsDrawer != null) {
            mFacePointsDrawer.release();
            mFacePointsDrawer = null;
        }
        if (mRealTimeFilter != null) {
            mRealTimeFilter.release();
            mRealTimeFilter = null;
        }
    }

    /**
     * 释放缓冲资源
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
        if (mRealTimeFilter != null) {
            mRealTimeFilter.onDisplayChanged(width, height);
        }
        // 调整视图大小
        adjustViewSize();
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
     * 绘制渲染
     * @param textureId
     */
    public void drawFrame(int textureId) {
        if (mRealTimeFilter == null) {
            mCameraFilter.drawFrame(textureId);
        } else {
            int id = mCameraFilter.drawFrameBuffer(textureId);
            mRealTimeFilter.drawFrame(id, mVertexBuffer, mTextureBuffer);
        }
        // 是否绘制点
        if (enableDrawPoints && mFacePointsDrawer != null) {
            mFacePointsDrawer.drawPoints();
        }
    }

    /**
     * 调整由于surface的大小与SurfaceView显示大小不一致带来的显示问题
     */
    private void adjustViewSize() {
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
        float ratioWidth = (float) imageWidth / (float) mTextureWidth;
        float ratioHeight = (float) imageHeight / (float) mTextureHeight;
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
     * 滤镜或视图发生变化时调用
     */
    private void cameraFilterChanged() {
        if (mDisplayWidth != mDisplayHeight) {
            mCameraFilter.onDisplayChanged(mDisplayWidth, mDisplayHeight);
        }
        mCameraFilter.initFramebuffer(mTextureWidth, mTextureHeight);
    }

    //------------------------------ setter and getter ---------------------------------

    /**
     * 设置顶点坐标缓冲
     * @param buffer
     */
    public void setVertexBuffer(FloatBuffer buffer) {
        mVertexBuffer = buffer;
    }

    /**
     *  设置UV坐标缓冲
     * @param buffer
     */
    public void setTextureBuffer(FloatBuffer buffer) {
        mTextureBuffer = buffer;
    }

    /**
     * 设置SurfaceTexture 的Transform矩阵
     * @param matrix
     */
    public void setTransformMatrix(float[] matrix) {
        mCameraFilter.setTextureTransformMatirx(matrix);
    }


}
```
其中，CameraFilter 的实现可以参考本人的相机项目[CainCamera](https://github.com/CainKernel/CainCamera)。这里并没有开启实时磨皮渲染等操作，仅仅是绘制渲染了Camera的相机流。
我们再来看看渲染线程如何实现：
```
public class RenderTestThread extends HandlerThread implements SurfaceTexture.OnFrameAvailableListener {
    private static final String TAG = "RenderTestThread";

    private final Handler mHandler;

    private EglCore mEglCore;
    private WindowSurface mDisplaySurface;
    private SurfaceTexture mSurfaceTexture;
    private final float[] mMatrix = new float[16];

    private int mCameraTexture;

    private int mImageWidth;
    private int mImageHeight;
    // 预览的角度
    private int mOrientation;

    public RenderTestThread() {
        super(TAG);
        start();
        mHandler = new Handler(getLooper());
    }

    public RenderTestThread(String name) {
        super(name);
        start();
        mHandler = new Handler(getLooper());
    }

    /**
     * 销毁线程
     */
    public void destoryThread() {
        internalRelease();
        mHandler.removeCallbacksAndMessages(null);
        quitSafely();
    }

    /**
     * 检查handler是否可用
     */
    private void checkHandleAvailable() {
        if (mHandler == null) {
            throw new NullPointerException("Handler is not available!");
        }
    }

    /**
     * 等待
     */
    private void waitUntilReady() {
        try {
            wait();
        } catch (InterruptedException e) {
            Log.w(TAG, "wait was interrupted");
        }
    }

    synchronized public void surfaceCreated(final SurfaceHolder holder) {
        checkHandleAvailable();
        mHandler.post(new Runnable() {
            @Override
            public void run() {
                internalSurfaceCreated(holder);
                notifySurfaceProcessed();
            }
        });
        waitUntilReady();
    }

    synchronized public void surfaceChanged(final int width, final int height) {
        checkHandleAvailable();
        mHandler.post(new Runnable() {
            @Override
            public void run() {
                internalSurfaceChanged(width, height);
                notifySurfaceProcessed();
            }
        });
        waitUntilReady();
    }

    synchronized public void surfaceDestory() {
        checkHandleAvailable();
        mHandler.post(new Runnable() {
            @Override
            public void run() {
                internalSurfaceDestory();
                notifySurfaceProcessed();
            }
        });
        waitUntilReady();
    }

    /**
     * Surface变化处理完成通知
     */
    synchronized private void notifySurfaceProcessed() {
        notify();
    }

    /**
     * 设置图片大小
     * @param width         宽度
     * @param height        高度
     * @param orientation   角度
     */
    public void setImageSize(int width, int height, int orientation) {
        mImageWidth = width;
        mImageHeight = height;
        mOrientation = orientation;
        checkHandleAvailable();
        mHandler.post(new Runnable() {
            @Override
            public void run() {
                internalupdateImageSize();
            }
        });
    }

    /**
     * 更新帧
     */
    public void updateFrame() {
        checkHandleAvailable();
        mHandler.post(new Runnable() {
            @Override
            public void run() {
                internalRendering();
            }
        });
    }

    @Override
    public void onFrameAvailable(SurfaceTexture surfaceTexture) {
//        updateFrame();
    }

    // ------------------------------ 内部方法 -------------------------------------

    private void internalSurfaceCreated(SurfaceHolder holder) {
        mEglCore = new EglCore(null, EglCore.FLAG_RECORDABLE | EglCore.FLAG_TRY_GLES3);
        mDisplaySurface = new WindowSurface(mEglCore, holder.getSurface(), false);
        mDisplaySurface.makeCurrent();
        mCameraTexture = GlUtil.createTextureOES();
        mSurfaceTexture = new SurfaceTexture(mCameraTexture);
        RenderManager.getInstance().init();
    }

    private void internalSurfaceChanged(int width, int height) {
        RenderManager.getInstance().onDisplaySizeChanged(width, height);
    }

    private void internalSurfaceDestory() {
        internalRelease();
    }

    /**
     * 释放所有资源
     */
    private void internalRelease() {
        if (mEglCore != null) {
            mEglCore.release();
        }
        if (mDisplaySurface != null) {
            mDisplaySurface.release();
        }
        if (mSurfaceTexture != null) {
            mSurfaceTexture.release();
        }
        RenderManager.getInstance().release();
    }

    /**
     * 更新图片大小(相机流大小)
     */
    private void internalupdateImageSize() {
        calculateImageSize();
        RenderManager.getInstance().onInputSizeChanged(mImageWidth, mImageHeight);
    }

    /**
     * 计算image的宽高
     */
    private void calculateImageSize() {
        Log.d("calculateImageSize", "orientation = " + mOrientation);
        if (mOrientation == 90 || mOrientation == 270) {
            int temp = mImageWidth;
            mImageWidth = mImageHeight;
            mImageHeight = temp;
        }
    }
    /**
     * 渲染
     */
    private void internalRendering() {
        mDisplaySurface.makeCurrent();
        if (mSurfaceTexture != null) {
            mSurfaceTexture.updateTexImage();
        }
        renderFrame();
        mDisplaySurface.swapBuffers();
    }

    /**
     * 渲染一帧
     */
    private void renderFrame() {
        mSurfaceTexture.getTransformMatrix(mMatrix);
        RenderManager.getInstance().setTransformMatrix(mMatrix);
        RenderManager.getInstance().drawFrame(mCameraTexture);
    }

    // ------------------------------- setter and getter ---------------------------

    /**
     * 获取SurfaceTexture
     * @return
     */
    public SurfaceTexture getSurafceTexture() {
        return mSurfaceTexture;
    }

}
```
然后在Activity里面控制打开渲染操作：
```
public class CameraTestActivity extends AppCompatActivity implements SurfaceHolder.Callback,
        Camera.PreviewCallback {

    private static final int REQUEST_CAMERA = 0x01;
    private static final int REQUEST_STORAGE_READ = 0x02;
    private static final int REQUEST_STORAGE_WRITE = 0x03;
    private static final int REQUEST_RECORD = 0x04;
    private static final int REQUEST_LOCATION = 0x05;

    private AspectFrameLayout mAspectLayout;

    private SurfaceView mSurfaceView;
    // 相机控制线程
    private CameraHandlerThread mCameraThread;
    // 渲染控制线程
    private RenderTestThread mRenderThread;

    private byte[] mPreviewBuffer;

    private boolean mCameraEnable = false;
    private boolean mStorageWriteEnable = false;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_camera_test);

        mCameraEnable = PermissionUtils.permissionChecking(this,
                Manifest.permission.CAMERA);
        mStorageWriteEnable = PermissionUtils.permissionChecking(this,
                Manifest.permission.WRITE_EXTERNAL_STORAGE);
        if (mCameraEnable && mStorageWriteEnable) {
            initView();
        } else {
            ActivityCompat.requestPermissions(this, new String[]{ Manifest.permission.CAMERA,
                    Manifest.permission.WRITE_EXTERNAL_STORAGE }, REQUEST_CAMERA);
        }

    }

    /**
     * 初始化视图
     */
    private void initView() {
        mAspectLayout = (AspectFrameLayout) findViewById(R.id.layout_aspect);
        mAspectLayout.setAspectRatio(CameraManager.getInstance().getCurrentRatio());

        mSurfaceView = new SurfaceView(this);
        mSurfaceView.getHolder().addCallback(this);

        mAspectLayout.addView(mSurfaceView);
        mAspectLayout.requestLayout();
    }

    @Override
    protected void onResume() {
        super.onResume();
        mCameraThread = new CameraHandlerThread();
        mCameraThread.calculatePreviewOrientation(this);
        mRenderThread = new RenderTestThread();
    }

    @Override
    public void onRequestPermissionsResult(int requestCode,
                                           @NonNull String[] permissions,
                                           @NonNull int[] grantResults) {
        super.onRequestPermissionsResult(requestCode, permissions, grantResults);
        switch (requestCode) {
            // 相机权限
            case REQUEST_CAMERA:
                if (grantResults.length > 0
                        && grantResults[0] == PackageManager.PERMISSION_GRANTED) {
                    mCameraEnable = true;
                    initView();
                }
                break;

            // 存储权限
            case REQUEST_STORAGE_WRITE:
                if (grantResults.length > 0
                        && grantResults[0] == PackageManager.PERMISSION_GRANTED) {
                    mStorageWriteEnable = true;
                }
                break;
        }
    }


    @Override
    public void surfaceCreated(SurfaceHolder holder) {

        // CameraThread是一个HandlerThread，调用Camera.open相机，onPreviewFrame回调所绑定的线程是CameraThread，
        // 如果使用new Thread的方式调用Camera.open相机，onPreviewFrame回调将绑定到到MainLooper，也就是回调到主线程
        // 这里使用单独的HandlerThread控制Camera逻辑
        mCameraThread.openCamera();

        Size previewSize = mCameraThread.getPreviewSize();
        int size = previewSize.getWidth() * previewSize.getHeight() * 3 / 2;
        mPreviewBuffer = new byte[size];

        mRenderThread.surfaceCreated(holder);
        mRenderThread.setImageSize(previewSize.getWidth(), previewSize.getHeight(),
                mCameraThread.getPreviewOrientation());

        // 设置预览SurfaceTexture
        mCameraThread.setPreviewSurface(mRenderThread.getSurafceTexture());
        // 设置回调
        mCameraThread.setPreviewCallbackWithBuffer(CameraTestActivity.this, mPreviewBuffer);
    }

    @Override
    public void surfaceChanged(SurfaceHolder holder, int format, int width, int height) {
        mCameraThread.startPreview();
        mRenderThread.surfaceChanged(width, height);
    }

    @Override
    public void surfaceDestroyed(SurfaceHolder holder) {
        mCameraThread.destoryThread();
        mRenderThread.destoryThread();
    }

    /**
     测试情况：
     高通：
     VIVO X9i，高通625
     红米Note 4X，高通625
     Nexus 5X，高通808
     onPreviewFrame回调时间： 单一 HandlerThread， 30~40ms； 双HandlerThread，30~40ms
     preview size： 1280 x 720
     单一HandlerThread的情况，请参考CameraActivity里面的情况

     联发科:
     红米Note4， 联发科 Helio X20
     魅蓝Note 2，联发科 MTK6573
     乐视 X620，联发科X20
     onPreviewFrame回调时间：单一HandlerThread， 30 ~ 40ms；双HandlerThread，60~70ms
     preview size： 1280 x 720

     操作：
     Camera数据流渲染到SurfaceTexture显示到SurfaceView上，设置setPreviewCallbackWithBuffer，查看onPreviewFrame的帧率
     */
    long time = 0;
    @Override
    public void onPreviewFrame(byte[] data, Camera camera) {
        Log.d("onPreviewFrame", "update time = " + (System.currentTimeMillis() - time));
        time = System.currentTimeMillis();
        mRenderThread.updateFrame();

        if (mPreviewBuffer != null) {
            camera.addCallbackBuffer(mPreviewBuffer);
        }
    }

}
```
至此，把以上代码运行在高通的CPU上和MTK的CPU上，就会发现，MTK的CPU的onPreviewFrame回调的时间大约在 60~70ms 左右，而高通的CPU 的 onPreviewFrame回调的时间大约在30 ~ 40ms左右。MTK的CPU 预览画面出现了明显的不连贯感。
如果我们把相机控制和渲染的操作均放在同一个HandlerThread，你会发现，MTK CPU的onPreviewFrame 回调的时间跟高通的表现一致，在MTK的X20等旗舰级的CPU上，做完磨皮算法，onPreviewFrame的回调时间也能保持在30~40ms内，详细的写法可以参考本人的相机项目，并且在接入了Face++的人脸检测SDK后，依旧能保证onPreviewFrame回调时间比市面上几乎所有商业相机的回调时间要短。在做完磨皮、颜色滤镜的前提下，在红米Note 2 (MTK X10）上渲染帧率稳定在19.5帧左右，而市面上几乎所有相机均存在不同程度的掉帧现象，均不能稳定保持在19帧以上。

接下来我想谈一谈人脸检测对onPreviewFrame回调帧率的影响。市面上的人脸检测SDK 在做人脸关键点检测的时间有长有短，那么什么样的方案对onPreviewFrame的回调帧率不怎么影响呢？我认为，理想状态下的方案是应该是，人脸检测放在另外一个单独的HandlerThread里面，每次做完检测，回调到RenderThread通知渲染线程绘制新的一帧， 这样人脸检测和渲染操作并行处理，onPreviewFrame将图像数据data 通过handler 传递给人脸检测的HandlerThread，然后调用camera的addCallbackBuffer，回传缓冲。这样当我们渲染完成，可以立马进入下一次的人脸检测，人脸检测完成回调里面通知渲染刷新，因此，onPreviewFrame的帧率可以一直得到保证，而人脸检测和渲染操作也能得到并行化处理，此时的帧率是 1000 / (Max(人脸检测, 渲染操作)(ms))。理论上，如果人脸检测的时间小于30ms，可以实现实时预览帧率接近30帧。可以参考本人的相机项目：[CainCamera](https://github.com/CainKernel/CainCamera)，face++ 的人脸SDK 1280x720的分辨率下，在高通625上，单人脸106个关键点，大约在20ms左右，也就是说，在下一帧来临之前就能得到人脸关键点，然后回调做磨皮处理、绘制滤镜等操作。我这样的方案，从人脸检测到磨皮渲染处理、渲染滤镜等操作完成，在高通808上的处理时间和高通625上的表现相当，MTK X10上的时间大约为50ms左右，下图是在高通808上的从人脸检测到渲染一帧完成所需要的时间：
![绘制一帧的时间](http://upload-images.jianshu.io/upload_images/2103804-d0a98d1dd44e15d2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
从上面可以看到，人脸关键点检测的时间在小于30ms的时候，并行化的方案是可以做到接近30帧的预览效率的。在这样的前提下，做瘦脸、贴纸等渲染处理，实时渲染预览的帧率在大部分手机上都能保证在20帧以上，超过20帧对相机预览画面来说，基本没有不连贯感了。






