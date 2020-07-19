四分镜无法就是把整张图片缩成四份，然后分别放在左上角、右上角、左下角、右下角等地方。我们可以通过改变UV坐标得到。
四分镜的fragment shader 如下：
```
varying highp vec2 textureCoordinate;

uniform sampler2D inputImageTexture;

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
    gl_FragColor = texture2D(inputImageTexture, fract(uv));
}
```
实现效果如下：
![四分镜](https://upload-images.jianshu.io/upload_images/2103804-b144dbd30ef5c7ec.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
