在编写着色器总会使用到内建函数的时候，这里主要是写自己用到的一些函数进行汇总，不定期更新。更多资源可以访问以下网站：
http://www.shaderific.com/glsl-functions

由于glsl是基于C语言的，所以很多时候，一些内建函数跟C语言的数学函数是一致的，更多时候，我们基本上可以从matlab中找到相关的函数，甚至函数名称基本一致。

dot : 计算两个向量的点积
```
函数接口: dot(x, y)
x, y : 输入变量，必须是向量
return : 点积结果
描述 : 对于向量a, b，返回的结果是 y = ∑(ai * bi) 乘积之和。关于点积的数学知识请参考《线性代数》等相关书籍
```
clamp : 规整输入值
```
函数接口: clamp(x, min, max)
x : 输入值
min : 最小值
max : 最大值
return : 根据输入的x，返回介于 min 与 max 之间的值。
描述 : 当 x < min时，返回min，当 x > max 时，返回 max
```
mix : 线性插值
```
函数接口mix(x, y, level);
x, y : 输入值
level :  插值系数
return : 返回插值结果
描述 : dest = x * (1 - level) + y * level;
```
