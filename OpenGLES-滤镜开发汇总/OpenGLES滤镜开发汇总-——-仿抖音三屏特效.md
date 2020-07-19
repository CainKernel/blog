分屏特效中的三屏特效。就是分成三等份，取中间部分。shader如下：
```
// 仿抖音三屏特效
precision highp float;
uniform sampler2D inputTexture;
varying highp vec2 textureCoordinate;

void main() {
    highp vec2 uv = textureCoordinate;
    if (uv.y < 1.0 / 3.0) {
        uv.y = uv.y + 1.0 / 3.0;
    } else if (uv.y > 2.0 / 3.0) {
        uv.y = uv.y - 1.0 / 3.0;
    }
    gl_FragColor = texture2D(inputTexture, uv);
}
```
效果如下：
![三分屏特效.png](https://upload-images.jianshu.io/upload_images/2103804-ed4fb57e8d74024c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
