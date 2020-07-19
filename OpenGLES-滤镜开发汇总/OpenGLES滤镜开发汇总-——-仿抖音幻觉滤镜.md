首先，我们观察抖音的幻觉滤镜，可以看到有两个过程组成。主纹理以及上一帧纹理组成，主纹理经过LUT颜色映射得到。如果拍摄的视频是静态的，则由于上一帧和当前主纹理是图像是一样的，那样整个画面就重叠了，此时是看不出幻觉效果的，只有LUT颜色映射之后的图像效果。我们了解了整个过程的构建，接下来我们来看看如何实现：

1、拿上一帧与主图像进行混合：
```
precision mediump float;
varying vec2 textureCoordinate;
uniform sampler2D inputTexture;     // 当前输入纹理
uniform sampler2D inputTextureLast; // 上一次的纹理

// 分RGB通道混合，不同颜色通道混合值不一样
const lowp vec3 blendValue = vec3(0.1, 0.3, 0.6);

void main() {
    // 当前纹理颜色
    vec4 currentColor = texture2D(inputTexture, textureCoordinate);
    // 上一轮纹理颜色
    vec4 lastColor = texture2D(inputTextureLast, textureCoordinate);
    // 将两者混合
    gl_FragColor = vec4(mix(lastColor.rgb, currentColor.rgb, blend), currentColor.w);
}
```
备注：这里对RGB通道使用不同的值进行线性混合得到，这里可以任意自己调整

2、LUT映射函数
我们可以看到图像是经过滤镜处理得到的，这个过程我们采用一个lut滤镜进行处理，加载lut函数如下：
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
3、经过颜色映射之后，我们得到最终的fragment shader，如下所示：
```
precision mediump float;
varying vec2 textureCoordinate;
uniform sampler2D inputTexture;     // 当前输入纹理
uniform sampler2D inputTextureLast; // 上一次的纹理
uniform sampler2D lookupTable;      // 颜色查找表纹理

// 分RGB通道混合，不同颜色通道混合值不一样
const lowp vec3 blendValue = vec3(0.1, 0.3, 0.6);

// 计算lut映射之后的颜色值
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

void main() {
    // 当前纹理颜色
    vec4 currentColor = texture2D(inputTexture, textureCoordinate);
    // 上一轮纹理颜色
    vec4 lastColor = texture2D(inputTextureLast, textureCoordinate);
    // lut映射的颜色值
    vec4 lutColor = getLutColor(currentColor, lookupTable);
    // 将lut映射之后的纹理与上一轮的纹理进行线性混合
    gl_FragColor = vec4(mix(lastColor.rgb, lutColor.rgb, blendValue ), currentColor.a);
}
```
4、绑定纹理。我们需要对纹理进行绑定，操作如下：
```
    @Override
    public void onDrawFrameBegin() {
        super.onDrawFrameBegin();

        // 绑定上一次纹理
        GLES30.glActiveTexture(GLES30.GL_TEXTURE1);
        GLES30.glBindTexture(getTextureType(), mLastTexture);
        GLES30.glUniform1i(mLastTextureHandle, 1);

        // 绑定lut纹理
        GLES30.glActiveTexture(GLES30.GL_TEXTURE2);
        GLES30.glBindTexture(getTextureType(), mLookupTable);
        GLES30.glUniform1i(mLastTextureHandle, 2);
    }

    /**
     * 设置上一次纹理id
     * @param lastTexture
     */
    public void setLastTexture(int lastTexture) {
        mLastTexture = lastTexture;
    }

    /**
     * 设置lut纹理id
     * @param lookupTableTexture
     */
    public void setLookupTable(int lookupTableTexture) {
        mLookupTable = lookupTableTexture;
    }
```
