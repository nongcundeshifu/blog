---
title: TIFF文件解析
date: 2021-10-22 11:13:43
tags:
  - TIFF
categories:
  - TIFF
---

这篇文章我们来尝试实现我们自己的tiff文件解析器。

<!-- more -->

## 说明

这里的解析器，仅仅只是解析出tiff中的数据，但不涉及到数据的解释，这一部分我觉得只有在你确定你的解析器要用来做什么之后才开始考虑，原因在于，tiff格式本身包含的规范非常复杂，基于标签的数据存储能够使其非常灵活的同时，也给解析器编写带来了麻烦，主要不是说数据解析本身麻烦，而是基于标签就代表着解析器需要对这个tiff文件中所具有的标签（IFD -> DE -> tag）的含义能够有一个正确的认识。可以说，解析器的功能，取决于这个解析器能认识多少种tiff tag标签：

- 标签多且繁杂，因为tiff标准本身就支持多种色彩、多种类型的图像
- 也许tiff文件中包含私有标签

所以，如果要写解析器，会更倾向于基于业务本身的需求去编写，就比如我司需要支持基于tiff规范的数字切片的解析，所以我会仅考虑基于瓦片图像（平铺）的TIFF文件去编写我的解析器。

## 解析tiff中包含的数据

解析出tiff文件中的数据，并将其结构化还是比较简单的，因为不涉及到对其数据的解释。而且，基本上，解析过程就是按照之前我们的那两篇文章所描述的内容一步一步去处理的：

- [TIFF文件格式了解](/2021/10/05/TIFF/TIFF文件格式了解/)
- [TIFF6文档参考](/2021/10/13/TIFF/TIFF6文档参考/)

> 这里会支持bigtiff的解析，但是需要注意的是，8位的BigInt转换为数字之后精度会丢失。

### 解析IFH

我们最开始需要读取tiff的IFH信息，即头文件信息：

- 为了兼容bigtiff，我们先从tiff文件中，读取前16个字节的buffer数据：headBuffer（如果仅兼容标准tiff，则只需要读取8个字节即可，而bigtiff则为16个）
- 从headBuffer数据中读取前2个字节，用于判断字节序
- 读取第3-4个字节，用于判断其值为42或者43（表明是标准tiff还是bigtiff）
- 确定他是标准tiff或者bigtiff之后，根据其IFH的标准，读取第一个IFD所在的偏移量。

将得到的信息组装为格式化数据。

### 解析IFD

在我们获取到tiff的IFH头信息之后，我们假设为`标准tiff文件`吧（bigtiff也是一样的，只不过一部分数据的所占据的字节不一致而已），现在我们需要递归读取完IFD中的数据：

- 我们根据之前获取的IFH中的第一个IFD偏移量找到IFD数据。读取其中的2字节，得到这个IFD中的DE的数量
- 从文件中读取完整个IFD的数据ifdBuffer：`2 + DE数 * 12 + 4`
- 然后，依次解析完这个IFD中的DE数据
- 我们定义一个`tiffTagCodeMap`和`tiffTagTypeMap`数据，用于映射和标准化我们获取到的所有tag信息。
- 根据DE中的tag值（注意一个细节，tag值是升序排列的）、type类型、数量和偏移量去从文件中读取该数据并转换为标准的数据结构。
- 解析完DE之后，如果ifdBuffer的最后4个字节不为0，则递归解析下一个IFD的数据，知道某个IFD的最后4字节为0，则表明IFD解析完毕。

tiffTagCodeMap参考

```ts

export enum TiffTagCodeEnum {
    NewSubfileType = 254,
    ImageWidth = 256,
    ImageLength = 257,
    BitsPerSample = 258,
    Compression = 259,
    PhotometricInterpretation = 262,
    Threshholding = 263,
    FillOrder = 266,
    ImageDescription = 270,
    Make = 271,
    Model = 272,
    StripOffsets = 273,
    Orientation = 274,
    SamplesPerPixel = 277,
    RowsPerStrip = 278,
    StripByteCounts = 279,
    XResolution = 282,
    YResolution = 283,
    PlanarConfiguration = 284,
    ResolutionUnit = 296,
    Software = 305,
    DateTime = 306,
    Artist = 315,
    ColorMap = 320,
    TileWidth = 322,
    TileLength = 323,
    TileOffsets = 324,
    TileByteCounts = 325,
    SubIFDs = 330,
    ExtraSamples = 338,
    JPEGTables = 347,
    GlobalParametersIFD = 400,
    YCbCrCoefficients = 529,
    YCbCrSubSampling = 530,
    YCbCrPositioning = 531,
    Copyright = 33432,
}

export const tiffTagCodeMap = new Map<
    TiffTagCodeEnum,
    {
        tagName: TiffTagName
        description: string
    }
>()
    .set(TiffTagCodeEnum.NewSubfileType, {
        tagName: 'NewSubfileType',
        description: '此子文件中包含的数据类型的一般指示，替换SubfileType，在TIFF文件中有多个子文件时有用',
    })
    .set(TiffTagCodeEnum.ImageWidth, {
        tagName: 'ImageWidth',
        description: '图像宽度',
    })
    /**
     * 我就好奇，这个为啥是图像高度
     */
    .set(TiffTagCodeEnum.ImageLength, {
        tagName: 'ImageLength',
        description: '图像长度',
    })
    .set(TiffTagCodeEnum.BitsPerSample, {
        tagName: 'BitsPerSample',
        description: '每样本位数',
    })
    /**
     * 1：未压缩
     */
    .set(TiffTagCodeEnum.Compression, {
        tagName: 'Compression',
        description: '压缩',
    })
    .set(TiffTagCodeEnum.PhotometricInterpretation, {
        tagName: 'PhotometricInterpretation',
        description: '光度解释',
    })
    .set(TiffTagCodeEnum.Threshholding, {
        tagName: 'Threshholding',
        description: '对于表示灰度阴影的黑白 TIFF 文件，该技术用于将灰色像素转换为黑白像素。',
    })
    .set(TiffTagCodeEnum.FillOrder, {
        tagName: 'FillOrder',
        description: '填充顺序，默认为1，且通常不会使用2',
    })

    /**
     * 是ASCII
     */
    .set(TiffTagCodeEnum.ImageDescription, {
        tagName: 'ImageDescription',
        description: '描述图像主题的字符串。',
    })
    .set(TiffTagCodeEnum.Make, {
        tagName: 'Make',
        description: '扫描仪制造商。',
    })
    .set(TiffTagCodeEnum.Model, {
        tagName: 'Model',
        description: '扫描仪型号名称或编号。',
    })

    /**
     * 可以理解为图像数据的开始
     */
    .set(TiffTagCodeEnum.StripOffsets, {
        tagName: 'StripOffsets',
        description: '条带偏移',
    })
    .set(TiffTagCodeEnum.Orientation, {
        tagName: 'Orientation',
        description: '图像相对于行和列的方向。',
    })
    .set(TiffTagCodeEnum.SamplesPerPixel, {
        tagName: 'SamplesPerPixel',
        description: '每个像素的组件数。（分向量数），比如rgb图像，一个像素有3个组件，分别代表r、g、b',
    })
    .set(TiffTagCodeEnum.RowsPerStrip, {
        tagName: 'RowsPerStrip',
        description: '每条行数',
    })
    /**
     * 数据的字节数
     */
    .set(TiffTagCodeEnum.StripByteCounts, {
        tagName: 'StripByteCounts',
        description: '条带字节计数',
    })
    .set(TiffTagCodeEnum.XResolution, {
        tagName: 'XResolution',
        description: 'X分辨率',
    })
    .set(TiffTagCodeEnum.YResolution, {
        tagName: 'YResolution',
        description: 'Y分辨率',
    })
    /**
     * 这个很烦，默认为1，PlanarConfiguration = 2未广泛支持。
     * 如果 SamplesPerPixel 为 1，则 PlanarConfiguration 无关紧要
     */
    .set(TiffTagCodeEnum.PlanarConfiguration, {
        tagName: 'PlanarConfiguration',
        description: '每个像素的分量是如何存储的。默认为1，PlanarConfiguration = 2未广泛支持。',
    })
    .set(TiffTagCodeEnum.ResolutionUnit, {
        tagName: 'ResolutionUnit',
        description: '分辨率单位',
    })
    .set(TiffTagCodeEnum.Software, {
        tagName: 'Software',
        description: '用于创建映像的软件包的名称和版本号。',
    })
    .set(TiffTagCodeEnum.DateTime, {
        tagName: 'DateTime',
        description: '图像创建的日期和时间。',
    })
    .set(TiffTagCodeEnum.Artist, {
        tagName: 'Artist',
        description: '创建图像的人。',
    })
    .set(TiffTagCodeEnum.ColorMap, {
        tagName: 'ColorMap',
        description: '调色板颜色图像的颜色图。',
    })
    /**
     * 瓦片图相关
     */
    .set(TiffTagCodeEnum.TileWidth, {
        tagName: 'TileWidth',
        description: '瓦片图的宽',
    })
    .set(TiffTagCodeEnum.TileLength, {
        tagName: 'TileLength',
        description: '瓦片图的高',
    })
    .set(TiffTagCodeEnum.TileOffsets, {
        tagName: 'TileOffsets',
        description: '瓦片图的偏移量-大数组',
    })
    .set(TiffTagCodeEnum.TileByteCounts, {
        tagName: 'TileByteCounts',
        description: '每张瓦片图的字节数-大数组',
    })

    .set(TiffTagCodeEnum.SubIFDs, {
        tagName: 'SubIFDs',
        description: '子IFD',
    })

    .set(TiffTagCodeEnum.ExtraSamples, {
        tagName: 'ExtraSamples',
        description: '额外组件的描述。',
    })
    .set(TiffTagCodeEnum.JPEGTables, {
        tagName: 'JPEGTables',
        description: 'JPEG 量化和/或霍夫曼表。',
    })
    .set(TiffTagCodeEnum.GlobalParametersIFD, {
        tagName: 'GlobalParametersIFD',
        description:
            '指向包含全局适用于完整 TIFF 文件的标签的 IFD。对于全局TIFF图像IFD中都适用的字段集，如果有，最好在第一个IFD中',
    })

    .set(TiffTagCodeEnum.YCbCrCoefficients, {
        tagName: 'YCbCrCoefficients',
        description: '从 RGB 到 YCbCr 图像数据的转换。',
    })
    .set(TiffTagCodeEnum.YCbCrSubSampling, {
        tagName: 'YCbCrSubSampling',
        description: '指定用于 YCbCr 图像的色度分量的子采样因子。',
    })
    .set(TiffTagCodeEnum.YCbCrPositioning, {
        tagName: 'YCbCrPositioning',
        description: '指定子采样色度分量相对于亮度样本的位置。',
    })

    .set(TiffTagCodeEnum.Copyright, {
        tagName: 'Copyright',
        description: '版权声明。',
    })
```


tiffTagTypeMap参考

```ts
const tiffTagTypeMap = new Map()
        .set(1, {
            name: 'BYTE',
            size: 1,
            description: '8位无符号整数。',
            getValue: (data: Buffer) => {
                // 根据改类型去解析该值
            },
        })
        .set(2, {
            name: 'ASCII',
            size: 1,
            description: '包含7位ASCII码的8位字节；最后一个字节必须是 NUL（二进制零）',
            getValue(data: Buffer) {
                return data.toString('ASCII')
            },
        })
        .set(3, {
            name: 'SHORT',
            size: 2,
            description: '16位（2 字节）无符号整数。',
            getValue: (data: Buffer) => {
                // 根据改类型去解析该值
            },
        })
        .set(4, {
            name: 'LONG',
            size: 4,
            description: '32 位（4 字节）无符号整数。',
            getValue: (data: Buffer) => {
                // 根据改类型去解析该值
            },
        })
        .set(5, {
            name: 'RATIONAL',
            size: 8,
            description: '分数。两个 LONG：第一个代表分数的分子；第二，分母。',
        })
        .set(6, {
            name: 'SBYTE',
            size: 1,
            description: '一个 8 位有符号（二进制补码）整数。',
        })
        .set(7, {
            name: 'UNDEFINED',
            size: 1,
            description: '一个 8 位字节，可以包含任何内容，具体取决于字段的定义。',
            getValue(data: Buffer) {
                return data
            },
        })
        .set(8, {
            name: 'SSHORT',
            size: 2,
            description: '一个 16 位（2 字节）有符号（二进制补码）整数。',
        })
        .set(9, {
            name: 'SLONG',
            size: 4,
            description: '一个 32 位（4 字节）有符号（二进制补码）整数。',
        })
        .set(10, {
            name: 'SRATIONAL',
            size: 8,
            description: '分数。两个 SLONG：第一个代表分数的分子，第二个代表分母。',
        })
        .set(11, {
            name: 'FLOAT',
            size: 4,
            description: '单精度（4 字节）IEEE 格式。',
            getValue: (data: Buffer) => {
                // 根据改类型去解析该值
            },
        })
        .set(12, {
            name: 'DOUBLE',
            size: 8,
            description: '双精度（8 字节）IEEE 格式。',
            getValue: (data: Buffer) => {
                // 根据改类型去解析该值
            },
        })
        .set(16, {
            name: 'LONG8',
            size: 8,
            description: 'bigTiff支持的无符号 8',
            getValue: (data: Buffer) => {
                // 根据改类型去解析该值
            },
        })
        .set(17, {
            name: 'SLONG8',
            size: 8,
            description: 'bigTiff支持的有符号 8',
            getValue: (data: Buffer) => {
                // 根据改类型去解析该值
            },
        })
        .set(18, {
            name: 'IFD8',
            size: 8,
            description: 'bigTiff支持的无符号 8 字节 IFD 偏移量',
            getValue: (data: Buffer) => {
                // 根据改类型去解析该值
            },
        })
```

一个基于TIFF实现的数字切片文件所解析出来的的结果如下：

![解析tiff的结果](https://image.ncdsf.com/2022/07/06/20220706170054.png)
![解析tiff的结果](https://image.ncdsf.com/2022/07/06/20220706165852.png)
![解析tiff的结果](https://image.ncdsf.com/2022/07/06/20220706170023.png)

当我们解析完毕IFD数据之后，其实我们就完成了对整个tiff数据的解析。当然，这里仅仅只是将tiff中的数据解析为一个格式化的数据结构而已，想要真正的得到图像，就需要对tag标签进行解读，并根据得到的tag信息去解读tiff中的数据了。以下文的中的IFD为例，这个IFD表示的是一个条带图像（普通的tiff图像文件只有一个IFD结构，且图像大部分是以条带图像的形式存储）：

![条带图像的IFD数据](https://image.ncdsf.com/2022/07/06/20220706171720.png)

我在得到这个IFD之后，根据TIFF规范，如果是一个条带图像（Strip），则需要包含如下三个tag（其实每一种类型图像，都有必要的属性，你根据必要的属性去判断即可）：

- RowsPerStrip：每个条带的行数（row、height的意思）
- StripOffsets：每个条带数据的偏移量（要解析为数组）
- StripByteCounts：每个条带的数据值（要解析为数组）

我们只需要判断这个IFD中是否包含这三个tag即可，如果不包含，则表明他不是一个条带图像，如果他包含了，则说明是一个条带图像，然后按照条带图像的规范去解析：

- 根据StripOffsets和StripByteCounts获取所有的条带数据
- 通过Compression获取其条带数据的压缩方式
- 已知图像的压缩方式和所有条带图像数据，将其拼接成一幅完整的图像

至此，就得到了这个IFD所存储的图像了。

## 总结

这里我们主要说明了该如何解析一个tiff文件，并获取其中的数据。而且我们还以一个条带图像的IFD为例，展示了如何解析IFD中所存储的图像数据，而以瓦片图形式存储图像的tiff文件也是是类似的解析方式，有兴趣的读者可以自行尝试一下。然而TIFF格式规范确实复杂，想要支持的较为完善，可不简单啊。
