分屏特效中的模糊分屏特效，好说的，就是把整张图片的先做模糊处理，然后裁剪保留中间1/3的纹理，上层模糊后的纹理经过缩放处理在贴图。缩放倍数大于1.0。分屏的shader 如下：
```
// 仿抖音模糊分屏
precision mediump float;
uniform sampler2D inputTexture; // 原始图像
uniform sampler2D blurTexture;  // 经过高斯模糊的图像
varying vec2 textureCoordinate;

uniform float blurOffsetY;  // y轴边框模糊偏移值
uniform float scale;        // 模糊部分的缩放倍数

void main() {
    // uv坐标
    vec2 uv = textureCoordinate.xy;
    vec4 color;
    // 中间为原图部分
    if (uv.y >= blurOffsetY && uv.y <= 1.0 - blurOffsetY) {
        color = texture2D(inputTexture, uv);
    } else { // 边框部分使用高斯模糊的图像
        vec2 center = vec2(0.5, 0.5);
        uv -= center;
        uv = uv / scale;
        uv += center;
        color = texture2D(blurTexture, uv);
    }
    gl_FragColor = color;
}
```
效果如下：
![模糊分屏特效.png](https://upload-images.jianshu.io/upload_images/2103804-d9cafe770ef4f50b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
