在前一篇文章[Android OpenGLES 实时美颜(磨皮)的优化](https://www.jianshu.com/p/a76a1201ae53)，我们已经介绍了关于实时美颜(磨皮)的一些优化点。但在实际的优化测试中发现，当处理器发热之后，就无法保证预览帧率了，主要还是高斯模糊处理的数据量比较大导致。因此，我们需要寻找新的磨皮方法。
目前市面上关于磨皮方法有好多种，使用PS磨皮经常用到的方法包括高反差保留、高低频、中性灰以及双线性等。其中中性灰和双线性的效率一般，因此，我们从高反差保留、高低频这两种方法中选择。这里选择使用高反差保留法做磨皮处理，PS中的高反差保留法进行磨皮，随手一搜便能找到很多文章，比如：
https://jingyan.baidu.com/article/455a99504d568fa1662778d6.html

接下来，我们尝试着实现文章中讲到的过程。

#### 第一步，对图像做高斯模糊处理
关于高斯模糊的优化，可以参考本人的文章：
[OpenGLES滤镜开发汇总 —— 高斯模糊实现以及优化](https://www.jianshu.com/p/b65313d281a2)

对于人像进行高斯模糊，我们设计一个11x11的高斯算子对图像进行高斯模糊，shader如下：
vertex shader :
```
uniform mat4 uMVPMatrix;
attribute vec4 aPosition;
attribute vec4 aTextureCoord;

// 高斯算子左右偏移值，当偏移值为5时，高斯算子为 11 x 11
const int SHIFT_SIZE = 5;

uniform highp float texelWidthOffset;
uniform highp float texelHeightOffset;

varying vec2 textureCoordinate;
varying vec4 blurShiftCoordinates[SHIFT_SIZE];

void main() {
    gl_Position = uMVPMatrix * aPosition;
    textureCoordinate = aTextureCoord.xy;
    // 偏移步距
    vec2 singleStepOffset = vec2(texelWidthOffset, texelHeightOffset);
    // 记录偏移坐标
    for (int i = 0; i < SHIFT_SIZE; i++) {
        blurShiftCoordinates[i] = vec4(textureCoordinate.xy - float(i + 1) * singleStepOffset,
                                       textureCoordinate.xy + float(i + 1) * singleStepOffset);
    }
}
```
fragment shader:
```
precision mediump float;
varying vec2 textureCoordinate;
uniform sampler2D inputTexture;
const int SHIFT_SIZE = 5; // 高斯算子左右偏移值
varying vec4 blurShiftCoordinates[SHIFT_SIZE];
void main() {
    // 计算当前坐标的颜色值
    vec4 currentColor = texture2D(inputTexture, textureCoordinate);
    mediump vec3 sum = currentColor.rgb;
    // 计算偏移坐标的颜色值总和
    for (int i = 0; i < SHIFT_SIZE; i++) {
        sum += texture2D(inputTexture, blurShiftCoordinates[i].xy).rgb;
        sum += texture2D(inputTexture, blurShiftCoordinates[i].zw).rgb;
    }
    // 求出平均值
    gl_FragColor = vec4(sum * 1.0 / float(2 * SHIFT_SIZE + 1), currentColor.a);
}
```
经过以上的shader进行高斯模糊处理之后，我们得到这样一张高斯模糊图像：
![高斯模糊处理](https://upload-images.jianshu.io/upload_images/2103804-6aedd82c91da0784.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### 第二步，利用高通滤波器做高反差保留
在PS的高反差保留磨皮方法中，高反差保留磨皮混合采用的是强光模式，计算公式为：color = 2 * color1 * color2。因此，我们设计出这样一个高通滤波器，其shader如下：
fragment shader:
```
precision mediump float;
varying vec2 textureCoordinate;
uniform sampler2D inputTexture; // 输入原图
uniform sampler2D blurTexture;  // 高斯模糊图片
const float intensity = 24.0;   // 强光程度
void main() {
    lowp vec4 sourceColor = texture2D(inputTexture, textureCoordinate);
    lowp vec4 blurColor = texture2D(blurTexture, textureCoordinate);
    // 高通滤波之后的颜色值
    highp vec4 highPassColor = sourceColor - blurColor;
    // 对应混合模式中的强光模式(color = 2.0 * color1 * color2)，对于高反差的颜色来说，color1 和color2 是同一个
    highPassColor.r = clamp(2.0 * highPassColor.r * highPassColor.r * intensity, 0.0, 1.0);
    highPassColor.g = clamp(2.0 * highPassColor.g * highPassColor.g * intensity, 0.0, 1.0);
    highPassColor.b = clamp(2.0 * highPassColor.b * highPassColor.b * intensity, 0.0, 1.0);
    // 输出的是把痘印等过滤掉
    gl_FragColor = vec4(highPassColor.rgb, 1.0);
}
```
经过高通滤波器之后，我们得到这样一个纹理图像：
![高通滤波器得到的纹理](https://upload-images.jianshu.io/upload_images/2103804-d5a7d316c6c6c619.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

可以看到，经过三通道强光混合处理后，痘印、边沿等地方都清晰起来了。强光的程度，一般是3的倍数，这里取24倍。

#### 第三步，保边预处理
到这一步，其实我们已经得到了需要过滤颜色值，但在这一张图中，也把边沿的颜色差值包含进来了。我们接下来需要过滤掉边沿的颜色差值。这样在后续的处理中，我们可以保留边沿的细节不被模糊掉。因此接下来，我们需要将经过高通滤波得到的纹理，再做一次高斯模糊。不过这一次不能11 x11 这么大的高斯算子，我们选择一个 5 x 5 大小的高斯算子。高斯模糊的shader 如下：
vertex shader:
```
uniform mat4 uMVPMatrix;
attribute vec4 aPosition;
attribute vec4 aTextureCoord;

// 高斯算子左右偏移值，当偏移值为2时，高斯算子为5 x 5
const int SHIFT_SIZE = 2;

uniform highp float texelWidthOffset;
uniform highp float texelHeightOffset;

varying vec2 textureCoordinate;
varying vec4 blurShiftCoordinates[SHIFT_SIZE];

void main() {
    gl_Position = uMVPMatrix * aPosition;
    textureCoordinate = aTextureCoord.xy;
    // 偏移步距
    vec2 singleStepOffset = vec2(texelWidthOffset, texelHeightOffset);
    // 记录偏移坐标
    for (int i = 0; i < SHIFT_SIZE; i++) {
        blurShiftCoordinates[i] = vec4(textureCoordinate.xy - float(i + 1) * singleStepOffset,
                                       textureCoordinate.xy + float(i + 1) * singleStepOffset);
    }
}
```
fragment shader:
```
precision mediump float;
varying vec2 textureCoordinate;
uniform sampler2D inputTexture;
// 高斯算子左右偏移值，当偏移值为2时，高斯算子为5 x 5
const int SHIFT_SIZE = 2;
varying vec4 blurShiftCoordinates[SHIFT_SIZE];
void main() {
    // 计算当前坐标的颜色值
    vec4 currentColor = texture2D(inputTexture, textureCoordinate);
    mediump vec3 sum = currentColor.rgb;
    // 计算偏移坐标的颜色值总和
    for (int i = 0; i < SHIFT_SIZE; i++) {
        sum += texture2D(inputTexture, blurShiftCoordinates[i].xy).rgb;
        sum += texture2D(inputTexture, blurShiftCoordinates[i].zw).rgb;
    }
    // 求出平均值
    gl_FragColor = vec4(sum * 1.0 / float(2 * SHIFT_SIZE + 1), currentColor.a);
}
```
将高通滤波器得到的纹理，经过高斯模糊处理后，得到这样一张纹理：
![高斯模糊处理的纹理](https://upload-images.jianshu.io/upload_images/2103804-56abe6a998114d62.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

对比高通滤波器处理后的纹理，边沿细节变得模糊了，而且，需要过滤的颜色差值仍旧保留着。到这一步，我们就得到了做磨皮处理的前置纹理。接下来就是高反差保留磨皮的最后也是最重要的一步。

#### 第四步，磨皮调节
经过前面的处理，我们得到一张输入图片的高斯模糊纹理，以及一张高反差保留的高斯模糊纹理。我们使用这两张纹理，通过比较蓝色通道，计算出需要磨皮的实际强度值，与原图进行混合处理，然后输出最终的纹理。shader如下所示：
```
precision mediump float;
varying vec2 textureCoordinate;
uniform sampler2D inputTexture;         // 输入原图
uniform sampler2D blurTexture;          // 原图的高斯模糊纹理
uniform sampler2D highPassBlurTexture;  // 高反差保留的高斯模糊纹理
uniform lowp float intensity;           // 磨皮程度
void main() {
    lowp vec4 sourceColor = texture2D(inputTexture, textureCoordinate);
    lowp vec4 blurColor = texture2D(blurTexture, textureCoordinate);
    lowp vec4 highPassBlurColor = texture2D(highPassBlurTexture, textureCoordinate);
    // 调节蓝色通道值
    mediump float value = clamp((min(sourceColor.b, blurColor.b) - 0.2) * 5.0, 0.0, 1.0);
    // 找到模糊之后RGB通道的最大值
    mediump float maxChannelColor = max(max(highPassBlurColor.r, highPassBlurColor.g), highPassBlurColor.b);
    // 计算当前的强度
    mediump float currentIntensity = (1.0 - maxChannelColor / (maxChannelColor + 0.2)) * value * intensity;
    // 混合输出结果
    lowp vec3 resultColor = mix(sourceColor.rgb, blurColor.rgb, currentIntensity);
    // 输出颜色
    gl_FragColor = vec4(resultColor, 1.0);
}
```
经过上面的处理之后，我们就得到磨皮处理的结果如下：
![磨皮处理效果](https://upload-images.jianshu.io/upload_images/2103804-7c4b9dd24608705e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

可以看到，经过高反差保留磨皮后的结果，磨皮效果还不错，而且720P磨皮处理时，在高通骁龙625处理器上，经过高反差保留磨皮之后，预览帧率能够保持在30FPS左右。我们可以看到，边沿细节还是不够明显，所以，我们可以使用USM锐化增强边沿细节部分。这篇文章就不讲解USM锐化的实现了。

详细实现过程，可以参考本人的开源相机项目：
[CainCamera](https://github.com/CainKernel/CainCamera)
CainCamera的FilterLibrary中有经过优化后的实时美颜(磨皮)实现。
