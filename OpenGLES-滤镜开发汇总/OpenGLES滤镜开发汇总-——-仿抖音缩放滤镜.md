缩放滤镜实现的过程是根据所缩放值计算出实际的纹理坐标，实现思路——首先将纹理坐标中心点转成(0.0, 0.0)，做缩放，然后再转换回去即可.
fragment shader 如下：
```
precision mediump float;
varying vec2 textureCoordinate;
uniform sampler2D inputTexture;

uniform float scale;

void main() {
    vec2 uv = textureCoordinate.xy;
    // 将纹理坐标中心转成(0.0, 0.0)再做缩放
    vec2 center = vec2(0.5, 0.5);
    uv -= center;
    uv = uv / scale;
    uv += center;

    gl_FragColor = texture2D(inputTexture, uv);
}
```
调节方法如下：
```
    @Override
    public void onDrawFrameBegin() {
        super.onDrawFrameBegin();
        mOffset += plus ? +0.06f : -0.06f;
        if (mOffset >= 1.0f) {
            plus = false;
        } else if (mOffset <= 0.0f) {
            plus = true;
        }
        mScale = 1.0f + 0.5f * getInterpolation(mOffset);
        GLES20.glUniform1f(mScaleHandle, mScale);
    }

    private float getInterpolation(float input) {
        return (float)(Math.cos((input + 1) * Math.PI) / 2.0f) + 0.5f;
    }
```
备注： 抖音的缩放值是跟视频操作时长绑定的，并且插值器的公式不一定是这样的。这里只是模拟了实时渲染环境下相近的情况，有兴趣的可以自行修改。

详细实现过程，可以参考本人的开源相机项目：
[CainCamera](https://github.com/CainKernel/CainCamera)
CainCamera的FilterLibrary中有边框模糊的详细实现代码。
