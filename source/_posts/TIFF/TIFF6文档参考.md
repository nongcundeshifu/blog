---
title: TIFF6文档参考
date: 2021-10-13 11:13:43
tags:
  - TIFF
categories:
  - TIFF
---

本文主要是基于TIFF 6.0官方规范文档整理出来的常用信息来供大家参考。

<!-- more -->

TIFF规范版本：TIFF 6.0

参考资料：

- [TIFF 6.0文档](https://web.archive.org/web/20180810205359/https://www.adobe.io/content/udp/en/open/standards/TIFF/_jcr_content/contentbody/download/file.res/TIFF6.pdf)
- [tiff标签含义查询网站](https://www.awaresystems.be/imaging/tiff/tifftags.html)
- 针对扩展的压缩方式=7和deflate压缩的扩展资料：[TIFF 规范补充 2](https://www.awaresystems.be/imaging/tiff/specification/TIFFphotoshop.pdf)

## 说明

TIFF规范分为两部分。第1部分描述了基线TIFF。基线TIFF是TIFF的核心，这是所有主流TIFF开发人员在其产品中应该支持的基本要素

### 遵守

- `Is and shall`：指要强制要求
- `Should`：表示一个建议
- `May`：表示一个选项，可选的
- `not recommended for general data interchange`：指定为“不推荐进行一般数据交换”的功能被视为对基线TIFF的扩展。使用这些功能的文件应指定为“扩展TIFF6.0”文件，并应记录所使用的特定扩展名。不需要一个基线TIFF6.0读取器来支持任何扩展。也就是如果是这种的标识，表示基本的TIFF解析器不需要支持。

> TIFF解析器应该要支持解析基础的字段，而对于可选或者扩展字段，TIFF不应该要求文件一定存在这些字段，TIFF解析器应该能够安全的跳过这些字段。
> 不要假设图像数据是顺序拼接的，比如图像的条带数据，可能是分散或者乱序排列的。所以，不要做任何假设。
> 对于基线字段，是必须要处理的，且如果这些基线字段不存在，则读取器解析时必须设置该字段的值为默认值。（对于特定的TIFF文件，有些字段不是必需的）

### 私有字段和值

枚举值超过32768的为私有字段， 65000-65535为可变的任意字段。

## TIFF结构体

可以参考:[TIFF文件格式了解](/2021/10/05/TIFF/TIFF文件格式了解/)所介绍的部分进行相互印证

### Image File Header

头文件

- 0-1字节表示字节顺序："II"：小字节顺序 18761，"MM"：大字节顺序 19789
- 2-3：如果是指为42则为标准tiff，如果为43则为bigtiff
- 4-7：表示 第一个IFD的偏移量（以字节为单位）。文档中，术语字节偏移量总是用于指关于TIFF文件开头的位置。该文件的第一个字节的偏移量为0。

### Image File Directory-图像文件目录

这个就是用来存放图像文件信息的一个结构体。如果只有一个图像的TIFF，大概是只有一个IFD

一个IFD包括：

1. 2个字节：表示该IFD中DE数量（Directory Entry Count）
2. 第一个DE数据（每个DE固定12字节）
3. 第二个DE数据。。。以此类推
4. 第count个DE数据
5. 4个字节：下一个IFD数据结构的起始偏移量的位置，如果没有下一个，则需要补填上4个字节的0

一个TIFF可以定义多个IFD，而IFD的潜在作用就是为了描述相关图像的。

#### IFD Entry（DE）

一个DE存的是一个IFD中的一个字段信息，比如imageWidth，imageLength信息都是以DE来存放。

- 最开始前两个字节是表示：Tag（知道为什么叫标签图像文件格式了吧），数据类型是word（word就是表示2字节数据而已，其实应该是ushort）。标识该字段的标签（标签都是定义好的，是一个TIFF规定的枚举，比如imageWidth的标签的十进制是：256，所以这个DE中的0-1个字节存的就是256这个数据。
- 第3、4个字节表示：Type，标识该DE字段的数据类型，其值为1-12，表示不同的数据类型，以此知道该字段所占据的字节数是多少以及如何解析该数据。
- 再后面的4个字节，表示Length，他表示这个DE的数据有多少个Type，数据类型是：	Unsigned Long。比如这里是4，type是ushort，那么表示这个DE有4个type为ushort的数据，那么这个DE数据你可以解析为一个4个ushort的数组。
- 最后4个字节表示：value Offset 数据类型是：Unsigned Long。根据type（字节数） * length的大小，如果超出4字节，那么这里存放的就是数据所在的偏移量，如果小于等于4字节，那么存放的就是数据。

> `Is and shall`：IFD中的DE必须按标记（tag）的值从小到大进行升序排序
> 将来可能会添加其他TIFF字段类型。实现的读取器应该跳过包含意外字段类型的字段
> 注意：所有已定义字段（tag）都有一个关联的length，即使只有一个值，其实这些字段都是一个一维数组（所以length就是数组length？）。如果要在字段中存储复杂的数据结构，那么请使用未定义（UNDEFINED类型）的字段，并将length设置为所需数据类型的字节数。

## 常用字段：tag

### 颜色

光度测量解释-其实叫做图像的颜色空间更合适，比如灰度，黑边，rgb这种颜色空间。
PhotometricInterpretation
tag：262
type：short

双层图像（Bilevel Images）包含黑白两种颜色。TIFF允许应用程序以白零或黑零（标识将0解释为黑还是将1解释为黑）格式写入双层数据。记录这些信息的字段称为光度测量解释。

rgb图像，则PhotometricInterpretation = 2

### 压缩

Compression
Tag = 259 (103.H)
Type = SHORT

- 1：未压缩
- 2：ccitt组31-维修改霍夫曼运行长度编码
- 32773：包位压缩，一个简单的面向字节的运行长度方案。有关详细信息，请参阅“软件包位”部分。数据压缩仅适用于光栅图像数据。所有其他TIFF字段都不受影响

> 基线TIFF读取器必须处理所有三种压缩方案

### 图像行和列

图像中的行数（有时描述为扫描行），居然不是图像高度，可能他有其他含义。但是图像行数，就是图像有多少行（row），不就是高度吗。

ImageWidth：这个不说了

ImageLength
Tag = 257 (101.H)
Type = SHORT or LONG

> 其实imageLength可以指：图像height、图像row

### 物理尺寸

应用程序通常希望知道由图像所表示的图片的大小。给定以下分辨率数据，可以根据图像宽度和图像长度计算此信息

ResolutionUnit
Tag = 296 (128.H)
Type = SHORT

### 数据位置

压缩或未压缩的图像数据几乎可以存储在TIFF文件中的任何地方。TIFF还支持将图像分割成单独的条带（Strip图像），以提高编辑灵活性和高效的I/O缓冲。每个条带的位置和大小由以下字段给出

### Strip图像数据：Strip（条带）

TIFF图像数据被组织成条状，以便实现更快的随机访问和高效的I/O缓冲。

拥有Strip数据字段的，表明该图像应该是用条带来存放图像数据的。啥意思呢，就是说把图像分为多个条带（以height为分割元素），假设一个图像为350像素高，那么你可以把350像素高的图像（每个条带表示的图像宽度是一致的，都是imageWidth）都放到一个条带中，或者每个条带只能有100像素高，那么350高度的图像，分为了4个条带，每个条带100像素，最后一个条带只有50像素数据，不填充空白数据。

他拥有下面3个属性，来表明图像数据的组成：

- RowsPerStrip：每个条带的行数（row、height的意思）
- StripOffsets：每个条带数据的偏移量（要解析为数组）
- StripByteCounts：每个条带的数据值（要解析为数组）

### 图像样本位数

参考：<https://www.awaresystems.be/imaging/tiff/tifftags/bitspersample.html>

Sample指样本、组件的意思。比如rgb图像，那么一个像素有3个样本，而这3个样本就是指的：r、g、b 的值。

SamplesPerPixel

Code		277 (hex 0x0115)
Type		SHORT
Count		1
Default		1

每个像素的组件数（样本数）

对于双层、灰度和调色板颜色图像，SamplesPerPixel 通常为 1。RGB 图像的 SamplesPerPixel 通常为 3。如果此值更高，ExtraSamples 应指示附加通道的含义，参考：<https://www.awaresystems.be/imaging/tiff/tifftags/extrasamples.html>。

BitsPerSample
Tag = 258 (102.H)
Type = SHORT

每个样本的位数（每个样本有多少位）

SampleFormat

Code		339 (hex 0x0153)
Type		SHORT
Count		其值：N = SamplesPerPixel 数
Default		1 (unsigned integer data)

指定如何解释像素中的每个数据样本。

规范定义了这些值：

1 = 无符号整数数据
2 = 二进制补码有符号整数数据
3 = IEEE 浮点数据
4 = 未定义的数据格式

注意 SampleFormat 字段没有指定数据样本的大小；这仍然由 BitsPerSample 字段完成。他只是告诉你如何解释BitsPerSample中的数据，比如每个样本有8位，SampleFormat=2，那么就按照“二进制补码有符号整数数据”去解析这8位的数据。

字段值为 undefined 是作者声明它不知道如何解释数据样本；例如，如果它正在复制现有图像。阅读器通常会将具有未定义数据的图像视为该字段不存在（即作为无符号整数数据）。

## 基线TIFF图像示例

基线TIFF其实就是指由图像数据是从做到右，从上到下的顺序进行像素排列的图像。数字切片中的以瓦片来拼接为一张图的图像叫做平铺图像。

基线图像的字段参考：

参考`TIFF 6.0.pdf`：`Section 8: Baseline Field Reference Guide` 基线字段参考。

> 定义下面这些字段之前，必须了解这些基本概念：基线TIFF图像定义为二维像素数组（每行多少个像素，一个图像有多少行），每个像素由一个或多个颜色组件组成。单色数据每像素有一个颜色分量（比如黑白图像，灰度图像），而RGB颜色数据每像素有三个颜色分量（指r、g、b三个颜色分量，通常一个颜色分量是0-255，占一个字节）

### Baseline TIFF bilevel images 二值图像（黑白图像）

基线（基础的）TIFF二值图像在TIFF规范中称为TIFF B类图像。

该图像中的所有必填字段：

```text
TagName Decimal Hex Type Value
ImageWidth 256 100 SHORT or LONG
ImageLength 257 101 SHORT or LONG
Compression 259 103 SHORT 1, 2 or 32773  // 这种压缩是推荐的吧，因为灰度图像只有两种色彩，其他压缩效果可能不太好
PhotometricInterpretation 262 106 SHORT 0 or 1
StripOffsets 273 111 SHORT or LONG
RowsPerStrip 278 116 SHORT or LONG
StripByteCounts 279 117 LONG or SHORT
XResolution 282 11A RATIONAL
YResolution 283 11B RATIONAL
ResolutionUnit 296 128 SHORT 1, 2 or 3
```

> 注意，上面的字段中，省略了其中一些字段。这是允许的，因为省略的字段都有一个默认值，并且默认值适用于此种文件。

###  Grayscale Images：灰度图像

灰度图像是双层图像的推广。双层图像只能存储黑白图像数据，但灰度图像也可以存储灰度阴影。比如颜色为256种灰色的图像。

他和黑白图像的区别：

Compression = 1 or 32773 (PackBits)  压缩方式2通常对连续的色调图像无效，包括许多灰度图像。在这种情况下，最好让图像不被压缩

BitsPerSample = 4 or 8 基线TIFF灰度图像的BitsPerSample允许值为4和8，允许16或256个不同的灰度阴影。

基线TIFF灰度图像在早期版本的TIFF规范中被称为TIFF G类图像

###  Palette-color Images 调色板彩色图像

调色板颜色的图像类似于灰度图像。它们仍然每个像素有一个组件，但是组件值用作完整RGB查找表的索引。

他和黑白图像的区别：

PhotometricInterpretation = 3 (Palette Color)

ColorMap - 颜色贴图（色彩映射表）
Tag = 320 (140.H)
Type = SHORT
N = 3 * (2**BitsPerSample)

其他必需的字段与灰度图像的字段相同

基线TIFF调色板颜色图像在早期版本的TIFF规范中被称为TIFF P类图像

### RGB Full Color Images rgb全彩色图像

在RGB图像中，每个像素由三个组件组成：红、绿和蓝色，没有ColorMap

他和色彩调色板图像的区别：BitsPerSample=8、8、8。每个组件在一个基线TIFF RGB图像中都有8位深。PhotometricInterpretation=2(RGB)。没有颜色贴图。

SamplesPerPixel
Tag = 277 (115.H)
Type = SHORT

对于RGB图像，这个数字为3，除非有额外的样本

基线TIFF RGB图像在早期版本的TIFF规范中被称为TIFF类R图像

## 其他基准TIFF要求

编写TIFF读写器和写入器的要求：`Section 7: Additional Baseline TIFF Requirements`  这些编写一个解析TIFF或者生成TIFF的一些规范和要求。

如果写入了多个子文件（IFD），则`第一个子文件必须是全分辨率的图像`。

## 基线TIFF扩展

基线TIFF的扩展。TIFF扩展是可能所有TIFF阅读器都不支持的TIFF功能。使用这些功能的TIFF创建者将必须与他们所在行业的TIFF读者密切合作，以确保成功的交换（解析），比如瓦片图。

基线TIFF文件可以说，只是用来显示基础TIFF图像的一个文件格式，也是用来显示图片的初衷，而基线TIFF虽然可以包含多个图像（IFD），但是通常只需要解析第一个IFD即可。

而TIFF格式之所以能被用来作为其他私有格式的基础格式，就在于这些扩展字段，因为IFD中的字段是可以自定义的。这就有了很大的灵活性，但也因此，除了基线TIFF格式应该被所有TIFF解析器所支持之外，其他任意扩展TIFF都需要单独看TIFF解析器编写者的心情了，虽然这些扩展也属于标准。但完全兼容，实在太过复杂。

### LZW压缩

无损压缩方式

> LZW适用于不同位深度的图像

每个条都被独立压缩。

## 瓦片图像（平铺）

瓦片图像和条带图像的组织方式请参考svs文档中的一些说明。

对于低分辨率到中分辨率的图像，将图像分成条状的标准TIFF方法是足够的。然而，如果图像被分割成大致正方形的瓷砖，而不是水平宽但垂直的箭头带，就可以更有效地访问高分辨率的图像，而且压缩效果往往更好

> 描述的平铺字段时，它们将替换“StripOffsets”、“StripByteCounts”和“ RowsPerStrip”字段。因此，使用瓷砖将导致旧的TIFF阅读器放弃，因为他们将无法知道图像数据在哪里或它是如何组织的。不要在同一TIFF文件中同时使用条状字段和面向瓦片的字段

### 填充

瓷砖尺寸由瓷砖宽度和瓷砖长度（ TileWidth and TileLength）来定义。图像中的所有瓷砖都大小相同；也就是说，它们具有相同的像素尺寸。

如果填充宽度为64，图像宽度为129，则图像为3块填充栏宽，并且`必须添加`63个像素的填充栏来填充最右边的填充栏。瓷砖长度和图像长度也是如此。使用什么进行填充的值并不重要，因为`良好的TIFF读卡器`只显示由图像宽度和图像长度定义的像素，并忽略任何填充像素。如果通过复制最后一列和最后一行而不是用0填充来完成填充，某些压缩方案效果有效。也就是说，图像的宽和高属性的值，是真实的图像的宽高，而瓦片图中，可能会具有填充像素，但是这个宽高不包含这些填充的像素（你不要`依赖`它包含）。

瓷砖被单独压缩

### 平铺图像需要以下字段

TileWidth
Tag = 322 (142.H)
Type = SHORT or LONG
N = 1 （数量为1）

没有默认值

> TileWidth必须是16的倍数。这一限制提高了某些图形环境中的性能，并增强了与JPEG等压缩方案的兼容性。

TileLength
Tag = 323 (143.H)
Type = SHORT or LONG
N = 1

替换平铺TIFF文件中的RowsPerStrip。

TileOffsets
Tag = 324 (144.H)
Type = LONG
N = TilesPerImage for PlanarConfiguration = 1  图像的瓦片数
= SamplesPerPixel * TilesPerImage for PlanarConfiguration = 2 （这个并没有广泛使用，可以不用支持）

数组

对于每个磁块，该磁块的字节偏移量，作为压缩并存储在磁盘上。偏移量针对TIFF文件的开头指定。请注意，这意味着每个瓷砖都有一个独立于其他瓷砖的位置的位置。偏移是从左到右和从上到下排序

TileByteCounts
Tag = 325 (145.H)
Type = SHORT or LONG
N = TilesPerImage for PlanarConfiguration = 1
= SamplesPerPixel * TilesPerImage for PlanarConfiguration = 2

数组

对于每个平铺，该平铺中的`（压缩）`字节数

## TIFF中的jpeg

参考：[JPEG/Exif/TIFF格式解读(1):JEPG图片压缩与存储原理分析](https://zhuanlan.zhihu.com/p/163502463)

Compression
Tag = 259 (103.H)
Type = SHORT
N = 1

6 = JPEG（“旧式”JPEG，后来在 Technote2 中被覆盖）
7 = JPEG（'新型'JPEG）

他们之间的区别参考：<https://stackoverflow.com/questions/5733867/jpeg-from-tiff-jpeg-compressed>

> 旧式 TIFF-JPEG（压缩类型 6）基本上是在 TIFF 包装器内填充了一个普通的 JFIF 文件。较新样式的 TIFF-JPEG（压缩类型 7）允许将 JPEG 表数据（霍夫曼，量化）存储在单独的标签 (0x015B JPEGTables) 中（如果使用该字段的话）。这允许您将带有 SOI/EOI 标记的 JPEG 数据条带放入文件中，而无需重复 Huffman 和量化表。这可能是您在文件中看到的内容。各个条带以序列 FFD8 开头，但缺少霍夫曼和量化表。这是 Photoshop 产品通常写入文件的方式。

参考对jpeg格式的理解，你可以知道这个量化表和霍夫曼表是干啥的了，以及怎么添加到data数据中。

> svs文件中JPEGTables的这个字段，会在最开始和最结尾分别添加上：FFD8和FFD9，这是jpeg文件的开始和结束标记符。

正确使用JPEGTables，查看如下资料：

- [jpeg标记段文档](https://exiftool.org/TagNames/JPEG.html)
- [相关问题](https://stackoverflow.com/questions/50798014/determining-color-space-for-jpeg)
