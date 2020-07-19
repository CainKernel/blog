分屏特效中的两屏特效。分成上下两层，uv坐标的y轴在 0.0 ~ 0.5 和 0.5 ~ 1.0 的时候，均填充 0.25 ~ 0.75 区间的纹理图像。shader 如下：
```
// 仿抖音两屏特效
precision highp float;
uniform sampler2D inputTexture;
varying highp vec2 textureCoordinate;

void main() {
    // 纹理坐标
    vec2 uv = textureCoordinate.xy;
    float y;
    if (uv.y >= 0.0 && uv.y <= 0.5) {
        y = uv.y + 0.25;
    } else {
        y = uv.y - 0.25;
    }

    gl_FragColor = texture2D(inputTexture, vec2(uv.x, y));
}

```
效果如下：
![二分屏特效.png](https://upload-images.jianshu.io/upload_images/2103804-a8e795e2fe060d9f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
