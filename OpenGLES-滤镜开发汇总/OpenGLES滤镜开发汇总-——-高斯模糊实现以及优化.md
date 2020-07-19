#### 高斯模糊
关于高斯模糊的基本概念，可以参考以下这篇文章内容：
https://blog.csdn.net/Serious_Tanx/article/details/53366438
这里就不多讲解了

#### OpenGLES实现高斯模糊
我们看看GPUImage中的高斯模糊是如何实现的：
GPUImage中的高斯模糊是分成横向和纵向进行低通滤波，然后再做混合得到的。
shader如下所示：
vertex shader:
```
uniform mat4 uMVPMatrix;
attribute vec4 aPosition;
attribute vec4 aTextureCoord;

// 高斯算子大小(3 x 3)
const int GAUSSIAN_SAMPLES = 9;

uniform float texelWidthOffset;
uniform float texelHeightOffset;

varying vec2 textureCoordinate;
varying vec2 blurCoordinates[GAUSSIAN_SAMPLES];

void main()
{
    gl_Position = uMVPMatrix * aPosition;
    textureCoordinate = aTextureCoord.xy;

    int multiplier = 0;
    vec2 blurStep;
    vec2 singleStepOffset = vec2(texelHeightOffset, texelWidthOffset);

    for (int i = 0; i < GAUSSIAN_SAMPLES; i++) {
        multiplier = (i - ((GAUSSIAN_SAMPLES - 1) / 2));
        blurStep = float(multiplier) * singleStepOffset;
        blurCoordinates[i] = aTextureCoord.xy + blurStep;
    }
}
```
fragment shader:
```
precision mediump float;
varying highp vec2 textureCoordinate;
uniform sampler2D inputTexture;
const lowp int GAUSSIAN_SAMPLES = 9;
varying highp vec2 blurCoordinates[GAUSSIAN_SAMPLES];

void main()
{
	lowp vec3 sum = vec3(0.0);
   lowp vec4 fragColor=texture2D(inputTexture,textureCoordinate);

    sum += texture2D(inputTexture, blurCoordinates[0]).rgb * 0.05;
    sum += texture2D(inputTexture, blurCoordinates[1]).rgb * 0.09;
    sum += texture2D(inputTexture, blurCoordinates[2]).rgb * 0.12;
    sum += texture2D(inputTexture, blurCoordinates[3]).rgb * 0.15;
    sum += texture2D(inputTexture, blurCoordinates[4]).rgb * 0.18;
    sum += texture2D(inputTexture, blurCoordinates[5]).rgb * 0.15;
    sum += texture2D(inputTexture, blurCoordinates[6]).rgb * 0.12;
    sum += texture2D(inputTexture, blurCoordinates[7]).rgb * 0.09;
    sum += texture2D(inputTexture, blurCoordinates[8]).rgb * 0.05;

	gl_FragColor = vec4(sum, fragColor.a);
}
```
我们可以看到计算量有点大。vertex shader中，做了9次的偏移坐标运算。这里是否可以节省效率？
答案是可以的，接下来我们讲解如何进行高斯模糊的优化。

#### 高斯模糊的优化
首先我们来看下这两篇文章：
[OpenGL（十六）通过 卷积 实现： 边缘混合 、 Blur 和 高斯模糊](https://blog.csdn.net/fansongy/article/details/79263735)
[iOS 图像处理系列 - 基于GPUImage的滤镜实现及优化](https://cloud.tencent.com/developer/article/1167273) 

上面两篇文章都有讲到优化的一些思路。尤其是天天P图团队的文章中讲到了如何巧妙实现高斯算子的证明过程。本人思考了一下，在此基础上，进行了进一步的优化。
对于高斯算子来说，对于当前像素来说，一个5x5的高斯算子，左右的偏移值均为2。因此，我们可以得到偏移坐标，在VertexShader中，计算的过程如下：
```
// 优化后的高斯模糊
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
在得到偏移坐标之后，我们可以就可以在fragment shader中计算高斯算子的像素总和以及平均值了，fragment shader如下：
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
我们将vertex shader 和 fragment shader 的内容替换掉原来的，可以看到，得到的效果几乎一致。而且计算量进一步减少了。vertex shader 中计算偏移量的过程少了几次，fragment shader 中并不需要每一步都乘上一个权重值，从而减少了乘法的运算次数，使得效率得到的进一步的提升。至于高斯算子的大小我们可以通过改变SHIFT_SIZE来实现，高斯算子的边长 = 2 * SHIFT_SIZE + 1。

上面是我们在高斯模糊算法进行优化的，另外一个问题则是，由于经过高斯模糊处理后，图片的细节已经丢失了，那么对于图像来说，我们可以先对图像缩放到一半之后，再做高斯模糊处理，这样需要处理的数据量就变成了原来的1/4左右。因此，在做高斯模糊的时候我们可以对纹理图像宽高均缩小一倍然后再做处理，此时的效率将会得到更进一步的提升。

详细实现过程，可以参考本人的开源相机项目：
[CainCamera](https://github.com/CainKernel/CainCamera)
CainCamera的FilterLibrary中有经过优化后的高斯模糊实现。
