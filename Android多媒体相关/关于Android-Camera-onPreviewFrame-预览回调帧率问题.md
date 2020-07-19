最近在做相机开发的过程中，关于onPreviewFrame的问题必须单独拿出来说一下的。

公司的相机项目，是通过两个HandlerThread 来对Camera进行控制以及对贴纸、美颜等进行渲染，是通过SurfaceView来实现的。其中，一个HandlerThread 用来控制Camera打开、关闭等操作，另外一个HandlerThread用来控制渲染贴纸、滤镜等操作。按照Google官方的说法，Camera 的控制最好开启一个线程，而不是在UI线程操作，照理说，这样的方案是没什么问题的。但实际上，部分手机的帧率非常不自然，onPreviewFrame回调的帧率过低。

但为啥onPreviewFrame的帧率那么低？带着这个问题，我对相机逐步进行了拆解，一开始我以为是渲染上出现了问题，但事实证明并不是。我把所有的渲染部分操作都剔除，只保留渲染Camera相机流到SurfaceTexture，情况依旧，onPreviewFrame的回调时间一直在60多毫秒。只渲染了Camera相机流到SurfaceTexture，理论上是有20多帧的。这样的情况很明显不正常，我也排除了所有可能造成onPreviewFrame阻塞的情况。

而且，我自己的CainCamera项目中，onPreviewFrame的帧率是正常的，每一帧的回调时间在30多毫秒，有20多帧。通过对比，我发现自己的项目跟公司的项目方案是有差异的。我写的CainCamera项目，是通过单一线程模型控制相机和渲染的，一个HandlerThread同时控制Camera操作和Render渲染。也就是说，这是因为双HandlerThread线程导致的问题？我自己又写了一个Demo验证，发现果然跟我猜想的结论一致，单一线程模型，所有的手机都是正常的，但是双HandlerThread模型下，部分手机的onPreviewFrame回调的帧率表现是一致的，几乎正常的帧率慢了一倍。

我对手机进行了分类，发现所有onPreviewFrame 回调帧率不正常的情况都是MTK的CPU。测试的设备结果如下：
```
高通：
VIVO X9i，高通625
红米Note 4X，高通625
Nexus 5X，高通808
onPreviewFrame回调时间： 单一 HandlerThread， 30~40ms； 双HandlerThread，30~40ms
preview size： 1280 x 720 

联发科:
红米Note2， 联发科X10
魅蓝Note 2，联发科 MTK6573
乐视 X620，联发科X20
onPreviewFrame回调时间：单一HandlerThread， 30 ~ 40ms；双HandlerThread，60~70ms
preview size： 1280 x 720 

操作：
Camera数据流渲染到SurfaceTexture显示到SurfaceView上，设置setPreviewCallbackWithBuffer，查看onPreviewFrame的帧率
```
发现了没有，所有使用了MTK CPU的手机，在单一HandlerThread线程模型上的表现跟高通CPU一致，但双HandlerThread线程模型上，则会导致onPreviewFrame预览回调的帧率降了差不多一倍左右。我只能说MTK的驱动或者CPU的架构有问题。
