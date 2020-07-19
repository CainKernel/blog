Snapdragon Profiler是高通开发的用于调试分析高通Adreno GPU的一款桌面应用，支持Windows、MacOS 和 Linux 。在调试opengles应用程序上能发挥非常重要的作用。该工具能够捕捉CPU、GPU、DSP、内存、功率、网络连接和设备运行时的发热数据等，具有Realtime、Trace Capture、Snapshot Capture 三种模式。实时(Realtime)模式用于实时跟踪数据，跟踪(Trace Capture)模式用于跟踪事件和数据，默认最大值是10秒。快照(Snapshot Capture)模式用于捕获OpenGL ES应用程序的当前帧并可以进行调试，包括单步调试绘制指令，查看和编辑着色器、程序、纹理以及查看像素历史的能力。着色器代码是通过反编译得到，得到的代码跟原glsl代码基本一致，并且可以在截图后修改glsl进行调试。
Snapdragon Profiler 各个版本的下载地址：
[Snapdragon Profiler Linux](https://developer.qualcomm.com/download/sdprofiler/snapdragon-profiler-linux.tar.gz) 
[Snapdragon Profiler Windows](https://developer.qualcomm.com/download/sdprofiler/snapdragon-profiler.zip)
[Snapdragon Profiler MacOS](https://developer.qualcomm.com/download/sdprofiler/snapdragon-profiler-os-x.dmg)
要想下载，首先得有高通的开发者帐号。
本人的Linux环境是ubuntu16.04，Android 设备是Nexus 5X，系统是自己编译的Android 7.1.1 版AOSP。
安装过程如下：
1、安装Mono。Snapdragon Profiler Linux版是基于Linux Mono开发的，因此首先需要在Linux下安装Mono。关于Snapdragon Profiler 更多信息可以参考官方的faq：
https://developer.qualcomm.com/software/snapdragon-profiler/faq
由于每个Linux版本的安装稍微有些不同，具体安装过程请参考官网：
```
http://www.mono-project.com/download/
```
安装过程如下：
导入key
```
$ sudo apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv-keys 3FA7E0328081BFF6A14DA29AA6A19B38D3D831EF
```
添加Mono的地址：
```
$ echo "deb http://download.mono-project.com/repo/ubuntu xenial main" | sudo tee /etc/apt/sources.list.d/mono-official.list
```
更新：
```
$ sudo apt-get update
```
安装Mono：
```
$ sudo apt-get install mono-complete
```
2、Mono安装完成后，进入Snapdragon Profiler目录下运行命令：
```
$ cd /opt/SnapdragonProfiler/
$ ./run_sdp.sh
```
运行时可能会出现以下错误：
```
Unhandled Exception:
System.TypeInitializationException: The type initializer for 'SDPCorePINVOKE' threw an exception. ---> System.TypeInitializationException: The type initializer for 'SWIGExceptionHelper' threw an exception. ---> System.DllNotFoundException: SDPCore
```
根据高通观望的说法：
```
https://developer.qualcomm.com/forum/qdn-forums/software/snapdragon-profiler/29450
```
这是缺少C++Runtime库造成的，解决方法如下：
```
$ sudo apt-get install libc++1 
```
安装完成后，不出意外就能够看到以下界面了：

![主页.png](http://upload-images.jianshu.io/upload_images/2103804-15d48fefad0c5c82.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

连接到手机上，选择Connect to a Device，会出现以下弹框，等待右边的图标变成绿色后点击Connect连接手机：
![连接.png](http://upload-images.jianshu.io/upload_images/2103804-7e3e6803abdca5a1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

在StartPage页面选择New SnapShot Capture，进入后打开opengles的应用程序，我们可以看到页面如下：
![SnapShot Capture选项.png](http://upload-images.jianshu.io/upload_images/2103804-35615e6d9139a3e1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
在Data Sources里面选中对应的应用程序，然后选择GPUShader Processing等你需要调试的应用，调试之前需要添加相应的权限：
```
    <!--高通GPU调试权限-->
    <uses-permission android:name="com.qti.permission.PROFILER" />
    <uses-permission android:name="android.permission.INTERNET" />
```
打开应用后，点击Take Snapshot，然后慢慢等待数据回传。如果操作比较多，这个过程会非常非常慢，这是因为截图的话，由于一帧绘制的Texture 比较多且比较大的时候，传输的数据量会非常大，因此绘制的特效越多，传输越慢。我们可以看到，传输回来的截图情况，右边的Texture 可以看到当前帧的情况，右下角的ImagePreview可以看到Textures中的图片预览。Context相同的Textures 和Program 是一一对应的，我们可以选中Programs的ID，然后在左边的VS、FS中那个就可以看到对应的着色器代码了。中间显示的是捕捉到的图片。底部则是运行的opengles绘制方法，你可以双击相应的方法进行单步执行。另外，你可以双击中间的图片的某个位置，然后在左上角的Pixel History中可以看到DrawCall对应的颜色值，方便调试opengles渲染的颜色、透明度是否正确。
![截图情况.png](http://upload-images.jianshu.io/upload_images/2103804-67fe17b6d2a69667.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

实时模式跟Android Device Monitor的使用方式大同小异，在选中应用的包名后，双击你想要监听的项目，程序的运行状态将会实时反馈出来。
跟踪模式与实时模式类似，只不过最大跟踪10秒的时间，结束后整体状况将会回传显示出你想要监听的项目运行的状况。

至此，Snapdragon Profiler的介绍就到这里，接下来大家就可以欢快地使用该工具调试opengles应用程序啦。



