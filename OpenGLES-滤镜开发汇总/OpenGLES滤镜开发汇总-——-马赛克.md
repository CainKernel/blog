#### 什么是马赛克

#### 马赛克原理

#### 马赛克的实现

* 方格马赛克的glsl实现如下:
```
precision highp float;

varying vec2 textureCoordinate; // 纹理坐标

uniform sampler2D inputTexture; // 输入图像纹理

uniform float imageWidthFactor; // 图像宽度
uniform float imageHeightFactor;// 图像高度

uniform float mosaicSize;       // 马赛克大小(像素值)

void main()
{
    vec2 uv  = textureCoordinate.xy;
    // 计算出马赛克的宽度
    float dx = mosaicSize * imageWidthFactor;
    // 计算出马赛克的高度
    float dy = mosaicSize * imageHeightFactor;
    // 使用floor函数计算出横坐标和纵坐标经过马赛克变换后的值
    vec2 coord = vec2(dx * floor(uv.x / dx), dy * floor(uv.y / dy));
    // 计算出马赛克的颜色
    vec3 tc = texture2D(inputTexture, coord).xyz;
    // 输出马赛克图像
    gl_FragColor = vec4(tc, 1.0);
}
```
得到的马赛克效果如下：
![马赛克效果](https://upload-images.jianshu.io/upload_images/2103804-36de318827a8638b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

* 圆形马赛克的实现：
我们经过上一步，可以得到基本的马赛克效果，我们实现一个圆形的马赛克效果。圆形马赛克其实就是在原来马赛克的基础上，判断马赛克格子中的元素是否落在圆内，圆外部的像素保留。fragment shader如下：
```
precision highp float;
uniform sampler2D inputTexture;
varying vec2 textureCoordinate;

uniform float imageWidth;     // 图片宽度
uniform float imageHeight;    // 图片高度
uniform float mosaicSize;

void main(void)
{
    vec2 texSize = vec2(imageWidth, imageHeight);
    // 计算实际图像位置
    vec2 xy = vec2(textureCoordinate.x * texSize.x, textureCoordinate.y * texSize.y);
    // 计算某一个小mosaic的中心坐标
    vec2 mosaicCenter = vec2(floor(xy.x / mosaicSize) * mosaicSize + 0.5 * mosaicSize,
                         floor(xy.y / mosaicSize) * mosaicSize + 0.5 * mosaicSize);
    // 计算距离中心的长度
    vec2 delXY = mosaicCenter - xy;
    float delLength = length(delXY);
    // 换算回纹理坐标系
    vec2 uvMosaic = vec2(mosaicCenter.x / texSize.x, mosaicCenter.y / texSize.y);

    vec4 color;
    if (delLength < 0.5 * mosaicSize) {
        color = texture2D(inputTexture, uvMosaic);
    } else {
        color = texture2D(inputTexture, textureCoordinate);
    }
    gl_FragColor = color;
}
```
![圆形马赛克效果](https://upload-images.jianshu.io/upload_images/2103804-639f28358250aa44.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

* 六边形马赛克的实现：
 六边形马赛克滤镜可以参考以下这篇文章:
[『openframeworks』shader制作六边形马赛克效果](https://blog.csdn.net/simpledrunk/article/details/17095821)
修改之后得到的fragment shader 如下：
```
precision highp float;
uniform sampler2D inputTexture;
varying vec2 textureCoordinate;

uniform float mosaicSize;   // 马赛克大小

void main (void)
{
    float length = mosaicSize;
    float TR = 0.866025;
    float x = textureCoordinate.x;
    float y = textureCoordinate.y;
    int wx = int(x / 1.5 / length);
    int wy = int(y / TR / length);
    vec2 v1, v2, vn;
    if (wx/2 * 2 == wx) {
        if (wy/2 * 2 == wy) {
            v1 = vec2(length * 1.5 * float(wx), length * TR * float(wy));
            v2 = vec2(length * 1.5 * float(wx + 1), length * TR * float(wy + 1));
        } else {
            v1 = vec2(length * 1.5 * float(wx), length * TR * float(wy + 1));
            v2 = vec2(length * 1.5 * float(wx + 1), length * TR * float(wy));
        }
    } else {
        if (wy/2 * 2 == wy) {
            v1 = vec2(length * 1.5 * float(wx), length * TR * float(wy + 1));
            v2 = vec2(length * 1.5 * float(wx + 1), length * TR * float(wy));
        } else {
            v1 = vec2(length * 1.5 * float(wx), length * TR * float(wy));
            v2 = vec2(length * 1.5 * float(wx + 1), length * TR * float(wy + 1));
        }
    }
    float s1 = sqrt(pow(v1.x - x, 2.0) + pow(v1.y - y, 2.0));
    float s2 = sqrt(pow(v2.x - x, 2.0) + pow(v2.y - y, 2.0));
    if (s1 < s2) {
        vn = v1;
    } else {
        vn = v2;
    }
    vec4  color = texture2D(inputTexture, vn);
    gl_FragColor = color;
}
```
六边形马赛克的效果如下：
![六边形马赛克效果](https://upload-images.jianshu.io/upload_images/2103804-fb7d271ae72f6957.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
这里的六边形并不是正六边形，这是由于没有做长宽比的计算。你也可以在此基础上添加长宽比实现正六边形的马赛克。这里不做详细介绍，有兴趣的可以去尝试实现以下。

* 三角形马赛克的实现：
三角形马赛克其实就是在原来六边形马赛克的基础上，将六边形分成留个三角形即可。可以参考这篇文章：
[『openframeworks』shader制作三角形马赛克效果](https://blog.csdn.net/simpledrunk/article/details/17170965)
修改之后得到的fragment shader 如下：
```
precision highp float;
uniform sampler2D inputTexture;
varying vec2 textureCoordinate;

// len 是六边形的边长
uniform float mosaicSize;

void main (void){
    const float TR = 0.866025;  // .5*(3)^.5
    const float PI6 = 0.523599; // PI/6

    float x = textureCoordinate.x;
    float y = textureCoordinate.y;

    // 1.5*len 是矩形矩阵的长，TR*len 是宽
    // 计算矩形矩阵的顶点坐标 (0,0)(0,1)(1,0)(1,1)
    int wx = int(x/(1.5 * mosaicSize));
    int wy = int(y/(TR * mosaicSize));

    vec2 v1, v2, vn;

    // 判断是矩形的哪个顶点，上半部还是下半部
    if (wx / 2 * 2 == wx) {
        if (wy/2 * 2 == wy) {
            v1 = vec2(mosaicSize * 1.5 * float(wx), mosaicSize * TR * float(wy));
            v2 = vec2(mosaicSize * 1.5 * float(wx + 1), mosaicSize * TR * float(wy + 1));
        } else {
            v1 = vec2(mosaicSize * 1.5 * float(wx), mosaicSize * TR * float(wy + 1));
            v2 = vec2(mosaicSize * 1.5 * float(wx + 1), mosaicSize * TR * float(wy));
        }
    } else {
        if (wy/2 * 2 == wy) {
            v1 = vec2(mosaicSize * 1.5 * float(wx), mosaicSize * TR * float(wy + 1));
            v2 = vec2(mosaicSize * 1.5 * float(wx+1), mosaicSize * TR * float(wy));
        } else {
            v1 = vec2(mosaicSize * 1.5 * float(wx), mosaicSize * TR * float(wy));
            v2 = vec2(mosaicSize * 1.5 * float(wx + 1), mosaicSize * TR * float(wy+1));
        }
    }
    // 计算参考点与当前纹素的距离
    float s1 = sqrt(pow(v1.x - x, 2.0) + pow(v1.y - y, 2.0));
    float s2 = sqrt(pow(v2.x - x, 2.0) + pow(v2.y - y, 2.0));
    // 选择距离小的参考点
    if (s1 < s2) {
        vn = v1;
    } else {
        vn = v2;
    }

    vec4 mid = texture2D(inputTexture, vn);
    float a = atan((x - vn.x)/(y - vn.y)); // 计算夹角
    // 分别计算六个三角形的中心点坐标，之后将作为参考点
    vec2 area1 = vec2(vn.x, vn.y - mosaicSize * TR / 2.0);
    vec2 area2 = vec2(vn.x + mosaicSize / 2.0, vn.y - mosaicSize * TR / 2.0);
    vec2 area3 = vec2(vn.x + mosaicSize / 2.0, vn.y + mosaicSize * TR / 2.0);
    vec2 area4 = vec2(vn.x, vn.y + mosaicSize * TR / 2.0);
    vec2 area5 = vec2(vn.x - mosaicSize / 2.0, vn.y + mosaicSize * TR / 2.0);
    vec2 area6 = vec2(vn.x - mosaicSize / 2.0, vn.y - mosaicSize * TR / 2.0);

    // 根据夹角判断是哪个三角形
    if (a >= PI6 && a < PI6 * 3.0) {
        vn = area1;
    } else if (a >= PI6 * 3.0 && a < PI6 * 5.0) {
        vn = area2;
    } else if ((a >= PI6 * 5.0 && a <= PI6 * 6.0) || (a < -PI6 * 5.0 && a > -PI6 * 6.0)) {
        vn = area3;
    } else if (a < -PI6 * 3.0 && a >= -PI6 * 5.0) {
        vn = area4;
    } else if(a <= -PI6 && a> -PI6 * 3.0) {
        vn = area5;
    } else if (a > -PI6 && a < PI6) {
        vn = area6;
    }

    vec4 color = texture2D(inputTexture, vn);
    gl_FragColor = color;
}
```
三角形马赛克效果如下：
![三角形马赛克](https://upload-images.jianshu.io/upload_images/2103804-2f627898d7aadeb8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

可以看到，这里的马赛克不是正三角形的。由于没有处理长宽比，所以得不到正三角形的马赛克，你也可以添加长宽比来计算。这里不做相似的介绍，有兴趣的可以去尝试以下。

详细实现过程，可以参考本人的开源相机项目：
[CainCamera](https://github.com/CainKernel/CainCamera)
CainCamera的FilterLibrary中有马赛克的全部实现。
