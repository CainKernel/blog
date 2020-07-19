抖音的抖动特效的实现原理是，分别对RGB通道进行分离计算不同的大小得到。废话不多数，直接上fragment shader 代码：
```
precision highp float;
varying vec2 textureCoordinate;
uniform sampler2D inputTexture;

uniform float scale;

void main()
{
    vec2 uv = textureCoordinate.xy;
    vec2 scaleCoordinate = vec2((scale - 1.0) * 0.5 + uv.x / scale ,
                                (scale - 1.0) * 0.5 + uv.y / scale);
    vec4 smoothColor = texture2D(inputTexture, scaleCoordinate);

    // 计算红色通道偏移值
    vec4 shiftRedColor = texture2D(inputTexture,
         scaleCoordinate + vec2(-0.1 * (scale - 1.0), - 0.1 *(scale - 1.0)));

    // 计算绿色通道偏移值
    vec4 shiftGreenColor = texture2D(inputTexture,
         scaleCoordinate + vec2(-0.075 * (scale - 1.0), - 0.075 *(scale - 1.0)));

    // 计算蓝色偏移值
    vec4 shiftBlueColor = texture2D(inputTexture,
         scaleCoordinate + vec2(-0.05 * (scale - 1.0), - 0.05 *(scale - 1.0)));

    vec3 resultColor = vec3(shiftRedColor.r, shiftGreenColor.g, shiftBlueColor.b);

    gl_FragColor = vec4(resultColor, smoothColor.a);
}
```
缩放计算如下：
```
    @Override
    public void onDrawFrameBegin() {
        super.onDrawFrameBegin();
        mScale = 1.0f + 0.3f * getInterpolation(mOffset);
        mOffset += 0.06f;
        if (mOffset > 1.0f) {
            mOffset = 0.0f;
        }
        GLES20.glUniform1f(mScaleHandle, mScale);
    }

    private float getInterpolation(float input) {
        return (float)(Math.cos((input + 1) * Math.PI) / 2.0f) + 0.5f;
    }
```
如果用在视频编辑阶段，scale的值可以跟播放器的播放时间绑定得到。

详细实现过程，可以参考本人的开源相机项目：
[CainCamera](https://github.com/CainKernel/CainCamera)
CainCamera的FilterLibrary中有边框模糊的详细实现代码。
