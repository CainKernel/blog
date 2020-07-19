#### 德罗斯特效应
德罗斯特效应是一种递归的视觉形式，是指一张图片的某个部分与整张图片相同，如此产生无限循环。

#### 德罗斯特效应的OpenGLES实现
fragment shader 如下：
```
precision highp float;
uniform sampler2D inputTexture;
varying highp vec2 textureCoordinate;

uniform float repeat; // 画面重复的次数

void main() {
    vec2 uv = textureCoordinate;
    // 反向UV坐标
    vec2 invertedUV = 1.0 - uv;
    // 计算重复次数之后的uv值以及偏移值
    vec2 fiter = floor(uv * repeat * 2.0);
    vec2 riter = floor(invertedUV * repeat * 2.0);
    vec2 iter = min(fiter, riter);
    float minOffset = min(iter.x, iter.y);
    // 偏移值
    vec2 offset = (vec2(0.5, 0.5) / repeat) * minOffset;
    // 当前实际的偏移值
    vec2 currenOffset = 1.0 / (vec2(1.0, 1.0) - offset * 2.0);
    // 计算出当前的实际UV坐标
    vec2 currentUV = (uv - offset) * currenOffset;

    gl_FragColor = texture2D(inputTexture, fract(currentUV));
}
```
重复4次的德罗斯特效应的效果如下：
![四重德罗斯特的效果](https://upload-images.jianshu.io/upload_images/2103804-68ccbc23eb6b00de.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

详细实现过程，可以参考本人的开源相机项目：
[CainCamera](https://github.com/CainKernel/CainCamera)
CainCamera的FilterLibrary中有德罗斯特效应滤镜的详细实现代码。
