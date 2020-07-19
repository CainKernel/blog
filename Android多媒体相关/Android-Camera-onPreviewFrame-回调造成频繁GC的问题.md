在开发相机的过程中，本人遇到一个奇怪的Bug，在这里记录完整的调试过程。
事情是这样的，公司的相机项目使用了Camera的onPreviewFrame回调取出预览数据用于人脸检测和人脸朝向的计算，用于准确绘制贴纸的位置。在调试过程中发现，内存抖动非常频繁，如下图所示：

![内存抖动和.png](http://upload-images.jianshu.io/upload_images/2103804-811858f560346132.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
如图所示，内存呈锯齿状，查看Log可以看到频繁地GC，如下图所示：

![频繁GC.png](http://upload-images.jianshu.io/upload_images/2103804-b317da3de796ddc9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
这样频繁GC 很明显是不正常的，很明显有可能发生了内存泄漏或者某些操作导致数据不断地创建销毁造成频繁GC。我们用Allocation Tracking记录，结果如下：
![AllocationTracking.png](http://upload-images.jianshu.io/upload_images/2103804-561d8a2b51c825d1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
这里可以看到这里不断创建销毁的数据类型是byte[]数组，而且是跟Binder相关。那么这几个Binder是用来做啥的？
我们打开Android Device Monitor，选中应用，点击Start Method Profiling，就是下面这个图标：

![Start Method Profiling.png](http://upload-images.jianshu.io/upload_images/2103804-cf187d39dac425e2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
录制停止后，我们可以看到各个方法以及线程号等情况：

![Method Profile情况.png](http://upload-images.jianshu.io/upload_images/2103804-ae5125900ec7b39a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
根据前面的Allocation Tracking所得到的线程号，用鼠标点击上图中黑色的横线，我们可以在下面一栏看到对应的执行方法，结果如下：
![线程操作情况.png](http://upload-images.jianshu.io/upload_images/2103804-ecfd983df475eacd.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![线程操作情况1.png](http://upload-images.jianshu.io/upload_images/2103804-f540db222458eba2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
我们可以看到这个是android.hardware.camera在native层所做的操作，这几个Thread 和Binder主要操作是发送消息，更新SurfaceTexture等。这里也没法看出什么情况，但可以知道这是Camera预览数据出现了问题。我们再回到Android Studio中，搜索一下Log，使用关键字Alloc，发现情况如下（备注，有些手机是不会出现这些Log的，我的Nexus5X没有，但红米Note2出现了）：
![不断创建callbackbuffer.png](http://upload-images.jianshu.io/upload_images/2103804-86fb42ee0f881c1d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
也就是说，问题出在了addCallbackBuffer方法上了。但我查看了项目，发现已经再startPreview 之前添加了回调，而且在onPreviewFrame里面也添加了addCallbackBuffer了的。那问题产生的地方在哪里？
我用Google 搜索了相关关键字，终于搜索到了一个讨论：
Camera API: Excessive GC caused by preview callbacks 
地址： https://groups.google.com/forum/#!topic/android-platform/wjMDSdJJ1xU
有人问预览回调导致频繁GC 的问题，这个跟我遇到的情况非常相似。我看了下里面讨论的过程，突然想到，是否是addCallbackBuffer使用得不对？
后来试着修改一下，发现解决方法，addCallbackBuffer 的使用情况应该是这样的：
1、在startPreview之前，我们需要设置addCallbackBuffer 和 setPreviewCallbackWithBuffer，这两个方法必须同时使用才有效果，不能使用addCallbackBuffer 和 setPreviewCallback配合使用，这样同样没有效果。
```
mCamera.addCallbackBuffer(mPreviewBuffer);
mCamera.setPreviewCallbackWithBuffer(this);
```
2、在onPrevieFrame中，处理完成后，不管有没有数据，都需要在最后再次调用addCallbackBuffer，并且只有在最后调用才有效，否则在addCallbackBuffer之后还持有byte[] 数组，依旧有可能导致抖动，尤其是在其他线程持有byte[]数组的情况下。公司项目原来始在onPreviewFrame中添加了addCallbackBuffer，但是在addCallbackBuffer之后，还做了其他操作，byte[]数据其他线程持有，做完处理，线程就被销毁了，data跟着被销毁，导致Camera 需要重新创建Buffer。
```
    @Override
    public void onPreviewFrame(byte[] data, Camera camera) {
        // 处理预览数据
        .......
        // 预览处理完成后，不管有没有数据，都需要添加回调
        camera.addCallbackBuffer(mPreviewBuffer);
    }
```
修改后，编译运行，查看内存状况如下：
![内存运行状况.png](http://upload-images.jianshu.io/upload_images/2103804-91c0032c64e29f77.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
我们可以看到，相比之前的呈锯齿状的内存抖动，现在的内存相对平稳了许多（请忽略内存不断增大之后GC的情况，这是其他问题导致的，跟onPreviewFrame无关），不再出现锯齿状的抖动了，查看Log也发现，相比之前不断触发GC的情况，现在GC的log几乎不再出现了。
至此，onPreviewFrame回调导致频繁GC 的问题解决了。
