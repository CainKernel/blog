#### FaceU激萌相机中的边框模糊效果
在FaceU激萌相机中，我们可以看到一个类似边框做了模糊，然后中间放图像的效果。FaceU的边框模糊效果如下：
![FaceU的边框模糊效果](https://upload-images.jianshu.io/upload_images/2103804-0d9fd9c9c58cbb04.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### 边框模糊效果分析：
我们来拆分成以下两个部分：内部显示和外部边框模糊部分。
内部的显示内容是跟Full模式比较，可以得到，内部图是一个完整的显示图片，是一张原始输入图进行缩放得到的。
外部边框，我们仔细对比可以发现，也是由输入图像经过模糊处理之后得到。

#### 边框和内容栏的实现
实现的fragment shader 如下：
```
precision mediump float;
uniform sampler2D inputTexture; // 原始图像
uniform sampler2D blurTexture;  // 经过高斯模糊的图像
varying vec2 textureCoordinate;

uniform float blurOffsetX;  // x轴边框模糊偏移值
uniform float blurOffsetY;  // y轴边框模糊偏移值

void main() {
    // uv坐标
    vec2 uv = textureCoordinate.xy;
    vec4 color;
    // 中间为原图，需要缩小
    if (uv.x >= blurOffsetX && uv.x <= 1.0 - blurOffsetX
        && uv.y >= blurOffsetY && uv.y <= 1.0 - blurOffsetY) {
        // 内部UV缩放值
        float scaleX = 1.0 / (1.0 - 2.0 * blurOffsetX);
        float scaleY = 1.0 / (1.0 - 2.0 * blurOffsetY);
        // 计算出内部新的UV坐标
        vec2 newUV = vec2((uv.x - blurOffsetX) * scaleX, (uv.y - blurOffsetY) * scaleY);
        color = texture2D(inputTexture, newUV);
    } else { // 边框部分使用高斯模糊的图像
        color = texture2D(blurTexture, uv);
    }
    gl_FragColor = color;
}
```
我们将需要处理的原图和经过高斯模糊处理的图片传进来，就可以得到以下的效果：
![边框模糊效果](https://upload-images.jianshu.io/upload_images/2103804-c4996a4f5a40fe32.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

至此，我们就实现了仿FaceU边框模糊的效果。仔细对比，我们可以发现，FaceU激萌相机中的边框模糊是一种毛玻璃的效果，而且边沿部分还有横条之类的。毛玻璃效果的实现也不难，只需要在高斯模糊的纹理基础上添加亮度(Luminance)和饱和度(Saturation)的调节即可得到的。为了避免纠纷，这里只做了高斯模糊处理，有兴趣可以自行实现。

详细实现过程，可以参考本人的开源相机项目：
[CainCamera](https://github.com/CainKernel/CainCamera)
CainCamera的FilterLibrary中有边框模糊的详细实现代码。
