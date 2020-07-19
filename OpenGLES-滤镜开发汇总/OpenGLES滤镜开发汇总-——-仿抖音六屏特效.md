分屏特效中的六屏特效。六屏特效是横向三等份，纵向缩成一半，然后填充纹理，只有横向三等份部分是裁剪了中间部分，纵向并没有做裁剪处理。shader  如下：
```
// 仿抖音六屏特效
precision highp float;
uniform sampler2D inputTexture;
varying highp vec2 textureCoordinate;

void main() {
    highp vec2 uv = textureCoordinate;
    // 左右分三屏
    if (uv.x <= 1.0 / 3.0) {
        uv.x = uv.x + 1.0 / 3.0;
    } else if (uv.x >= 2.0 / 3.0) {
        uv.x = uv.x - 1.0 / 3.0;
    }
    // 上下分两屏，保留 0.25 ~ 0.75部分
    if (uv.y <= 0.5) {
        uv.y = uv.y + 0.25;
    } else {
        uv.y = uv.y - 0.25;
    }
    gl_FragColor = texture2D(inputTexture, uv);
}
```
效果如下：
![六分屏特效.png](https://upload-images.jianshu.io/upload_images/2103804-01e9c7b641bcd3bb.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
