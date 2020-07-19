观察抖音的灵魂出窍滤镜，可以看到主图像没变化，新增了一张经过缩放后的纹理，跟主图像进行一定alpha处理的线性混合得到。可以参考我写的缩放滤镜，得到fragment shader 如下：
```
precision highp float;
varying vec2 textureCoordinate;
uniform sampler2D inputTexture;

uniform float scale;

void main() {
     vec2 uv = textureCoordinate.xy;
     // 输入纹理
     vec4 sourceColor = texture2D(inputTexture, fract(uv));
     // 将纹理坐标中心转成(0.0, 0.0)再做缩放
     vec2 center = vec2(0.5, 0.5);
     uv -= center;
     uv = uv / scale;
     uv += center;
     // 缩放纹理
     vec4 scaleColor = texture2D(inputTexture, fract(uv));
     // 线性混合
     gl_FragColor = mix(sourceColor, scaleColor, 0.5 * (0.6 - fract(scale)));
}
```
缩放值插值公式如下：
```
    @Override
    public void onDrawFrameBegin() {
        super.onDrawFrameBegin();
        mScale = 1.0f + 0.5f * getInterpolation(mOffset);
        mOffset += 0.04f;
        if (mOffset > 1.0f) {
            mOffset = 0.0f;
        }
        GLES20.glUniform1f(mScaleHandle, mScale);
    }

    private float getInterpolation(float input) {
        return (float)(Math.cos((input + 1) * Math.PI) / 2.0f) + 0.5f;
    }
```
备注：抖音的灵魂出窍滤镜是跟视频操作时长绑定的，并且插值公式也不一定是这样的，这里只是模拟了实时渲染环境下相近的情况。有兴趣的可以自行修改，实现百分百的模拟效果。

详细实现过程，可以参考本人的开源相机项目：
[CainCamera](https://github.com/CainKernel/CainCamera)
CainCamera的FilterLibrary中有边框模糊的详细实现代码。
