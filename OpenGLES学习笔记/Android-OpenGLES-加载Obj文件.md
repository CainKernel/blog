### Obj文件
 Obj文件是WaveFront公司为它的一套基于工作站的3D 建模和动画软件“Advanced Visualize”开发的一种文件格式，这种格式同样也可以通过Maya读写。
Obj是一种文本文件。
Obj3.0格式支持多边形(Polygon)、直线(Lines)、表面(Surfaces) 和自由形态曲线(Free-form Curves)。直线和多边形通过点来描述，曲线和表面则根据控制点和依附于曲线类型的额外信息来定义，这些信息支持规则和不规则曲线，包括贝塞尔曲线、B样条、基础(Cardinal/Catmull-Rom样条)、泰勒方程等

### Obj文件特点
1) 是一种3D模型文件，因此不包含动画、材质特性、贴图路径、动力学、粒子等信息。
2) 主要支持多边形模型。虽然Obj格式也支持曲线、表面、点组材质，但Maya导出的Obj文件并不包括这些信息。
3) Obj文件支持三个点以上的面。
4）Obj文件支持法线和贴图坐标

### Obj文件基本结构
Obj文件不需要任何种文件头，尽管经常使用几行文件信息的注释作为文件的开头
注释: 以 "#" 开头，空格和空行可以随意加到文件中
关键字：说明这一行是什么样的数据。多行可以逻辑地连接在一起表示一行，方法是在每一行最后添加一个连接符(\)，但后面不能出现空格或tab。
关键字描述：
1) 顶点数据(Vertex data):
v 几何顶点 geometric vertices
vt 贴图坐标点 texture vertices
vn 法线 vertex normal
vp 参数空格顶点 parameter space vertices

2) 自由形态曲线(Free-form curve)/表面属性(surface attributes):
deg 度 degree
bmat 基础矩阵 basis matrix
step 步尺寸 step size
cstype 曲线或表面类型 curve or surface type

3) 元素(Elements)：
p 点 point
i 先 line
f 面 face
curv 曲线 curve
curv2 2D曲线 2D curve
surf 表面 surface

4) 自由形态曲线(Free-form curve)/表面主题陈述(surface body statements)：
parm 参数值 parameter values
trim 外部修剪循环 outer trimming loop
hole 内部修剪循环 Inner trimming loop
scrv 特殊曲线 special curve
sp 特殊点 special point
end 结束陈述 end statement

5) 自由形态表面之间的连接(Connectivity between free-form surfaces)：
con 连接 connect

6) 成组(Grouping)：
g 组名称 group name
s 光滑组 smoothing group
mg 合并组 merging group
o 对象名称 object name

7) 显示(Display)/渲染属性(render attributes)：
bevel 导角插值 bevel interpolation
c_interp 颜色插值 color interpolation
lod 细节层次 Level of detail
usemtl 材质名称 material name
mtlib 材质库 material library
shadow_obj 投影阴影 shadow casting
trace_obj 光线跟踪 ray tracing
ctech 曲线近似技术 curve approximation technique
stech 表面近似技术 surface approximation technique


