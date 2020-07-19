在介绍实时美颜算法之前，你可以参考程序员杠把子的博客：
http://blog.csdn.net/oshunz/article/details/50536031
实时美颜算法考虑到性能的影响，PC平台上的很多美颜算法在手机镜头预览渲染的时候是很力不从心的。一般情况下都是基于高斯模糊进行实时美颜，这是性能和效果的折中方案。在程序员杠把子的博客中所描述的实时美颜算法实现的效果并不是很好，程序员杠把子的glsl是这样的:
```
precision mediump float;

varying mediump vec2 textureCoordinate;

uniform sampler2D inputImageTexture;
uniform vec2 singleStepOffset;
uniform mediump float params;

const highp vec3 W = vec3(0.299,0.587,0.114);
vec2 blurCoordinates[20];

float hardLight(float color)
{
	if(color <= 0.5)
		color = color * color * 2.0;
	else
		color = 1.0 - ((1.0 - color)*(1.0 - color) * 2.0);
	return color;
}

void main(){

    vec3 centralColor = texture2D(inputImageTexture, textureCoordinate).rgb;
    blurCoordinates[0] = textureCoordinate.xy + singleStepOffset * vec2(0.0, -10.0);
    blurCoordinates[1] = textureCoordinate.xy + singleStepOffset * vec2(0.0, 10.0);
    blurCoordinates[2] = textureCoordinate.xy + singleStepOffset * vec2(-10.0, 0.0);
    blurCoordinates[3] = textureCoordinate.xy + singleStepOffset * vec2(10.0, 0.0);
    blurCoordinates[4] = textureCoordinate.xy + singleStepOffset * vec2(5.0, -8.0);
    blurCoordinates[5] = textureCoordinate.xy + singleStepOffset * vec2(5.0, 8.0);
    blurCoordinates[6] = textureCoordinate.xy + singleStepOffset * vec2(-5.0, 8.0);
    blurCoordinates[7] = textureCoordinate.xy + singleStepOffset * vec2(-5.0, -8.0);
    blurCoordinates[8] = textureCoordinate.xy + singleStepOffset * vec2(8.0, -5.0);
    blurCoordinates[9] = textureCoordinate.xy + singleStepOffset * vec2(8.0, 5.0);
    blurCoordinates[10] = textureCoordinate.xy + singleStepOffset * vec2(-8.0, 5.0);
    blurCoordinates[11] = textureCoordinate.xy + singleStepOffset * vec2(-8.0, -5.0);
    blurCoordinates[12] = textureCoordinate.xy + singleStepOffset * vec2(0.0, -6.0);
    blurCoordinates[13] = textureCoordinate.xy + singleStepOffset * vec2(0.0, 6.0);
    blurCoordinates[14] = textureCoordinate.xy + singleStepOffset * vec2(6.0, 0.0);
    blurCoordinates[15] = textureCoordinate.xy + singleStepOffset * vec2(-6.0, 0.0);
    blurCoordinates[16] = textureCoordinate.xy + singleStepOffset * vec2(-4.0, -4.0);
    blurCoordinates[17] = textureCoordinate.xy + singleStepOffset * vec2(-4.0, 4.0);
    blurCoordinates[18] = textureCoordinate.xy + singleStepOffset * vec2(4.0, -4.0);
    blurCoordinates[19] = textureCoordinate.xy + singleStepOffset * vec2(4.0, 4.0);

    float sampleColor = centralColor.g * 20.0;
    sampleColor += texture2D(inputImageTexture, blurCoordinates[0]).g;
    sampleColor += texture2D(inputImageTexture, blurCoordinates[1]).g;
    sampleColor += texture2D(inputImageTexture, blurCoordinates[2]).g;
    sampleColor += texture2D(inputImageTexture, blurCoordinates[3]).g;
    sampleColor += texture2D(inputImageTexture, blurCoordinates[4]).g;
    sampleColor += texture2D(inputImageTexture, blurCoordinates[5]).g;
    sampleColor += texture2D(inputImageTexture, blurCoordinates[6]).g;
    sampleColor += texture2D(inputImageTexture, blurCoordinates[7]).g;
    sampleColor += texture2D(inputImageTexture, blurCoordinates[8]).g;
    sampleColor += texture2D(inputImageTexture, blurCoordinates[9]).g;
    sampleColor += texture2D(inputImageTexture, blurCoordinates[10]).g;
    sampleColor += texture2D(inputImageTexture, blurCoordinates[11]).g;
    sampleColor += texture2D(inputImageTexture, blurCoordinates[12]).g * 2.0;
    sampleColor += texture2D(inputImageTexture, blurCoordinates[13]).g * 2.0;
    sampleColor += texture2D(inputImageTexture, blurCoordinates[14]).g * 2.0;
    sampleColor += texture2D(inputImageTexture, blurCoordinates[15]).g * 2.0;
    sampleColor += texture2D(inputImageTexture, blurCoordinates[16]).g * 2.0;
    sampleColor += texture2D(inputImageTexture, blurCoordinates[17]).g * 2.0;
    sampleColor += texture2D(inputImageTexture, blurCoordinates[18]).g * 2.0;
    sampleColor += texture2D(inputImageTexture, blurCoordinates[19]).g * 2.0;

    sampleColor = sampleColor / 48.0;

    float highPass = centralColor.g - sampleColor + 0.5;

    for(int i = 0; i < 5;i++)
    {
        highPass = hardLight(highPass);
    }
    float luminance = dot(centralColor, W);

    float alpha = pow(luminance, params);

    vec3 smoothColor = centralColor + (centralColor-vec3(highPass))*alpha*0.1;

    gl_FragColor = vec4(mix(smoothColor.rgb, max(smoothColor, centralColor), alpha), 1.0);
}
```
实时渲染的效果并不是非常理想，有些边缘地方在高度磨皮之后会出现比较明显的错误，像影子一样的东西，如下图所示：
![MagicCamera的磨皮效果](http://upload-images.jianshu.io/upload_images/2103804-4d57b7134a627548.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
从图中可以看到，高度磨皮的效果非常不自然，而且在光照比较强的情况下，边缘出现了非常明显的影子一样的透明的效果。
经过本人的研究，核心部分的高斯模糊并没有什么问题，但采样的数值并不对，采样的数值应该根据宽度和高度的变化进行相应的调整，经过调试优化，得到的GLSL代码如下：
```
precision lowp float;
uniform sampler2D inputTexture;
varying lowp vec2 textureCoordinate;

uniform int width;
uniform int height;

// 磨皮程度(由低到高: 0.5 ~ 0.99)
uniform float opacity;

void main() {
    vec3 centralColor;

    centralColor = texture2D(inputTexture, textureCoordinate).rgb;

    if(opacity < 0.01) {
        gl_FragColor = vec4(centralColor, 1.0);
    } else {
        float x_a = float(width);
        float y_a = float(height);

        float mul_x = 2.0 / x_a;
        float mul_y = 2.0 / y_a;
        vec2 blurCoordinates0 = textureCoordinate + vec2(0.0 * mul_x, -10.0 * mul_y);
        vec2 blurCoordinates2 = textureCoordinate + vec2(8.0 * mul_x, -5.0 * mul_y);
        vec2 blurCoordinates4 = textureCoordinate + vec2(8.0 * mul_x, 5.0 * mul_y);
        vec2 blurCoordinates6 = textureCoordinate + vec2(0.0 * mul_x, 10.0 * mul_y);
        vec2 blurCoordinates8 = textureCoordinate + vec2(-8.0 * mul_x, 5.0 * mul_y);
        vec2 blurCoordinates10 = textureCoordinate + vec2(-8.0 * mul_x, -5.0 * mul_y);

        mul_x = 1.8 / x_a;
        mul_y = 1.8 / y_a;
        vec2 blurCoordinates1 = textureCoordinate + vec2(5.0 * mul_x, -8.0 * mul_y);
        vec2 blurCoordinates3 = textureCoordinate + vec2(10.0 * mul_x, 0.0 * mul_y);
        vec2 blurCoordinates5 = textureCoordinate + vec2(5.0 * mul_x, 8.0 * mul_y);
        vec2 blurCoordinates7 = textureCoordinate + vec2(-5.0 * mul_x, 8.0 * mul_y);
        vec2 blurCoordinates9 = textureCoordinate + vec2(-10.0 * mul_x, 0.0 * mul_y);
        vec2 blurCoordinates11 = textureCoordinate + vec2(-5.0 * mul_x, -8.0 * mul_y);

        mul_x = 1.6 / x_a;
        mul_y = 1.6 / y_a;
        vec2 blurCoordinates12 = textureCoordinate + vec2(0.0 * mul_x,-6.0 * mul_y);
        vec2 blurCoordinates14 = textureCoordinate + vec2(-6.0 * mul_x,0.0 * mul_y);
        vec2 blurCoordinates16 = textureCoordinate + vec2(0.0 * mul_x,6.0 * mul_y);
        vec2 blurCoordinates18 = textureCoordinate + vec2(6.0 * mul_x,0.0 * mul_y);

        mul_x = 1.4 / x_a;
        mul_y = 1.4 / y_a;
        vec2 blurCoordinates13 = textureCoordinate + vec2(-4.0 * mul_x,-4.0 * mul_y);
        vec2 blurCoordinates15 = textureCoordinate + vec2(-4.0 * mul_x,4.0 * mul_y);
        vec2 blurCoordinates17 = textureCoordinate + vec2(4.0 * mul_x,4.0 * mul_y);
        vec2 blurCoordinates19 = textureCoordinate + vec2(4.0 * mul_x,-4.0 * mul_y);

        float central;
        float gaussianWeightTotal;
        float sum;
        float sampler;
        float distanceFromCentralColor;
        float gaussianWeight;

        float distanceNormalizationFactor = 3.6;

        central = texture2D(inputTexture, textureCoordinate).g;
        gaussianWeightTotal = 0.2;
        sum = central * 0.2;

        sampler = texture2D(inputTexture, blurCoordinates0).g;
        distanceFromCentralColor = min(abs(central - sampler) * distanceNormalizationFactor, 1.0);
        gaussianWeight = 0.09 * (1.0 - distanceFromCentralColor);
        gaussianWeightTotal += gaussianWeight;
        sum += sampler * gaussianWeight;

        sampler = texture2D(inputTexture, blurCoordinates1).g;
        distanceFromCentralColor = min(abs(central - sampler) * distanceNormalizationFactor, 1.0);
        gaussianWeight = 0.09 * (1.0 - distanceFromCentralColor);
        gaussianWeightTotal += gaussianWeight;
        sum += sampler * gaussianWeight;

        sampler = texture2D(inputTexture, blurCoordinates2).g;
        distanceFromCentralColor = min(abs(central - sampler) * distanceNormalizationFactor, 1.0);
        gaussianWeight = 0.09 * (1.0 - distanceFromCentralColor);
        gaussianWeightTotal += gaussianWeight;
        sum += sampler * gaussianWeight;

        sampler = texture2D(inputTexture, blurCoordinates3).g;
        distanceFromCentralColor = min(abs(central - sampler) * distanceNormalizationFactor, 1.0);
        gaussianWeight = 0.09 * (1.0 - distanceFromCentralColor);
        gaussianWeightTotal += gaussianWeight;
        sum += sampler * gaussianWeight;

        sampler = texture2D(inputTexture, blurCoordinates4).g;
        distanceFromCentralColor = min(abs(central - sampler) * distanceNormalizationFactor, 1.0);
        gaussianWeight = 0.09 * (1.0 - distanceFromCentralColor);
        gaussianWeightTotal += gaussianWeight;
        sum += sampler * gaussianWeight;

        sampler = texture2D(inputTexture, blurCoordinates5).g;
        distanceFromCentralColor = min(abs(central - sampler) * distanceNormalizationFactor, 1.0);
        gaussianWeight = 0.09 * (1.0 - distanceFromCentralColor);
        gaussianWeightTotal += gaussianWeight;
        sum += sampler * gaussianWeight;

        sampler = texture2D(inputTexture, blurCoordinates6).g;
        distanceFromCentralColor = min(abs(central - sampler) * distanceNormalizationFactor, 1.0);
        gaussianWeight = 0.09 * (1.0 - distanceFromCentralColor);
        gaussianWeightTotal += gaussianWeight;
        sum += sampler * gaussianWeight;

        sampler = texture2D(inputTexture, blurCoordinates7).g;
        distanceFromCentralColor = min(abs(central - sampler) * distanceNormalizationFactor, 1.0);
        gaussianWeight = 0.09 * (1.0 - distanceFromCentralColor);
        gaussianWeightTotal += gaussianWeight;
        sum += sampler * gaussianWeight;

        sampler = texture2D(inputTexture, blurCoordinates8).g;
        distanceFromCentralColor = min(abs(central - sampler) * distanceNormalizationFactor, 1.0);
        gaussianWeight = 0.09 * (1.0 - distanceFromCentralColor);
        gaussianWeightTotal += gaussianWeight;
        sum += sampler * gaussianWeight;

        sampler = texture2D(inputTexture, blurCoordinates9).g;
        distanceFromCentralColor = min(abs(central - sampler) * distanceNormalizationFactor, 1.0);
        gaussianWeight = 0.09 * (1.0 - distanceFromCentralColor);
        gaussianWeightTotal += gaussianWeight;
        sum += sampler * gaussianWeight;

        sampler = texture2D(inputTexture, blurCoordinates10).g;
        distanceFromCentralColor = min(abs(central - sampler) * distanceNormalizationFactor, 1.0);
        gaussianWeight = 0.09 * (1.0 - distanceFromCentralColor);
        gaussianWeightTotal += gaussianWeight;
        sum += sampler * gaussianWeight;

        sampler = texture2D(inputTexture, blurCoordinates11).g;
        distanceFromCentralColor = min(abs(central - sampler) * distanceNormalizationFactor, 1.0);
        gaussianWeight = 0.09 * (1.0 - distanceFromCentralColor);
        gaussianWeightTotal += gaussianWeight;
        sum += sampler * gaussianWeight;

        sampler = texture2D(inputTexture, blurCoordinates12).g;
        distanceFromCentralColor = min(abs(central - sampler) * distanceNormalizationFactor, 1.0);
        gaussianWeight = 0.1 * (1.0 - distanceFromCentralColor);
        gaussianWeightTotal += gaussianWeight;
        sum += sampler * gaussianWeight;

        sampler = texture2D(inputTexture, blurCoordinates13).g;
        distanceFromCentralColor = min(abs(central - sampler) * distanceNormalizationFactor, 1.0);
        gaussianWeight = 0.1 * (1.0 - distanceFromCentralColor);
        gaussianWeightTotal += gaussianWeight;
        sum += sampler * gaussianWeight;

        sampler = texture2D(inputTexture, blurCoordinates14).g;
        distanceFromCentralColor = min(abs(central - sampler) * distanceNormalizationFactor, 1.0);
        gaussianWeight = 0.1 * (1.0 - distanceFromCentralColor);
        gaussianWeightTotal += gaussianWeight;
        sum += sampler * gaussianWeight;

        sampler = texture2D(inputTexture, blurCoordinates15).g;
        distanceFromCentralColor = min(abs(central - sampler) * distanceNormalizationFactor, 1.0);
        gaussianWeight = 0.1 * (1.0 - distanceFromCentralColor);
        gaussianWeightTotal += gaussianWeight;
        sum += sampler * gaussianWeight;

        sampler = texture2D(inputTexture, blurCoordinates16).g;
        distanceFromCentralColor = min(abs(central - sampler) * distanceNormalizationFactor, 1.0);
        gaussianWeight = 0.1 * (1.0 - distanceFromCentralColor);
        gaussianWeightTotal += gaussianWeight;
        sum += sampler * gaussianWeight;

        sampler = texture2D(inputTexture, blurCoordinates17).g;
        distanceFromCentralColor = min(abs(central - sampler) * distanceNormalizationFactor, 1.0);
        gaussianWeight = 0.1 * (1.0 - distanceFromCentralColor);
        gaussianWeightTotal += gaussianWeight;
        sum += sampler * gaussianWeight;

        sampler = texture2D(inputTexture, blurCoordinates18).g;
        distanceFromCentralColor = min(abs(central - sampler) * distanceNormalizationFactor, 1.0);
        gaussianWeight = 0.1 * (1.0 - distanceFromCentralColor);
        gaussianWeightTotal += gaussianWeight;
        sum += sampler * gaussianWeight;

        sampler = texture2D(inputTexture, blurCoordinates19).g;
        distanceFromCentralColor = min(abs(central - sampler) * distanceNormalizationFactor, 1.0);
        gaussianWeight = 0.1 * (1.0 - distanceFromCentralColor);
        gaussianWeightTotal += gaussianWeight;
        sum += sampler * gaussianWeight;

        sum = sum/gaussianWeightTotal;

        sampler = centralColor.g - sum + 0.5;

        // 高反差保留
        for(int i = 0; i < 5; ++i) {
            if(sampler <= 0.5) {
                sampler = sampler * sampler * 2.0;
            } else {
                sampler = 1.0 - ((1.0 - sampler)*(1.0 - sampler) * 2.0);
            }
        }

        float aa = 1.0 + pow(sum, 0.3) * 0.09;
        vec3 smoothColor = centralColor * aa - vec3(sampler) * (aa - 1.0);
        smoothColor = clamp(smoothColor, vec3(0.0), vec3(1.0));

        smoothColor = mix(centralColor, smoothColor, pow(centralColor.g, 0.33));
        smoothColor = mix(centralColor, smoothColor, pow(centralColor.g, 0.39));

        smoothColor = mix(centralColor, smoothColor, opacity);

        gl_FragColor = vec4(pow(smoothColor, vec3(0.96)), 1.0);
    }
 }
```
磨皮等级(0% ~ 100%)可以这么计算：
 ```
/**
     * 设置磨皮程度
     * @param percent 百分比
     */
    public void setSmoothOpacity(float percent) {
        float opacity;
        if (percent <= 0) {
            opacity = 0.0f;
        } else {
            opacity = calculateOpacity(percent);
        }
        setFloat(mOpacityLoc, opacity);
    }

    /**
     * 根据百分比计算出实际的磨皮程度
     * @param percent
     * @return
     */
    private float calculateOpacity(float percent) {
        float result = 0.0f;

        // TODO 可以加入分段函数，对不同等级的磨皮进行不一样的处理
        result = (float) (1.0f - (1.0f - percent + 0.02) / 2.0f);

        return result;
    }
```
经过优化后的磨皮算法效果如下：
![优化后的磨皮效果](http://upload-images.jianshu.io/upload_images/2103804-e622efba98fd62ec.jpeg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
可以看到，磨皮的细腻程度好了很多，并且，边缘出现的影子一样的效果也相对没那么严重。对此，你可以通过锐化或者其他特效覆盖掉边缘的影子一样的效果。

具体的效果，请参考本人正在开发的相机项目：
https://github.com/CainKernel/CainCamera
由于最近这段时间有其他事情，比较忙碌，相机项目还有很多功能没有实现，并且还有很多地方需要优化的地方，以后将会不定期更新实现剩余的功能。
