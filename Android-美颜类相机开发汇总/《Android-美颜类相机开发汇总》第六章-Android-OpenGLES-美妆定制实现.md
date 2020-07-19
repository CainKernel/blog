### 美妆介绍
美颜类相机中一般会有彩妆功能，彩妆基本上都是贴图实现，这跟动态贴纸的贴图又不一样，动态贴纸的贴图一般是通过对贴纸进行透视变换实现的。

### 美妆分类
美妆主要包括唇彩、腮红、眼影、眼线、双眼皮、美瞳、眉毛、阴影修容等处理。具体的思路如下：

#### 唇彩
唇彩功能，一般是通过遮罩来处理的。嘴巴的遮罩图三角剖分如下所示：
![唇彩遮罩三角剖分](https://upload-images.jianshu.io/upload_images/2103804-f3fa2a876b1e2fc8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
对应的关键点三角剖分如下：
![唇彩三角剖分](https://upload-images.jianshu.io/upload_images/2103804-35c53994bac8c88f.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

通过遮罩得到嘴唇的位置之后，经过lookup table 颜色查找表映射实现唇彩(口红)功能。

#### 腮红
腮红一般是直接将腮红的素材绘制的脸颊中，因此我们需要根据脸颊的功能，将素材纹理进行三角剖分：
![腮红素材三角剖分](https://upload-images.jianshu.io/upload_images/2103804-4f5f00f176135d85.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
与腮红素材对应的关键点三角剖分如下：
![脸颊关键点三角剖分](https://upload-images.jianshu.io/upload_images/2103804-9511642f4a41c394.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 眉毛部分
眉毛部分是直接绘制素材实现的。

### 眼睛部分
眼睛部分包括眼影、眼线、双眼皮等彩妆，其实实现都是类似的。都需要根据人眼关键点做三角剖分处理。比如眼影的三角剖分如图所示：
![眼影素材三角剖分.png](https://upload-images.jianshu.io/upload_images/2103804-65fb449e82b66e77.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![眼睛关键点三角剖分](https://upload-images.jianshu.io/upload_images/2103804-217aa99a6dffa007.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 美瞳
一般情况下美瞳的实现是将一个圆形的瞳孔绘制到眼睛的瞳孔上，覆盖了原来的瞳孔，然后根据前面的眼睛遮罩将，超出眼眶的瞳孔部分裁掉。

### 美妆的实现
经过前面的三角剖分和数据标注之后，我们就可以根据数据来绘制实现美妆功能了。

在拿到三角剖分数据之后，我们就可以通过glDrawElements 方法，将素材的纹理坐标绘制出来，这样素材就贴合到人脸上了。并且这样做比单纯的绘制素材更好地处理扭头、抬头、低头等原因导致的形变。Triangulation本身就能很好地处理形变过程，由于人脸的关键点是固定的，所以我们并不需要每次都用Delaunay三角算法来计算三角形，能够更好地做实时渲染处理。

对于唇彩来说，一般是通过Lookup Table 颜色查找表对嘴唇的颜色进行映射得到。由于嘴唇的颜色比较单一，所以我们可以采用 64 x 64 大小 lut 来映射唇彩纹理。OpenGL的shader实现如下：
```
     if (makeupType == 3) { // 映射唇彩

        lowp vec4 textureColor = texture2D(inputTexture, textureCoordinate.xy);

        lowp vec4 lipMaskColor = texture2D(maskTexture, maskCoordinate.xy);

        if (lipMaskColor.r > 0.005) {
            mediump vec2 quad1;
            mediump vec2 quad2;
            mediump vec2 texPos1;
            mediump vec2 texPos2;

            mediump float blueColor = textureColor.b * 15.0;

            quad1.y = floor(floor(blueColor) / 4.0);
            quad1.x = floor(blueColor) - (quad1.y * 4.0);

            quad2.y = floor(ceil(blueColor) / 4.0);
            quad2.x = ceil(blueColor) - (quad2.y * 4.0);

            texPos1.xy = (quad1.xy * 0.25) + 0.5/64.0 + ((0.25 - 1.0/64.0) * textureColor.rg);
            texPos2.xy = (quad2.xy * 0.25) + 0.5/64.0 + ((0.25 - 1.0/64.0) * textureColor.rg);

            lowp vec3 newColor1 = texture2D(materialTexture, texPos1).rgb;
            lowp vec3 newColor2 = texture2D(materialTexture, texPos2).rgb;

            lowp vec3 newColor = mix(newColor1, newColor2, fract(blueColor));

            textureColor = vec4(newColor, 1.0) * (lipMaskColor.r * strength);
        } else {
            textureColor = vec4(0.0, 0.0, 0.0, 0.0);
        }
        gl_FragColor = textureColor;

    }
```
对于其他素材来说，就是直接绘制到FBO即可：
```
    if (makeupType == 1) { // 绘制不带遮罩的彩妆素材，此时inputTexture是素材纹理

        lowp vec4 textureColor = texture2D(inputTexture, textureCoordinate.xy);
        gl_FragColor = textureColor * strength;

    }
```
其他素材比如眼影、眼线、眉毛等也是一样的过程，都是找到素材纹理顶点，人脸纹理顶点，然后通过glDrawElements方法将素材的三角剖分数据绘制出来即可。

至此，我们就介绍完彩妆功能了。

由于缺乏彩妆素材，这里只介绍如何彩妆的大体流程，大致流程可以参考本人的项目：
[CainCamera](https://github.com/CainKernel/CainCamera)
项目只做了唇彩的大概实现过程。其他的彩妆功能，大家可以自行实现。
