一、YUV格式
YUV 表示三个分量， Y 表示 亮度（Luminance），即灰度值，UV表示色度（Chrominance），描述图像色彩和饱和度，指定颜色。YUV格式有YUV444、 YUV422 和 YUV420 三种，差别在于：
YUV444： 每个Y分量对应一组UV分量
YUV422：每两个Y分量共用一组UV分量
YUV420：每四个Y分量共用一组UV分量

二、YUV444格式
![YUV444P像素存储](http://upload-images.jianshu.io/upload_images/2103804-6cb306d18d9bc1c0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

三、YUV422格式
1、YCbYCr(YUYV)格式 (打包格式)
![YUV422(YUYV)格式](http://upload-images.jianshu.io/upload_images/2103804-2b11864612736171.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

2、CbYCrY(UYVY)格式 (打包格式)
![YUV422(I422/UYVY)格式](http://upload-images.jianshu.io/upload_images/2103804-7b8b998f4a0ae251.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

3、YUV422P格式 (平面格式) :  YUV422Planar
![YUV422P像素存储](http://upload-images.jianshu.io/upload_images/2103804-7cf59dcfe9fc47ae.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
4、YUV422SP格式(平面格式) :  YUV422SemiPlanar
![YUV422SP像素存储](http://upload-images.jianshu.io/upload_images/2103804-ac17a2ccc7d2284e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

四、YUV420格式
YUV420格式分为 YUV420P 和 YUV420SP两种。
其中YUV420P格式，分为I420 和 YV12 两种，YUV420SP格式分为NV12 和 NV21 两种。他们的存储格式，区别如下：
YUV420P(I420)像素存储：
![YUV420P(I420)像素存储](http://upload-images.jianshu.io/upload_images/2103804-d685d8e497fe7a9e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

YUV420P(YV12)像素存储， 跟I420的UV存储顺序相反：
![YUV420P(YV12)像素存储](http://upload-images.jianshu.io/upload_images/2103804-f5f1d42ad655ea9e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

YUV420SP(NV12)像素存储，UV分量交织存储:
![YUV420SP(NV12)像素存储](http://upload-images.jianshu.io/upload_images/2103804-37cda85d99d000e1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

YUV420SP(NV21)像素存储，UV分量交织存储，跟NV12的UV分量相反：
![YUV420SP(NV21)像素存储](http://upload-images.jianshu.io/upload_images/2103804-f6f5c407b5f3b3c8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

摄像头采集到的数据是RGB24的格式，RGB24 一帧的大小是 width x height x 3Bit， RGB32 一帧的大小则是width x height x 4Bit。YUV420 一帧的大小是 width x height x 1.5Bit。


Android 中的格式定义如下：
```
/**
 * Enumeration defining possible uncompressed image/video formats.
 *
 * ENUMS:
 *  Unused                 : Placeholder value when format is N/A
 *  Monochrome             : black and white
 *  8bitRGB332             : Red 7:5, Green 4:2, Blue 1:0
 *  12bitRGB444            : Red 11:8, Green 7:4, Blue 3:0
 *  16bitARGB4444          : Alpha 15:12, Red 11:8, Green 7:4, Blue 3:0
 *  16bitARGB1555          : Alpha 15, Red 14:10, Green 9:5, Blue 4:0
 *  16bitRGB565            : Red 15:11, Green 10:5, Blue 4:0
 *  16bitBGR565            : Blue 15:11, Green 10:5, Red 4:0
 *  18bitRGB666            : Red 17:12, Green 11:6, Blue 5:0
 *  18bitARGB1665          : Alpha 17, Red 16:11, Green 10:5, Blue 4:0
 *  19bitARGB1666          : Alpha 18, Red 17:12, Green 11:6, Blue 5:0
 *  24bitRGB888            : Red 24:16, Green 15:8, Blue 7:0
 *  24bitBGR888            : Blue 24:16, Green 15:8, Red 7:0
 *  24bitARGB1887          : Alpha 23, Red 22:15, Green 14:7, Blue 6:0
 *  25bitARGB1888          : Alpha 24, Red 23:16, Green 15:8, Blue 7:0
 *  32bitBGRA8888          : Blue 31:24, Green 23:16, Red 15:8, Alpha 7:0
 *  32bitARGB8888          : Alpha 31:24, Red 23:16, Green 15:8, Blue 7:0
 *  YUV411Planar           : U,Y are subsampled by a factor of 4 horizontally
 *  YUV411PackedPlanar     : packed per payload in planar slices
 *  YUV420Planar           : Three arrays Y,U,V.
 *  YUV420PackedPlanar     : packed per payload in planar slices
 *  YUV420SemiPlanar       : Two arrays, one is all Y, the other is U and V
 *  YUV422Planar           : Three arrays Y,U,V.
 *  YUV422PackedPlanar     : packed per payload in planar slices
 *  YUV422SemiPlanar       : Two arrays, one is all Y, the other is U and V
 *  YCbYCr                 : Organized as 16bit YUYV (i.e. YCbYCr)
 *  YCrYCb                 : Organized as 16bit YVYU (i.e. YCrYCb)
 *  CbYCrY                 : Organized as 16bit UYVY (i.e. CbYCrY)
 *  CrYCbY                 : Organized as 16bit VYUY (i.e. CrYCbY)
 *  YUV444Interleaved      : Each pixel contains equal parts YUV
 *  RawBayer8bit           : SMIA camera output format
 *  RawBayer10bit          : SMIA camera output format
 *  RawBayer8bitcompressed : SMIA camera output format
 */
typedef enum OMX_COLOR_FORMATTYPE {
    OMX_COLOR_FormatUnused,
    OMX_COLOR_FormatMonochrome,
    OMX_COLOR_Format8bitRGB332,
    OMX_COLOR_Format12bitRGB444,
    OMX_COLOR_Format16bitARGB4444,
    OMX_COLOR_Format16bitARGB1555,
    OMX_COLOR_Format16bitRGB565,
    OMX_COLOR_Format16bitBGR565,
    OMX_COLOR_Format18bitRGB666,
    OMX_COLOR_Format18bitARGB1665,
    OMX_COLOR_Format19bitARGB1666,
    OMX_COLOR_Format24bitRGB888,
    OMX_COLOR_Format24bitBGR888,
    OMX_COLOR_Format24bitARGB1887,
    OMX_COLOR_Format25bitARGB1888,
    OMX_COLOR_Format32bitBGRA8888,
    OMX_COLOR_Format32bitARGB8888,
    OMX_COLOR_FormatYUV411Planar,
    OMX_COLOR_FormatYUV411PackedPlanar,
    OMX_COLOR_FormatYUV420Planar,
    OMX_COLOR_FormatYUV420PackedPlanar,
    OMX_COLOR_FormatYUV420SemiPlanar,
    OMX_COLOR_FormatYUV422Planar,
    OMX_COLOR_FormatYUV422PackedPlanar,
    OMX_COLOR_FormatYUV422SemiPlanar,
    OMX_COLOR_FormatYCbYCr,
    OMX_COLOR_FormatYCrYCb,
    OMX_COLOR_FormatCbYCrY,
    OMX_COLOR_FormatCrYCbY,
    OMX_COLOR_FormatYUV444Interleaved,
    OMX_COLOR_FormatRawBayer8bit,
    OMX_COLOR_FormatRawBayer10bit,
    OMX_COLOR_FormatRawBayer8bitcompressed,
    OMX_COLOR_FormatL2,
    OMX_COLOR_FormatL4,
    OMX_COLOR_FormatL8,
    OMX_COLOR_FormatL16,
    OMX_COLOR_FormatL24,
    OMX_COLOR_FormatL32,
    OMX_COLOR_FormatYUV420PackedSemiPlanar,
    OMX_COLOR_FormatYUV422PackedSemiPlanar,
    OMX_COLOR_Format18BitBGR666,
    OMX_COLOR_Format24BitARGB6666,
    OMX_COLOR_Format24BitABGR6666,
    OMX_COLOR_FormatKhronosExtensions = 0x6F000000, /**< Reserved region for introducing Khronos Standard Extensions */
    OMX_COLOR_FormatVendorStartUnused = 0x7F000000, /**< Reserved region for introducing Vendor Extensions */
    /**<Reserved android opaque colorformat. Tells the encoder that
     * the actual colorformat will be  relayed by the
     * Gralloc Buffers.
     * FIXME: In the process of reserving some enum values for
     * Android-specific OMX IL colorformats. Change this enum to
     * an acceptable range once that is done.
     * */
    OMX_COLOR_FormatAndroidOpaque = 0x7F000789,
    OMX_TI_COLOR_FormatYUV420PackedSemiPlanar = 0x7F000100,
    OMX_QCOM_COLOR_FormatYVU420SemiPlanar = 0x7FA30C00,
    OMX_QCOM_COLOR_FormatYUV420PackedSemiPlanar64x32Tile2m8ka = 0x7FA30C03,
    OMX_SEC_COLOR_FormatNV12Tiled = 0x7FC00002,
    OMX_QCOM_COLOR_FormatYUV420PackedSemiPlanar32m = 0x7FA30C04,
    OMX_COLOR_FormatMax = 0x7FFFFFFF
} OMX_COLOR_FORMATTYPE;
```

