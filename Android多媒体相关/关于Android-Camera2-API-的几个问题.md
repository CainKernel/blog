Google 在Android 5.0之后对Camera做了相当大的改动，新增了Camera2 的API，整体的开发流程也与旧版的Camera开发流程相去甚远。Google官方当前是建议各个厂商积极参与新API的适配了。
现在Android8.0预览版都已经出来了，那么现在的Camera2 的API支持的情况如何？我们来讨论一下。
1、支持程度问题
这都快三年了，我们来看看分辨率的支持程度吧。在下面这篇文章中说到，分辨率某些手机对camera2分辨率的支持不全：
https://www.polarxiong.com/archives/Android%E8%AE%BE%E5%A4%87%E5%AF%B9%E6%96%B0Camera2-API%E7%9A%84%E6%94%AF%E6%8C%81%E9%97%AE%E9%A2%98-%E4%BB%A5%E5%8D%8E%E4%B8%BAM2%E4%B8%BA%E4%BE%8B.html
那么我们来看看其他设备吧
红米Note2前置摄像头：
```
width = 1728, height = 1280
width = 1866, height = 1120
width = 1920, height = 1088
width = 1920, height = 1080
width = 1440, height = 1080
width = 1280, height = 720
width = 960, height = 540
width = 800, height = 600
width = 864, height = 480
width = 800, height = 480
width = 720, height = 480
width = 640, height = 480
width = 480, height = 368
width = 480, height = 320
width = 352, height = 288
width = 320, height = 240
width = 176, height = 144
```
我们来看看旧版相机API支持的分辨率：
```
width = 1728, height = 1280
width = 1866, height = 1120
width = 1920, height = 1088
width = 1920, height = 1080
width = 1440, height = 1080
width = 1280, height = 720
width = 960, height = 540
width = 800, height = 600
width = 864, height = 480
width = 800, height = 480
width = 720, height = 480
width = 640, height = 480
width = 480, height = 368
width = 480, height = 320
width = 352, height = 288
width = 320, height = 240
width = 176, height = 144
```
可以看到支持的分辨率是跟旧版本的Camera 支持的分辨率是一致的，这里没有遇到上面的文章中说到的分辨率支持不全的情况，分辨率支持不全的情况很可能只发生在某些厂商的某些设备上。

2、预览帧率的情况。
旧版的预览是通过setPreviewDisplay 将预览帧输出到SurfaceView，并通过接口中的onPreviewFrame获取预览帧。或者可以将SurfaceHolder绑定到EGLSurface 然后通过创建SurfaceTexture，利用opengles渲染完成后回调更新数据。

Camera2 这边则需要调用PreviewRequestBuilder 中的addTarget方法，给Camera2一个Surface。表示输出到Surface，一般来说就是屏幕了。红米Note2 前置摄像头在设置1080 x 1440 分辨率的情况下，不做任何处理，在中等光照下，fps在20 ~ 25 之间浮动。
问题在于，这里使用了ImageReader。问题是在红米Note2上，同一个环境，同样光照的条件下，旧版的Camera 预览输出的帧率能达到24~27范围内波动。明显要比使用ImageReader 的预览帧率要好。

3、关于兼容性问题
在Google官方例子中：
https://github.com/googlesamples/android-Camera2Basic
我们可以看到有57个Issues, 仔细查看，发现已经出现了兼容性问题，在三星S6上出现无响应的情况，并且不仅仅是一个人。另外一个问题是，当我使用红米Note2 测试前置摄像头的时候发现，第一次运行预览数据正常，关掉再打开之后发现预览图像变形的情况，此问题在Nexus 5X上，并没有出现。

4、关于SurfaceView  + opengles 渲染输出问题。
在上面第二小节中，我们可以知道， Camera2是通过addTarget 方法绑定到一个Surface当中的。假如使用SurfaceView + opengles 对预览图像进行渲染的流程根旧版本的其实差不多，整体流程都是创建一个OES 格式的Texture，并用该Texture创建一个新的SurfaceTexture  进行渲染。这里稍微有些不同，那就是旧版本可以通过自适应调整SurfaceView的大小，但当使用新的Camera2 API之后，你需要通过Surface surface = new Surface(mSurfaceTexture); 的方式来创建一个Surface，然后放到addTarget中。在开始预览前，你必须去设置mSurfaceTexture.setDefaultBufferSize方法，否则可能会出现第一次打开，没有画面，点击返回再次打开才有画面，或者画面非常模糊的情况。如果在使用Camera2 API 出现以上情况，请查找SurfaceTexture是否正确设置。
另外最关键的地方，当使用opengles 对SurfaceTexture进行渲染的时候，有可能会出现画面严重拉伸变形的情况。这是由于相机坐标跟opengles 的Texture 的坐标系不一致，哪怕你在opengles里面已经调整好了，由于使用Camera2 输出到Surface，这个Surface是新创建的，这里没法控制Surface的坐标，宽高还是反过来的，因此opengles 处理后画面方向是正常了，但是显示出来的宽度和高度还是反过来的，画面就出现拉伸变形的情况。这个问题如何解决？SurfaceView并没有相关的api做变形处理，处理起来非常麻烦，而且fps可能会受到严重的影响。但TextureView 则可以使用textureView.setTransform(matrix); 处理宽高反过来的情况。因此如果想使用Camera2 API 做滤镜之类的，个人建议使用opengles + TextureView 来实现。不过，个人建议还是使用旧版的API做相关的滤镜处理吧，TextureView 相比GLSurfaceView更省电一些(备注：之前这里写错了，这里做一次勘误)，因为GLSurfaceView是基于SurfaceView实现了对OpenGLES的支持，GLSurfaceView的实现中，GLThread在setRenderer()调用之后就一直处于运行状态，GLThread 中的guardedRun()就是GLSurfaceView 的核心，里面跑了一个while(true)循环，不管是RENDERMODE_WHEN_DIRTY 还是RENDERMODE_CONTINUOUSLY，都会一直处于运行状态，只不过是否区别在于会不会渲染而已，这样的实现相对TextureView来说，自然相对耗电很多，而SurfaceView 和TextureView的比较则没那么容易了，因为你利用SurfaceView 来实现得比较GLSurfaceView要省电，并且，在某些应用场合来说，比TextureView 要更加省电一些，比如频繁切换渲染Texture的场景来说，SurfaceView带有的双缓冲机制自然要比TextureView处理起来相对方便一些，效率也会更加高一些，更加省电一些，但这依赖于SurfaceView绑定OpenGLES 的具体实现方式。如果你使用SurfaceView + HandlerThread的方式来实现，在频繁处理Texture渲染的时候，可以实现得比TextureView更有效率、并且能更好地利用好双缓冲机制、可以解决GLSurfaceView所带来的耗电的缺点。但是实现起来麻烦些，需要自己绑定EGLSurface、EGLContext，实现的话，可以参考谷歌的开源项目Grafika的具体实现。Android7.0开发者版本中有这样一段话：
```
Android 7.0 可同步移动到 SurfaceView 类，此类在某些情况下提供的电池性能优于 TextureView：在渲染视频或 3D 内容时，包含滚动和动画视频位置的应用在使用 SurfaceView 时比 TextureView 耗电更少。
SurfaceView 类可减少屏幕合成对电池的消耗，因为它是在专用硬件中合成，与应用窗口内容分隔开。因此，它产生的中间副本少于 TextureView。
现在，SurfaceView 对象的内容位置和包含的应用内容同步更新。这一变化导致的一个结果是，在画面移动时，SurfaceView 中播放的视频的简单的平移或缩放不再在画面侧面产生黑条。
从 Android 7.0 开始，我们强烈建议您使用 SurfaceView 代替 TextureView，以实现省电。
```
地址如下：
https://developer.android.com/about/versions/nougat/android-7.0.html


新旧版本Camera API的问题
关于Nexus 5X 后置摄像头图像倒置的问题，这其实是一个硬件问题，只是Google官方不承认，说什么相机数据排布有两种方式，Nexus 5X使用了较少人用的方式排布， Android 5.0 新增的Camera2 API 接口可以解决预览倒置的问题。但经过本人的测试发现，哪怕Camera2 的API解决了预览图像倒置的问题，这里依旧存在一个严重的几乎无解的问题，那就是拍摄的视频是倒过来的，这是新API无法解决的。Google自己的相机，在Nexus 5X上，拍摄的视频显示是这样的：
![手机播放的情况.png](http://upload-images.jianshu.io/upload_images/2103804-a34973ce98daec9f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
我们把视频放到电脑上播放看看是怎样的？请看下图：
![TIM截图20170725181531.png](http://upload-images.jianshu.io/upload_images/2103804-e756c6254f32c8ca.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

我只想对Google 说，你这算什么鬼解决方案？

