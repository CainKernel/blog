分屏特效中的四屏特效。就是缩放成一半，然后四个位置填充整张纹理。shader如下：
```
// 仿抖音四屏特效
precision highp float;
uniform sampler2D inputTexture;
varying highp vec2 textureCoordinate;

void main() {
    vec2 uv = textureCoordinate;
    if (uv.x <= 0.5) {
        uv.x = uv.x * 2.0;
    } else {
        uv.x = (uv.x - 0.5) * 2.0;
    }
    if (uv.y <= 0.5) {
        uv.y = uv.y * 2.0;
    } else {
        uv.y = (uv.y - 0.5) * 2.0;
    }
    gl_FragColor = texture2D(inputTexture, fract(uv));
}
```
效果如下：
![四分屏特效.png](https://upload-images.jianshu.io/upload_images/2103804-edf881670242c757.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
