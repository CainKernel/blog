### 什么是景深

### 景深的原理

### 景深的实现
完整的glsl代码实现如下：
```
precision highp float;
varying vec2 textureCoordinate;
uniform sampler2D inputTexture;         // 输入纹理
uniform sampler2D blurImageTexture;     // 经过高斯模糊处理的纹理
uniform float inner;    // 内圆半径
uniform float outer;    // 外圆半径
uniform float width;    // 纹理宽度
uniform float height;   // 纹理高度
uniform vec2 center;    // 中心点的位置
uniform vec3 line1;     // 前景深
uniform vec3 line2;     // 后景深
uniform float intensity;// 景深程度

void main() {
    vec4 originalColor = texture2D(inputTexture, textureCoordinate);
    vec4 tempColor;
    float ratio = height / width;
    vec2 ellipse = vec2(1, ratio * ratio);
    float fx = (textureCoordinate.x - center.x);
    float fy = (textureCoordinate.y - center.y);
    // 用椭圆方程求离中心点的距离
    float dist = sqrt(fx * fx * ellipse.x + fy * fy * ellipse.y);
    // 如果小于内圆半径，则直接输出原图，否则拿原始纹理跟高斯模糊的纹理按照不同的半径进行alpha混合
    if (dist < inner) {
        tempColor = originalColor;
    } else {
        vec3 point = vec3(textureCoordinate.x, textureCoordinate.y, 1.0);
        float value1 = dot(line1, point);
        float value2 = dot(line2, point);
        if (value1 >= 0.0 && value2 >= 0.0) {
            tempColor = originalColor;
        } else {
            vec4 blurColor = texture2D(blurImageTexture, textureCoordinate);
            float lineAlpha = max(-value1 / 0.15, -value2 / 0.15);
            float alpha = (dist - inner)/outer;
            alpha = min(lineAlpha, alpha);
            alpha = clamp(alpha, 0.0, 1.0);
            tempColor = mix(originalColor, blurColor, alpha);
        }
    }
    gl_FragColor = mix(originalColor, tempColor, intensity);
}
```
景深的效果如下：
![景深效果](https://upload-images.jianshu.io/upload_images/2103804-dad7e83b899efbf3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

详细实现过程，可以参考本人的开源相机项目：
[CainCamera](https://github.com/CainKernel/CainCamera)
CainCamera的FilterLibrary中有景深效果的实现。
