一、使用。关于Camera2的API使用，参考Google官方的例子：
[Camera2Basic](https://github.com/googlesamples/android-Camera2Basic)
[Camera2Raw](https://github.com/googlesamples/android-Camera2Raw)
[Camera2Video](https://github.com/googlesamples/android-Camera2Video)
这是一手资料，配合官方的资料理解Camera2 API的底层原理：
[3A 模式和状态转换](https://source.android.com/devices/camera/camera3_3Amodes)

二、关于Camera2 API 的一些坑。
本人应公司要求，预研Camera2 相关API以及封装。在参考[Camera2Basic](https://github.com/googlesamples/android-Camera2Basic) 编写相机应用时，本人发现了Camera2 API 的关于自动对焦的一个非常严重的BUG。在此记录下来，希望后来者在使用Camera2 API时，慎重选择。
Camera2Basic 中出现问题的代码如下：
```
private CameraCaptureSession.CaptureCallback mCaptureCallback
            = new CameraCaptureSession.CaptureCallback() {

        private void process(CaptureResult result) {
            switch (mState) {
                case STATE_PREVIEW: {
                    // We have nothing to do when the camera preview is working normally.
                    break;
                }
                case STATE_WAITING_LOCK: {
                    Integer afState = result.get(CaptureResult.CONTROL_AF_STATE);
                    if (afState == null) {
                        captureStillPicture();
                    } else if (CaptureResult.CONTROL_AF_STATE_FOCUSED_LOCKED == afState ||
                            CaptureResult.CONTROL_AF_STATE_NOT_FOCUSED_LOCKED == afState) {
                        // CONTROL_AE_STATE can be null on some devices
                        Integer aeState = result.get(CaptureResult.CONTROL_AE_STATE);
                        if (aeState == null ||
                                aeState == CaptureResult.CONTROL_AE_STATE_CONVERGED) {
                            mState = STATE_PICTURE_TAKEN;
                            captureStillPicture();
                        } else {
                            runPrecaptureSequence();
                        }
                    }
                    break;
                }
                case STATE_WAITING_PRECAPTURE: {
                    // CONTROL_AE_STATE can be null on some devices
                    Integer aeState = result.get(CaptureResult.CONTROL_AE_STATE);
                    if (aeState == null ||
                            aeState == CaptureResult.CONTROL_AE_STATE_PRECAPTURE ||
                            aeState == CaptureRequest.CONTROL_AE_STATE_FLASH_REQUIRED) {
                        mState = STATE_WAITING_NON_PRECAPTURE;
                    }
                    break;
                }
                case STATE_WAITING_NON_PRECAPTURE: {
                    // CONTROL_AE_STATE can be null on some devices
                    Integer aeState = result.get(CaptureResult.CONTROL_AE_STATE);
                    if (aeState == null || aeState != CaptureResult.CONTROL_AE_STATE_PRECAPTURE) {
                        mState = STATE_PICTURE_TAKEN;
                        captureStillPicture();
                    }
                    break;
                }
            }
        }

        @Override
        public void onCaptureProgressed(@NonNull CameraCaptureSession session,
                                        @NonNull CaptureRequest request,
                                        @NonNull CaptureResult partialResult) {
            process(partialResult);
        }

        @Override
        public void onCaptureCompleted(@NonNull CameraCaptureSession session,
                                       @NonNull CaptureRequest request,
                                       @NonNull TotalCaptureResult result) {
            process(result);
        }

    };
```
出现问题的代码如下：
```
case STATE_WAITING_LOCK: {
                    Integer afState = result.get(CaptureResult.CONTROL_AF_STATE);
                    if (afState == null) {
                        captureStillPicture();
                    } else if (CaptureResult.CONTROL_AF_STATE_FOCUSED_LOCKED == afState ||
                            CaptureResult.CONTROL_AF_STATE_NOT_FOCUSED_LOCKED == afState) {
                        // CONTROL_AE_STATE can be null on some devices
                        Integer aeState = result.get(CaptureResult.CONTROL_AE_STATE);
                        if (aeState == null ||
                                aeState == CaptureResult.CONTROL_AE_STATE_CONVERGED) {
                            mState = STATE_PICTURE_TAKEN;
                            captureStillPicture();
                        } else {
                            runPrecaptureSequence();
                        }
                    }
                    break;
                }
```
调用拍照方法后，会进入STATE_WAITING_LOCK状态，此时获取Integer afState = result.get(CaptureResult.CONTROL_AF_STATE);的对焦状态afState 在某些机器上面，连续拍了几张图片之后，afState 会一直处于CaptureResult.CONTROL_AF_STATE_PASSIVE_SCAN状态，表示一个持续聚焦的算法正在做扫描。镜头正在移动中。然而实际上你并没有移动镜头。这里会导致后续的对焦完成ImageReader取出对焦完成的图像数据无法进行。也就是无法再拍照了。这是我测试得到的log:
![对焦状态](https://upload-images.jianshu.io/upload_images/2103804-18dd8f12799be99a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

afState 的状态为1，即CONTROL_AF_STATE_PASSIVE_SCAN。无法再继续走到后续的CONTROL_AF_STATE_FOCUSED_LOCKED 和 CONTROL_AF_STATE_NOT_FOCUSED_LOCKED 状态，导致无法取出图像数据，进而完成拍照功能。

测试设备：红米5 Plus

而且这个状态出错的情况一旦出现，就只能关掉相机重新打开才能恢复正常。由于手上的设备有限，无法做更多的测试。但至少这个情况在MIUI系统上非常大概率出现，基于MIUI国内的市场份额，对于这种情况，只有两种解决方案，要么放弃Camera2在拍照时的自动对焦，要么放弃使用Camera2 API。暂时没有找到满意的解决方法。

关于放弃拍照时自动对焦方案是：
```
// 等待对焦被锁定
                case STATE_WAITING_LOCK: {
                    Integer afState = result.get(CaptureResult.CONTROL_AF_STATE);
                    if (afState == null) {
                        Log.d(TAG, "STATE_WAITING_LOCK: mState = STATE_WAITING_LOCK;");
                        captureStillPicture();
                    } else if (
                            CaptureResult.CONTROL_AF_STATE_PASSIVE_SCAN == afState ||
                            CaptureResult.CONTROL_AF_STATE_FOCUSED_LOCKED == afState ||
                            CaptureResult.CONTROL_AF_STATE_NOT_FOCUSED_LOCKED == afState) {
                        // CONTROL_AE_STATE can be null on some devices
                        Integer aeState = result.get(CaptureResult.CONTROL_AE_STATE);
                        if (aeState == null || (CaptureResult.CONTROL_AF_STATE_PASSIVE_SCAN != afState
                                && aeState == CaptureResult.CONTROL_AE_STATE_CONVERGED)) {
                            mState = STATE_PICTURE_TAKEN;
                            captureStillPicture();
                        } else {
                            runPrecaptureSequence();
                        }
                    }
                    Log.d(TAG, "process: afState = " + afState);
                    break;
                }
```
加入CONTROL_AF_STATE_PASSIVE_SCAN 状态的判断，对于出现一直出现CONTROL_AF_STATE_PASSIVE_SCAN的情况时，直接走下一层的AE曝光处理runPrecaptureSequence()流程，此时可能会因为无法对焦，画面层次感丢失的情况。
如果有人有更好的解决方案。希望能够分享一下。个人感觉目前Camera2 API的坑相当的多，至少在使用的时候，注意一些深坑。
