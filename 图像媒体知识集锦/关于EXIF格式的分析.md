JPEG格式和标志
JPEG文件都是以十六进制的 0xFFD8 开始，以 0xFFD9 结束。在JPEG数据中，0xFF** 这样的数据被用作标志，表示JPEG信息数据段。0xFFD8表示SOI(Start Of Image 图像的开始)，以0xFFD9 表示EOI(End of Image 图像结束)。这两个特殊的标志没有附加的数据，而其他标志都在标志后面带有附加的数据。
0xFF + 标志数字(1字节) + 数据大小(2字节) + 数据(n字节)

数据大小(2字节) ： 是大端顺序表示，从高字节开始。

0xFFC1 00 0C
表示标志 0xFFC1 有0x000C(12)个字节数据，但是数据的大小"12"也包含了记录数据大小的字节，所以在0x000C后面只有10个字节的数据量。

在JPEG格式中，一些标志描述数据后，跟着的是SOS(Start Of Stream 数据流开始) 标志。在SOS标志后，就是JPEG图像流，知道EOI标志终结。格式如下所示：

![image.png](http://upload-images.jianshu.io/upload_images/2103804-734341beedb8cd7d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

Exif中使用的标志
从0xFFE0 ~ 0xFFEF 的标志是“应用程序标志”，在解码JPEG图像的时候不是必须使用的。这些标志被用在用户应用中。
Exif也使用应用程序标志来插入数据，但是Exif使用APP1(0xFFE1)标志以避免和JFIF格式冲突。
JFIF是JPEG档案交换格式，数码相机会用这个格式来存储照片。每个EXIF格式都是从下面格式开始的：

![image.png](http://upload-images.jianshu.io/upload_images/2103804-52595229018e113c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


从SOI(0xFFD8) 标志开始，因此是一个JPEG文件。后面跟着一个APP1标志。所有的Exif数据都储存在APP1数据区中。
上面的SSSS 部分表示APP1 数据大小，这个大小是包括SSSS本身的。

APP1 数据从 SSSS 后开始，第一部分是特殊数据，使用ASCII字符"Exif"和两个字节的0x00，它定义了是否使用Exif。
在APP1标志数据之后，是其他JPEG标志。

Exif数据结构采用的是Intel 的小端字节顺序方案，且包含JPEG格式的缩略图。总体上，Exif数据是从ASCII字符Exif和2个字节0x00开始，后面就是Exif的数据了。
Exif 使用TIFF格式来存储数据，TIFF 格式请参考官方文档： 
https://partners.adobe.com/asn/developer/PDFS/TN/TIFF6.pdf
如果不是adobe的partner，应该打不开上面的网址。不过本人已经备份了一份tiff6 的pdf文档在github，地址如下：
https://github.com/CainKernel/tiff6.git

下面我们介绍一下TIFF格式。

TIFF数据格式
TIFF数据格式是一种3级体系结构，从高到低一次为 文件头IFH(Image File Header)，一个或多个IFD(Image File Directory)的包含标记指针的目录和数据，一个文件中可以包含多个IFD，每个IFD对应一个图像。
```
+------------------------------------------------------------------------------+
|                           TIFF Structure                                     |
|  IFH                                                                         |
| +------------------+                                                         |
| | II/MM            |                                                         |
| +------------------+                                                         |
| | 42               |      IFD                                                |
| +------------------+    +------------------+                                 |
| | Next IFD Address |--->| IFD Entry Num    |                                 |
| +------------------+    +------------------+                                 |
|                         | IFD Entry 1      |                                 |
|                         +------------------+                                 |
|                         | IFD Entry 2      |                                 |
|                         +------------------+                                 |
|                         |                  |      IFD                        |
|                         +------------------+    +------------------+         |
|     IFD Entry           | Next IFD Address |--->| IFD Entry Num    |         |
|    +---------+           +------------------+   +------------------+         |
|    | Tag     |                                  | IFD Entry 1      |         |
|    +---------+                                  +------------------+         |
|    | Type    |                                  | IFD Entry 2      |         |
|    +---------+                                  +------------------+         |
|    | Count   |                                  |                  |         |
|    +---------+                                  +------------------+         |
|    | Offset  |--->Value                         | Next IFD Address |--->NULL |
|    +---------+                                  +------------------+         |
|                                                                              |
+------------------------------------------------------------------------------+
```
TIFF图像基本信息
基本信息包括大小、颜色模型(黑白图像、灰度图像、调色板图像、真彩色等)、每个像素的大小和单位等信息

每个图像必须包含的标签有（含压缩类型）：
```
ImageWidth 256 SHORT or LONG
ImageLength 257 SHORT or LONG
XResolution 282 RATIONAL
YResolution 283 RATIONAL
ResolutionUnit 128 SHORT 1, 2 or 3
Compression 259 SHORT 1, 2 or 32773
```
其中颜色模型由几个标签组合判断：
```
PhotometricInterpretation 262 106 SHORT 0/1/2/3
BitsPerSample 258 SHORT 4 or 8
SamplesPerPixel 277 SHORT default(1)
ColorMap 320 SHORT 3 * (2**BitsPerSample)
```
色彩模型：
TIFF基础部分定义了4中色彩模型，不讨论掩码图像和扩展色彩模型，主要有：二值图像、灰度图像、调色板图像、真彩色图像

二值图像(Bilevel Image)
如果没有BitsPerSample 则为二值图
颜色模型 PhotometricInterpretation 必须为0或1
压缩类型 Compression 1,2，32773，支持Huffman编码

灰度图(GrayScale Image)
有BitsPerSample，且PhotometricInterpretation 为0或1
压缩类型 Compression 1 或 32773，不支持Huffman编码

调色板图像(Palette-color Image)
PhotometricInterpretation 为3
必须包含ColorMap
BitsPerSample 为 4 或 8
Compression 为 1 或 32773， 不支持Huffman编码

真彩色图像(RGB Full Image)
PhotometricInterpretation 为 2
必须包含SamplesPerPixel 波段数信息，值为3
BitsPerSample 必须为 8, 8, 8
没有ColorMap
Compression 为 1 或 32773，不支持Huffman编码

其他模型
TIFF的扩展特性中还包含CMYK 和 YCbCr等其他色彩模型
详细信息请参考TIFF6.0规范。

条带和分片
其中数据组织方式有条带和分片两种，默认为条带方式存储

条带图像必要的标签
```
RowsPerStrip 278 SHORT or LONG
StripOffsets 273 SHORT or LONG
StripByteCounts 279 SHORT or LONG
```
RowPerStrip 缺省值为 2^32 - 1，图像将只有一个条带
如果条带数据不足，末尾并不需要数据填充

分片图像必要的标签：
```
TileWidth 322 SHORT or LONG
TileLength 323 SHORT or LONG
TileOffsets 324 SHORT or LONG
TileByteCounts 325 SHORT or LONG
```
每个片的大小必须是16的倍数，每个片的长和宽可以不同
如果片数据不足的话，需要填充（填充数据没有要求）

位平面
位平面由PlanarConfiguration 标签确定，缺省时为1表示按像素组织(RGBRGB)
当PlanarConfiguration 为2 时，为按通道存储（RRGGBB）
当只有1通道（SamplesPerPixel）时，位平面信息可以忽略

压缩方式
压缩方式由Compression 标签确定

基本算法：
```
1 = No compression
2 = CCITT modified Huffman RLE
32773 = PackBits compression, aka Macintosh RLE
```
扩展特性:
```
3 = CCITT Group 3 fax encoding
4 = CCITT Group 4 fax encoding
5 = LZW
6 = JPEG ('old-style' JPEG, later overriden in Technote2)
7 = JPEG ('new-style' JPEG)
8 = Deflate ('Adobe-style')
```
常见标签:
```
ImageDescription 270 ASCII

CellLength 265 SHORT
CellWidth 264 SHORT

Software 305 ASCII
DateTime 306 ASCII
Artist 315 ASCII
HostComputer 316 ASCII

Copyright 33432 ASCII
```
空闲空间
当创建或更新TIFF文件时，文件中会有一些空闲的空间(类似malloc、free导致的内存碎片)
TIFF规范中有FreeByteCounts 和 FreeOffsets 两个标签用于记录文件中的空闲空间

但是由于不同标签之间的弱引用关系，FreeByteCounts 和 FreeOffsets 记录的信息可能并不可靠，因此不能完全依赖这个信息。


忽略特性
文件中包含多个图像(多IFD)
扩展的压缩算法
扩展的色彩模型
多通道数大于4
文件实例

TIFF6.0文件规范中的二值图像片段：
```
Header:
 0000 Byte Order 4D4D
 0002 42 002A
 0004 1st IFD offset 00000014
 
IFD:
 0014 Number of Directory Entries 000C
 0016 NewSubfileType 00FE 0004 00000001 000000000022 ImageWidth 0100 0004 00000001 000007D0
 002E ImageLength 0101 0004 00000001 00000BB8
 003A Compression 0103 0003 00000001 8005 00000046 PhotometricInterpretation 0106 0003 00000001 0001 00000052 StripOffsets 0111 0004 000000BC 000000B6
 005E RowsPerStrip 0116 0004 00000001 00000010006A StripByteCounts 0117 0003 000000BC 000003A6
 0076 XResolution 011A 0005 00000001 000006960082 YResolution 011B 0005 00000001 0000069E
 008E Software 0131 0002 0000000E 000006A6
 009A DateTime 0132 0002 00000014 000006B6
 00A6 Next IFD offset 00000000

 Values longer than 4 bytes:
 00B6 StripOffsets Offset0, Offset1, ... Offset187
 03A6 StripByteCounts Count0, Count1, ... Count187
 0696 XResolution 0000012C 00000001069E YResolution 0000012C 0000000106A6 Software “PageMaker 4.0”
 06B6 DateTime “1988:02:18 13:59:59”

 Image Data:
 00000700 Compressed data for strip 10
 xxxxxxxx Compressed data for strip 179
 xxxxxxxx Compressed data for strip 53
 xxxxxxxx Compressed data for strip 160
```
介绍了TIFF的基本格式后，我们来看看Exif的数据结构， Exif 的数据结构如下：

![image.png](http://upload-images.jianshu.io/upload_images/2103804-56f3040a563e7355.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
TIFF头格式(IFH, Image File Header)
对于II (0x4949)格式，字节顺序总是从低字节到高字节排序的，也就是小端存储。
对于MM(0x4D4D)格式，字节顺序总是从高字节到低字节排序的，也就是大端存储。
虽然JPEG值采用大端字节顺序存储，但Exif允许采用两种方式存储
接下来是42(0x2A)固定值，小端存储的话，则是 0x49492A00，大端存储的话，0x4D4D002A
TIFF头格式的最后4个字节表示第一个IFD的偏移量。通常是0x00000008，因为TIFF 的头格式IFH占用了8个字节。
故TIFF头格式的IFH格式为：
0x4949 2A00 0800 0000 (小端存储) 或者 0x4D4D 002A 0000 0008(大端存储)

文中的数据指针Offset最大对应4个字节，因此一个文件不能超过4GB。如果单张图片要超过4G，则需要使用BigTiff格式，请自行查阅相关资料，这里不做介绍。

TIFF 图像文件目录(IFD, Image File Directory)
IFD 包含图像信息数据。开始两个字节(EEEE)表示这个IFD所包含的目录实体数量。
然后紧跟着实体对象(每个实体12个字节)。
最后一个目录实体后面有一个4字节大小的数据，表示偏移量，如果是0x00000000，则表示这个IFD是最后一个IFD。

数据格式
数据格式(上表中的FFFF)如下表所定义的一样。

![image.png](http://upload-images.jianshu.io/upload_images/2103804-4d610d7134d4bcb9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
rational 表示一个分数，它包含两个signed/unsigned long integer值且第一个为分子，第二个为分母

你可以用组成元素的字节数的值(Bytes/component)乘以存储在NNNNNNNN区域中的组成元素的数量得到总长度。
如果这个总长度小于4个字节，那么DDDDDDDD中的是这个标签(Tag)的值，如果总长度大于等于4个字节，DDDDDDDDD是数据存储地址的偏移量

IFD数据结构
在Exif格式中，第一个IFD 是IFD0(主图像的IFD)，连接IFD1(缩略图IFD)后IFD链终止。带式IFD0/IFD1 不包含想快门速度，焦距等任何数码相机信息。
IFD0总是包含特殊的标签(Tag)Exif的偏移量(0x8769)，它说明Exif SubIFD 的偏移量，SubIFD 包含数码相机的信息。

Exif格式的扩展方案(Exif2.1/DCF)中，Exif SubIFD 包含了特殊标签 —— Exif互用偏移量(Exif Interoperability Offset)(0xA005),它指向互用的IFD。
在DCF(数码相机格式)规范中，这个标签是必须且子IFD 和 IFD1 都可以使用互用的IFD。通常只有主图像带有这个标签。

一些数码相机使用IFD数据格式来表示制造商数据 —— 制造商特殊的神秘数字区。

IFD数据结构例子：
假设下面数据是TIFF数据的第一部分：
0x0000 : 4949 2A00 0800 0000 0200 1A01 0500 0100
0x0010 : 0000 2600 0000 6987 0400 0100 0000 1102
0x0020 : 0000 4000 0000 4800 0000 0100 0000

开头两个字节为 II， 小端存储顺序。
地址0x0004 ~ 0x0007 是 0x0800 0000, IFD 从地址 0x0008 开始
地址0x0008 ~ 0x0009 是 0x0200，实际值为2，表示 IFD0 有两个目录实体
地址0x000a ~ 0x000b 是 0x1A01，表示这是一个水平分辨率(XResolution)(0x011A) 标签，它包含了图像的水平分辨率
地址0x000c ~ 0x000d 是 0x0500，这个值表示的格式是无符号分数(0x0005，对应上面数据格式表格中的5)，无符号分数的尺寸是8个字节。
地址0x000e ~ 0x0011 是 0x01000000，组成的元素数量是1，无符号分数的尺寸是8个字节，故所有数据总长度为1 x 8 = 8 字节。
地址0x0012 ~ 0x0015 是 0x26000000，水平分辨率（XResolution）的数据存储在0x0026位置
地址0x0016 ~ 0x0017 是 0x6987，下一个标签是Exif偏移(ExifOffset)(0x8769)。它的值是Exif SubIFD 的偏移量
地址0x0018 ~ 0x0019 是 0x0400，数据格式是0x0004，对于数据格式表中的无符号长整形unsigned long
地址0x001a ~ 0x001d 是 0x01000000，组成元素数量是1，因此数据总长度为4字节
地址0x001e ~ 0x0021 是 0x11020000，表示Exif SubIFD 从 0x0211开始
地址0x0022 ~ 0x0025 是 0x40000000，表示下一个IFD从0x0040开始
// 水平分辨率数据存储的位置
地址0x0026 ~ 0x0029 是 0x48000000，分子是72(0x0048)
地址0x002a ~ 0x002d 是 0x01000000，分母是1(0x0001)，因此水平分辨率(XResolution)是72/1

缩略图
Exif格式包含图像的缩略图。缩略图通常在IFD1 后面，共有3中格式的缩略图： JPEG格式(JPEG 使用YCbCr)，RGB TIFF格式，YCbCr TIFF 格式。
Eif2.1 或者更高版本是推荐使用JPEG格式和160x120 的缩略图。而DCF规范则必须使用JPEG格式和160x120的缩略图

JPEG格式缩略图
如果IFD1中压缩(Compression)(0x0103)标签的值为6，那么缩略图是JPEG格式。大多数的Exif图像使用JPEG格式的缩略图。
在这些图像中，你可以在IFD1中JpegIFOffset(0x0201)标签中，以及JpegIFByteCount(0x0202)标签中分别获得缩略图的偏移量和大小。
其数据格式为普通的JPEG格式，从0xFFD8开始，到0xFFD9结束。

TIFF格式缩略图
如果IFD1 中压缩(Compression)(0x0103)标签值为1， 那么缩略图是非压缩的（TIFF格式）。
缩略图的起点数据在StripOffset(0x0111)标签，缩略图大小则是StripByeCounts（0x0117）标签的和

如果采用了TIFF格式缩略图，IFD1 标签中的PhotometricInterpretation(0x0106)的值为2，则缩略图采用RGB格式。

如果压缩标签的值为2， 则缩略图采用YCbCr格式。
