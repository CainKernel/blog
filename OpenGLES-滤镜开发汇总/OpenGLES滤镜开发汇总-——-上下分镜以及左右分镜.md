上下分镜的fragment shader 如下：
```
varying highp vec2 textureCoordinate;

uniform sampler2D inputImageTexture;

void main() {
    vec2 uv = textureCoordinate;
    if (uv.y < 0.5) {
        uv.y = 1.0 - uv.y;
    }
    gl_FragColor = texture2D(inputImageTexture, fract(uv));
}
```
实现效果如下：
![上下分镜](https://upload-images.jianshu.io/upload_images/2103804-2d7befe08ae096d2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

同样地，我们可以通过改变UV坐标的x轴来得到左右分镜：
```
varying highp vec2 textureCoordinate;

uniform sampler2D inputImageTexture;

void main() {
    vec2 uv = textureCoordinate;
    if (uv.x > 0.5) {
        uv.x = 1.0 - uv.x;
    }
    gl_FragColor = texture2D(inputImageTexture, fract(uv));
}
```
