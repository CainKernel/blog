最近，抖音上有个比较火的三重影像滤镜，如下图所示：
![抖音三重影像滤镜](https://upload-images.jianshu.io/upload_images/2103804-e3a134c7c5d1bfa0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

可以看到，三重影像的部分是由原图像分成三等份，取图像中间的部分，用不同的LookupTable颜色查找表映射得到的，不过抖音的三重影像滤镜在低端处理器(比如高通625)上会比较卡。这里我介绍如何实现并对此进行优化吧。

#### 三重影像的实现
可以看到前面说的，三重影像滤镜本身是取中间的图像渲染完成的。我们从uv坐标出发，将图像映射成三重影像的shader代码如下所示：
vertex shader:
```
attribute vec4 position;
attribute vec4 inputTextureCoordinate;
varying vec2 textureCoordinate;
void main()
{
    gl_Position = position;
    textureCoordinate = inputTextureCoordinate.xy;
}
```
fragment shader:
```
varying highp vec2 textureCoordinate;
uniform sampler2D inputImageTexture;
void main() 
{
    highp vec2 uv = textureCoordinate;
    vec4 color;
    if (uv.y >= 0.0 && uv.y <= 0.33) { // 上层
        vec2 coordinate = vec2(uv.x, uv.y + 0.33);
        color = texture2D(inputImageTexture, coordinate);
    } else if (uv.y > 0.33 && uv.y <= 0.67) {   // 中间层
        color = texture2D(inputImageTexture, uv);
    } else {    // 下层
        vec2 coordinate = vec2(uv.x, uv.y - 0.33);
        color = texture2D(inputImageTexture, coordinate);
    }
    gl_FragColor = color;
}
```
以上代码，可以直接使用GPUImage来接入即可。
实现的效果如下：
![三重影像实现](https://upload-images.jianshu.io/upload_images/2103804-9cff09f741793936.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
到这里，我们就实现了一个三重影像的滤镜了，至少图像是正确了。仔细观察发现，抖音的三重影像滤镜的每一重影像都有一个不同的滤镜。那么我们如何解决这个问题呢？接下来，我们在这基础上添加一个加载LUT的方法。

### 给图像添加LookupTable滤镜
我们在fragment shader 里面添加一个加载LookupTable的方法，如下所示：
```
vec4 getLutColor(vec4 textureColor, sampler2D lookupTexture) {
    mediump float blueColor = textureColor.b * 63.0;

    mediump vec2 quad1;
    quad1.y = floor(floor(blueColor) / 8.0);
    quad1.x = floor(blueColor) - (quad1.y * 8.0);

    mediump vec2 quad2;
    quad2.y = floor(ceil(blueColor) / 8.0);
    quad2.x = ceil(blueColor) - (quad2.y * 8.0);

    highp vec2 texPos1;
    texPos1.x = (quad1.x * 0.125) + 0.5/512.0 + ((0.125 - 1.0/512.0) * textureColor.r);
    texPos1.y = (quad1.y * 0.125) + 0.5/512.0 + ((0.125 - 1.0/512.0) * textureColor.g);

    highp vec2 texPos2;
    texPos2.x = (quad2.x * 0.125) + 0.5/512.0 + ((0.125 - 1.0/512.0) * textureColor.r);
    texPos2.y = (quad2.y * 0.125) + 0.5/512.0 + ((0.125 - 1.0/512.0) * textureColor.g);

    lowp vec4 newColor1 = texture2D(lookupTexture, texPos1);
    lowp vec4 newColor2 = texture2D(lookupTexture, texPos2);

    lowp vec4 newColor = mix(newColor1, newColor2, fract(blueColor));
    vec4 color = vec4(newColor.rgb, textureColor.w);
    return color;
}
```
然后在 shader 里面 把gl_FragColor 指向经过LUT变换后的颜色即可，完整的shader如下：
```
varying highp vec2 textureCoordinate;

uniform sampler2D inputImageTexture;
uniform sampler2D lookupTable1; // 上层lookupTable
uniform sampler2D lookupTable2; // 中层lookupTable
uniform sampler2D lookupTable3; // 下层lookupTable

// textureColor:输入纹理颜色，lookupTexture：lut纹理
vec4 getLutColor(vec4 textureColor, sampler2D lookupTexture) {
    mediump float blueColor = textureColor.b * 63.0;

    mediump vec2 quad1;
    quad1.y = floor(floor(blueColor) / 8.0);
    quad1.x = floor(blueColor) - (quad1.y * 8.0);

    mediump vec2 quad2;
    quad2.y = floor(ceil(blueColor) / 8.0);
    quad2.x = ceil(blueColor) - (quad2.y * 8.0);

    highp vec2 texPos1;
    texPos1.x = (quad1.x * 0.125) + 0.5/512.0 + ((0.125 - 1.0/512.0) * textureColor.r);
    texPos1.y = (quad1.y * 0.125) + 0.5/512.0 + ((0.125 - 1.0/512.0) * textureColor.g);

    highp vec2 texPos2;
    texPos2.x = (quad2.x * 0.125) + 0.5/512.0 + ((0.125 - 1.0/512.0) * textureColor.r);
    texPos2.y = (quad2.y * 0.125) + 0.5/512.0 + ((0.125 - 1.0/512.0) * textureColor.g);

    lowp vec4 newColor1 = texture2D(lookupTexture, texPos1);
    lowp vec4 newColor2 = texture2D(lookupTexture, texPos2);

    lowp vec4 newColor = mix(newColor1, newColor2, fract(blueColor));
    vec4 color = vec4(newColor.rgb, textureColor.w);
    return color;
}

void main() 
{
    highp vec2 uv = textureCoordinate;
    vec4 color;
    if (uv.y >= 0.0 && uv.y <= 0.33) { // 上层
        vec2 coordinate = vec2(uv.x, uv.y + 0.33);
        vec4 textureColor = texture2D(inputImageTexture, coordinate);
        color = getLutColor(textureColor, lookupTable1);
    } else if (uv.y > 0.33 && uv.y <= 0.67) {   // 中间层
        vec4 textureColor = texture2D(inputImageTexture, uv);
        color = getLutColor(textureColor, lookupTable2);
    } else {    // 下层
        vec2 coordinate = vec2(uv.x, uv.y - 0.33);
        vec4 textureColor = texture2D(inputImageTexture, coordinate);
        color = getLutColor(textureColor, lookupTable3);
    }
    gl_FragColor = color;
}
```
其中，lookupTable1、lookupTable2、lookupTable3分别对应上中下层图像的lut，绘制阶段我们将LUT加载进来，加载代码如下：
```
        GLES20.glActiveTexture(GLES30.GL_TEXTURE1);
        GLES20.glBindTexture(GLES20.GL_TEXTURE_2D, mLookupTable1);
        GLES20.glUniform1i(mLookupTable1Handle, 1);

        GLES20.glActiveTexture(GLES30.GL_TEXTURE2);
        GLES20.glBindTexture(GLES20.GL_TEXTURE_2D, mLookupTable2);
        GLES20.glUniform1i(mLookupTable2Handle, 2);

        GLES20.glActiveTexture(GLES30.GL_TEXTURE3);
        GLES20.glBindTexture(GLES20.GL_TEXTURE_2D, mLookupTable3);
        GLES20.glUniform1i(mLookupTable3Handle, 3);
```
最终实现的效果如下：
![三重影像最终效果](https://upload-images.jianshu.io/upload_images/2103804-888a4d4fc994d93d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

如果你还想对不同图像进行其他的处理比如调节图像变得红润白皙之类的，跟加载LUT的方法类似，分别对每重图像进行处理即可。到这里，三重影像的效果就实现了，而且你可以对比看到，这样的实现在低端处理器上跑起来比较流畅。这里并不需要处理三张大图然后再合成。举个例子，720P的纹理大小，如果先处理成三张大图，则GPU需要处理的数据是 1280 x720 x 3，然后再把三张图片合成，而我这里提到的方案，只需要处理1280 x 720 x 1的数据量，这里处理的数据量上相差了3倍之多。处理数据量越大，在性能比较弱的低端处理器上很大概率会触发bottleneck，导致预览帧率严重降低的，取景区卡顿的现象。
