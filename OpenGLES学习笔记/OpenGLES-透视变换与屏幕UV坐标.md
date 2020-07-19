一、从需求说起 
本人在做3D贴纸的时候，遇到这样的一个需求，在3D贴纸需要和图像进行混合。做远小近大的3D效果，需要将二维的贴纸经过透视变换绘制到屏幕上，如果要添加混合效果，则必须知道变换后的坐标，如果坐标对不上，则会导致混合后的贴纸绘制到屏幕上可能会出现错乱、重叠的情况，因此如何计算重叠部分的准确坐标是实现混合的关键。为此，重新拾起已经被遗忘了的数学知识，将透视变换过程重新推导一遍，以便更好地理解透视变换以及如何转换到屏幕的空间坐标的。

二、透视变换推导
1、透视投影公式
下图给出了一个空间点(x, y, z)到一般的投影参考点(xv, yv, zv)的投影路径。该投影线与观察平面交于坐标位置(xp, yp, zvp), 其中zvp是观察平面上选择的位于z轴上的点。
![投影点.png](http://upload-images.jianshu.io/upload_images/2103804-4fb58004f633ecb5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
由此我们可以计算得到坐标位置的参数方程如下：
![参数方程.png](http://upload-images.jianshu.io/upload_images/2103804-5939cf43500430f4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
坐标(x', y', z') 代表沿投影线的任意一点。当 u = 0 时，位置为(x, y, z)，当 u = 1 时位置为(xv, yv, zv)。
在观察平面上，由 z' = zvp，可以求解z' 的方程得到沿投影线该位置的u参数如下：
![u参数.png](http://upload-images.jianshu.io/upload_images/2103804-ef95234065aa8f09.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
将u代入方程，可得一般的投影变换公式：
![投影变换公式.png](http://upload-images.jianshu.io/upload_images/2103804-6907dac71fce023f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
当投影参考点在Z轴上时，xv = yv = 0，此时有：
![投影变换公式.png](http://upload-images.jianshu.io/upload_images/2103804-e11e93dce40221bf.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
2、透视投影的灭点
一组投影平行线汇聚到在一起的点称为灭点(vanishing point)。通过投影平面的方向可以控制灭点的数量，透视投影可以分为一点、两点或三点投影。
3、投影的观察体
通过观察平面上指定一个矩形裁剪窗口可以得到一个观察体。但观察体的边界面不再平行。透视投影观察体常称为视锥体(pyramid of vision)，与眼睛或相机的视觉圆锥体相近。添加垂直于z轴（且与观察平面平行）的近、远平面后，切掉了无限、透视投影观察体的一部分后，得到一个棱台(frustum)观察体。观察平面位于远近平面与投影参考点之间，在OpenGLES中，近平面就是观察平面，也就是裁剪窗口。
4、透视投影变换矩阵：
我们不能使用等式的x和y坐标系数来形成透视矩阵的元素，因为系数的分母是z的函数。但可以通过三维齐次坐标表示：
![齐次坐标.png](http://upload-images.jianshu.io/upload_images/2103804-2cb59e0061c9d81e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
其中，齐次参数h为：
![齐次参数.png](http://upload-images.jianshu.io/upload_images/2103804-353653dbb9882dc9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
因此可以求得x和y的值为：
![齐次坐标值.png](http://upload-images.jianshu.io/upload_images/2103804-b0366d48081079d7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

因此，我们可以建立一个变换矩阵将一个空间位置转为齐次坐标位置，使得矩阵仅包含透视参数而不包含坐标值。然后观察坐标系的透视投影变换分两步实现。先用透视变换矩阵计算齐次坐标：
![透视矩阵.png](http://upload-images.jianshu.io/upload_images/2103804-fd50246423b529b9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
Ph 是齐次点(xh, yh, zh, h) 的列矩阵表示， 而P是坐标(x, y, z, 1)的列矩阵表示，在实际当中，透视矩阵需要跟观察矩阵(ViewMatrix)合并，然后将组合矩阵应用于场景的世界坐标描述以生成齐次坐标。
为了防止z轴除以齐次参数h后出现扭曲，我们需要通过为z变换设定矩阵元素从而对透视投影zp的坐标进行规范化。有多种方法选择矩阵元素。下面是一种可能形成透视投影矩阵的方法：
![投影矩阵.png](http://upload-images.jianshu.io/upload_images/2103804-ab802708545a045a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
参数sz 和tz 是在对z坐标投影值规范化中的比例和平移因子。sz 和tz的特定值依赖于选择的规范化范围

5、对称的视锥体
从投影参考点到裁剪窗口中心平穿过观察体的线就会说透视投影棱台的中心线。棱台中心线与观察平面相交于坐标(xv, yv, zvp)位置，用窗口尺寸表示裁剪窗口的对角位置，可得：
![对角位置.png](http://upload-images.jianshu.io/upload_images/2103804-af8896e0f5df7548.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
整体如下图所示：
![对称视锥体.png](http://upload-images.jianshu.io/upload_images/2103804-1e6af6585c48f8e6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
或者，可以通过使用逼近相机镜头特性的参数来指定透视投影。视锥可以看做是一个视场角(field-of-view angle)，他是照相机镜头的尺寸度量。如下图所示：
![视锥体.png](http://upload-images.jianshu.io/upload_images/2103804-02395d55e0602089.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
由上图可以得到：
![角度与裁剪窗口高度的关系.png](http://upload-images.jianshu.io/upload_images/2103804-d14d9bd70e1221d5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
可以求得裁剪窗口高度：
![窗口高度.png](http://upload-images.jianshu.io/upload_images/2103804-dd98552ec2bfb079.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
上面的投影矩阵：
![投影矩阵.png](http://upload-images.jianshu.io/upload_images/2103804-ab802708545a045a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
中zv - zvp的对角线元素可以用下面表达式来表示：
![矩阵对角线元素.png](http://upload-images.jianshu.io/upload_images/2103804-37986ba11d85db6d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
其中aspect 是长宽比。

6、斜透视投影棱台
如果透视投影观察体中心线并不垂直于观察平面，则得到一个斜棱台(oblique frustum)。
为了方便计算，将投影参考点(xv, yv, zv)  = (0, 0, 0)，可得到错切变换矩阵的元素：
![错切变换矩阵.png](http://upload-images.jianshu.io/upload_images/2103804-f258b0bf806ae01c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
如果将观察平面放在近裁剪平面处，则透视投影矩阵可以进一步简化。将裁剪窗口中心移到观察平面坐标位置(0, 0)处，需要选择的错切参数值满足：
![错切参数条件.png](http://upload-images.jianshu.io/upload_images/2103804-30fc4f618fbb5a2d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

当投影参考点位于观察坐标原点切近裁剪平面与观察平面重合时，透视投影矩阵可以简化为：
![透视投影矩阵简化.png](http://upload-images.jianshu.io/upload_images/2103804-a5667ff789c037e7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

将透视矩阵和错切矩阵综合，就可以得到下面的将场景坐标位置转换成齐次正交坐标的斜透视投影矩阵。该变换的投影参考点是观察坐标原点，而近裁剪平面是观察平面。
![斜透视投影矩阵.png](http://upload-images.jianshu.io/upload_images/2103804-880d96f1186e58b7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

7、规范化透视投影变换坐标
矩阵将观察坐标系中的对角位置变换到透视投影齐次坐标。使用齐次参数h 除齐次坐标，可得实际的正交投影坐标。该透视投影将棱台观察体中所有点变换成矩形平行管道观察体中的位置，变换过程的最后一步是将该平行管道映射到规范化观察体(normalized view volume)中，其实也就是设备标准化坐标系(NDC)中的坐标。
转换过程遵循评语投影的规范化过程。从棱台观察体变换而来的矩形平行管道映射到对称左手参考系的规范化立方体中。完成规范化的缩放矩阵是：
![缩放矩阵.png](http://upload-images.jianshu.io/upload_images/2103804-7f1ed3357f5a8b3f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
将透视矩阵与缩放矩阵综合得到规范化矩阵：

![透视投影规范化矩阵.png](http://upload-images.jianshu.io/upload_images/2103804-ebe0d57ec6e41eab.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

将该规范化透视矩阵进行一般化，可以得到以下形式：
![透视投影规范化变换.png](http://upload-images.jianshu.io/upload_images/2103804-aa26c40bf4fe3362.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

如果透视投影观察体一开始就指定为对称棱台，则可用裁剪窗口的视场角和尺寸来表达规范化透视变换的元素。将投影参考点位于原点且观察平面在近裁剪平面位置时，可以得到：

![对称棱台规范化的透视投影.png](http://upload-images.jianshu.io/upload_images/2103804-93a59fdded6e63de.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

三、Android OpenGLES 的三维观察函数
1、观察变换函数
在Android中，你可以使用Matrix.setLookAtM方法来设置观察变换矩阵，方法原型如下：
```
public static void setLookAtM(float[] rm, int rmOffset,
            float eyeX, float eyeY, float eyeZ,
            float centerX, float centerY, float centerZ, float upX, float upY,
            float upZ);
```
在OpenGL中的方法跟Android中的OpenGLES 不一样。OpenGL建模观察模式用下列语句来设定的：
```
glMatrixMode(GL_MODELVIEW);
```
观察参数用GLU函数指定，该函数如下：
```
gluLookAt(x0, y0, z0, xref, yref, zref, Vx, Vy, Vz);
```
其中(x0, y0, z0) 跟Android中的方法setLookAtM的(eyeX, eyeY, eyeZ) 点均表示观察参考点在世界坐标系的位置。而(xref, yref,zref) 和 (centerX, centerY, centerZ) 表示参考点的坐标，(Vx, Vy, Vz) 和 (upX, upY, upZ) 表示向上向量。默认情况下 gluLookAt的参数是 P0 = (0, 0, 0), Pref = (0,0, -1), V = (0, 1, 0);

2、对称透视投影棱台
OpenGL中对称透视投影棱台观察体用gluPerspective表示，原型如下：
```
gluPerspective(theta, aspect, dnear, dfar);
```
在Android的OpenGLES 中也存在类似的方法：
```
public static void perspectiveM(float[] m, int offset,
          float fovy, float aspect, float zNear, float zFar);
```
其中 theta 和 fovy 表示视场角，0~180度可选。aspect表示长宽比。far 和near 表示观察参考点到远近裁剪平面的距离。

3、通用透视投影函数
在OpenGL中，通用棱台一般使用glFrustum函数来实现:
```
glFrustum(xwmin, xwmax, ywmin, ywmax, dnear, dfar);
```
当选择 xwmin = - xwmax 且 ywmin = - ywmax 时，表示的是一个对称棱台。
在Android的OpenGLES中，可以使用类似的方法：
```
public static void frustumM(float[] m, int offset,
            float left, float right, float bottom, float top,
            float near, float far);
```

四、空间坐标投影到观察平面上的UV坐标计算。
好了，至此，我们讨论了透视投影矩阵变换的整个过程，以及OpenGL 和OpenGLES 设置投影矩阵的方法。那么，如何求得空间坐标经过透视投影矩阵的变换后，得到屏幕的UV坐标呢？
假设在棱台(frustum)中的一点的坐标为(x, y, z)，经过投影变换后的坐标为(x', y', z')。则我们可以得到以下计算公式：
![透视投影换算.png](http://upload-images.jianshu.io/upload_images/2103804-9d0a910c58c1bcbf.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
新得到的坐标(x', y', z', w) 是经过透视投影变换后的坐标，该坐标就是前面第7小节规范化透视投影变换坐标中讨论的透视投影齐次坐标。由于OpenGL的观察平面就是近裁剪平面，因此该坐标就是坐标(x,y,z)经过透视变换后在近裁剪平面上的三维坐标。其中w记录了深度信息。当设置为对称棱台投影后，矩阵的实际值就是前面计算得到的对称棱台规范化的透视投影矩阵:
![对称棱台规范化的透视投影矩阵.png](http://upload-images.jianshu.io/upload_images/2103804-93a59fdded6e63de.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

那么换算得到的新坐标是投影齐次坐标，那么接下来如何转换成屏幕UV坐标呢？我们只需要将得到的新坐标进行归一化为NDC坐标系，然后将其转成UV坐标的表达形式即可。
1、转成NDC坐标系
这里将所有坐标均除以w值，即可得到NDC坐标，此时的w将被规整为-1 ~ 1 之间。
![NDC坐标.png](http://upload-images.jianshu.io/upload_images/2103804-19669ff6a99ab03c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
2、换算成屏幕UV坐标
根据前面计算得到的NDC坐标，可以换算成屏幕的UV坐标。由于NDC坐标的范围时-1 ~ 1 的立方体，而屏幕UV坐标则是0~1的平面。如何计算？ 其实很简单，只需要从NDC坐标中缩小到原来的一般然后平移到UV屏幕中间(0.5, 0.5)，即可求得UV坐标。
![UV坐标.png](http://upload-images.jianshu.io/upload_images/2103804-a2a36ab66331adb7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

五、OpenGLES 中计算透视变换到裁剪平面上的uv坐标
看到前面的一大堆计算，估计很多人都会头晕眼花的。其实你完全可以不了解前面的推导过程，在OpenGL 和OpenGLES 中经过透视投影矩阵变换后的屏幕uv坐标甚至是一件非常简单的事情，变换过程如下：
```
vec2 calculateUVPosition(vec3 modelPosition, mat4 mvpMatrix) {
    vec4 tmp = vec4(modelPosition, 1.0);
    tmp = mvpMatrix * tmp; // gl_Position
    tmp /= tmp.w; // 经过这个步骤，tmp就是归一化标准坐标了.
    tmp = tmp * 0.5 + vec4(0.5f, 0.5f, 0.5f, 0.5f); // NDC坐标
    return vec2(tmp.x, tmp.y); // 屏幕的UV坐标
}
```
通过将棱台(frustum)中的坐标，与总变换mvpMatrix的乘积，得到一个vec4的变量，该变量就是gl_Position的值。gl_Position代表着总变换后的空间位置。也就是我们所说的齐次坐标。然后将其转成NDC坐标，重新构建一个新的屏幕UV坐标即可。

六、最后说一下3D贴纸需求实现思路
实现3D贴纸并将贴纸与图像按照自定义的方式进行混合的整体过程实现如下：
1、根据人脸关键点坐标反推算出正脸的坐标(考虑歪头、转脸的情况，如果人脸关键点检测得到的是三维空间点另外计算)
2、根据正脸的坐标计算出贴纸处于屏幕空间四个顶点的UV坐标
3、根据计算出的贴纸处于屏幕空间的UV坐标，反推算出贴纸实际所处平面的空间坐标(假设未做歪头等动作时的坐标)
4、根据得到的贴纸实际空间的坐标，经过平移、旋转后，计算出投影变换后的综合矩阵。
5、在glsl中计算总最终的位置：
```
vertex shader中：
varying vec4 vPosition;
...
vPosition = mvpMatrix * position;
 gl_Position = vPosition;
```
6、将得到的最终的空间坐标vPosition传递给Fagment Shader，将坐标转换成屏幕的uv坐标
```
// 归一化处理，转成齐次坐标
vec4 position = vPosition / vPosition.w;
// 转换成屏幕uv坐标
position = position * 0.5 + vec4(0.5, 0.5, 0.5, 0.5);
```
7、获取图像与贴纸重叠部分的texture进行混合
```
// 重叠部分的texture
vec4 source = texture2D(inputTexture,  position.xy);
// 贴纸的texture
vec4 mipmap = texture2D(sTexture, texCoord);
// 混合
gl_FragColor = blendColor(mipmap, source);
```
