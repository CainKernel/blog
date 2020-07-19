#### RGB转成HSB/HSV
##### HSB/HSV色彩模式
关于HSB/HSV色彩模式的定义，可以参考百度百科：
https://baike.baidu.com/item/HSV/547122
https://baike.baidu.com/item/hsb/10338413?fr=aladdin

##### HSB/HSV和RGB相互转换的OpenGLES实现
rgb转hsb/hsv:
```
vec3 convertRgbToHsv(vec3 c) {
	vec4 K = vec4(0.0, -1.0 / 3.0, 2.0 / 3.0, -1.0); 
	vec4 p = mix(vec4(c.bg, K.wz), vec4(c.gb, K.xy), step(c.b, c.g)); 
	vec4 q = mix(vec4(p.xyw, c.r), vec4(c.r, p.yzx), step(p.x, c.r)); 
	
	float d = q.x - min(q.w, q.y); 
	float e = 1.0e-10; 
    return vec3(abs(q.z + (q.w - q.y) / (6.0 * d + e)), d / (q.x + e), q.x); 
} 
```
hsb/hsv转rgb:
```
vec3 convertHsvToRgb(vec3 c) { 
	vec4 K = vec4(1.0, 2.0 / 3.0, 1.0 / 3.0, 3.0); 
	vec3 p = abs(fract(c.xxx + K.xyz) * 6.0 - K.www); 
	return c.z * mix(K.xxx, clamp(p - K.xxx, 0.0, 1.0), c.y); 
} 
```
