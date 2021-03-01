---
title: Hello World
---

# electron和sdpc的node解析服务



关于sdpc文件解析服务，其方案被定为，electron中开启一个用于解析sdpc的node服务，以便提供http请求api给electron浏览器和服务器中使用。

**目的**

+ 用于在electron解析和获取切片的瓦片图
+ 用于在服务端读取sdpc瓦片数据。



**剩余问题**

+ electron和渲染层的通信。
+ type是否需要抽离出来，不然渲染层和electron的类型其实会重复定义的。
+ 如何开启http2？
+ 瓦片和缩放级别的一个对应关系问题。



## 方案

参考：https://www.processon.com/view/link/5fd812b8f346fb07100e37fa

**技术选型**

使用nestjs搭建node服务，然后再electron中进行启动，此时此node服务作为electron的子进程进行控制，并且，他可以和electron进行进程通信。



### electron主进程、渲染进程、sdpc服务

**electron**

主进程，用于启动和关闭sdpc服务

**sdpc服务**

一个nestjs搭建的node服务，提供http服务接口，用于sdpc文件的解析和瓦片图的读取。

**渲染进程-前端页面**

通过http直接访问sdpc服务



### 一些和普通http服务器的不同细节

#### 数据存储和缓存

在渲染进程中浏览一张切片，他除了需要展示切片信息之外，更多的是需要快速的获取瓦片图数据。而获取瓦片图数据的前提就是这张切片数据已经解析过了。所以，需要下面的存储和缓存提高性能。

**缓存**

缓存，使用类似mogo这些非关系数据库作为缓存，它存的是sqlite中的数据，说明渲染进程正在使用这一个切片数据。缓存会在node服务关闭后丢失。

> 目前简单做可以直接用内存，做一个简单的缓存失效或者让渲染层再调用一次http接口进行内存释放。

**存储**

类似sqlite数据库，存储远程解析出来的数据到本地数据库，本地文件解析出来的数据直接作为缓存，只在sdpc服务的生命周期中有效。因为本地解析本来就无法作为一个持久存储，他移动一下文件本地路径，其存储就失效了。

> 注意：一旦你决定使用了缓存，则你就要确定你所存储的数据是完整的且格式不能轻易改动，所以，对于数据结构的控制，你需要很小心。



**图片缓存**

因为图片都是动态获取的，所以，图片无法进行缓存，只能由node服务进行缓存。

浏览器域名的并发请求限制





#### electron和服务器的sdpc服务的异同

electron版本的sdpc服务和服务器上的sdpc服务他们在功能上是存在分化的，这是不可避免的，你需要在同一套代码中，兼容同一个接口的不同实现（注意实现要优雅）

**相同点**

+ 需要一定的容灾性
+ 可能需要提供主从模式，以提高性能（但是此时，就一定要用一个缓存数据库了，比如mogo，来共享不同子进程的缓存数据）

**electron**

+ 用于应用程序中，程序关闭后，sdpc服务就会关闭
+ 他需要缓存

**服务端**

+ 他在服务器需要一直跑起来
+ 需要更好的容灾
+ 基本上需要提供主从模式以提高性能
+ **他不需要数据的缓存数据，也不需要存储数据**，或者可以存，这样可以保持服务代码逻辑的一致性，但是需要定时任务清理，因为在服务端他属于垃圾数据。



**开发注意**

+ 提供不同的启动参数，注入环境变量，以判断服务端还是electron端的启动，以此来使用不同的实现类。





### 同步本地和远程文件的缓存

提供给electron版本的sdpc服务，提供一个缓存机制，渲染层在浏览一个数字切片之前，需要调用sdpc服务的一个接口，告诉sdpc服务，我将要浏览这个数字切片了，此时sdpc服务会对此切片的切片数据进行读取（不管是本地还是远程），读取此切片的数据之后，缓存在内存之中，并将相应的token返回给渲染层，渲染层利用这个uuid来标识某一个切片。





## sdk实现

**lib库**
lib库提供封装dll，转换dll数据，调用dll方法，转换dll方法的参数和数据为js对象。兼容linux和windos
**sdk库**
sdk库封装lib中的方法，组合lib中的方法，提供更好用的api。简化dll方法调用。
对本地的访问和远程s3访问的封装，提供统一接口。







### 待实现

**数据获取**

- [ ] 将图片数据，转换为偏移量
- [ ] 添加额外信息的获取





## 服务端实现



### api接口分类

**sdpc文件解析**

+ 解析sdpc文件，获取所有信息（非sdk结构体的，而是json的）







## 其他

electron显示本地图片，参考：

+ https://www.329329.com/key/Electron-vue%E5%BC%80%E5%8F%91%E5%AE%9E%E6%88%98%E4%B9%8BElectron%E5%85%A5%E9%97%A8.html


```typescript
import { ajax } from '../../../common/business/constant/allConstant'

export interface AIAbout {
    specType: string // 规格型号
    releaseType: string // 发布型号
    productSn: string // 生产批号
    productDate: string // 生产日期
    licenceInterval: string // 使用期限
    productCompany: string // 生产企业
    productCompanyAddr: string // 生产地址
}

/**
 * 类型，1:实视图片，2:数字切片
 */
export enum AIResultTypeEnum {
    RealViewImage = 1,
    DigitalSlice,
}

/**
 * 切片存储类型 1 - 本地存储 2 - 华为 s3c
 */
export enum SliceStoreTypeEnum {
    LocalStorage = 1,
    H3C,
}

interface AIResultRect {
    startX: number // 矩形左上角 x 坐标
    startY: number // 矩形左上角 y 坐标
    endX: number // 矩形右下角 x 坐标
    endY: number // 矩形右下角 y 坐标
}

export interface AIResultRealViewImage {
    // 实视图片分析结果，返回 type=1 时有效
    tags?: (AIResultRect & {
        tagId: number // 标注框标签 ID
    })[]
    path: string // 图片绝对路径
    height: number // 图片高度
    width: number // 图片宽度
}

export type AIResultDigitalSliceTag = AIResultRect & {
    multiple: number // 放大倍数
    slicePath: string // 切片地址
}

export type BackgroundImage = {
    path: string // 图片绝对路径
    height: number // 图片高度
    width: number // 图片宽度
}

// AI 整片判断结果 0 - 阴性 1 - 阳性
export enum WholeInfoPositiveEnum {
    Negative = 0,
    Positive,
}

export interface AIResultDigitalSlice {
    wholeInfo: {
        // 整片信息
        backgroundImage: AIResultDigitalSliceTag
        backgroundImageList: BackgroundImage[]
        wholeTagId?: number[] // 整片判断诊断术语 id
        backgroundType?: number // 背景类型
        positive?: WholeInfoPositiveEnum
    }
    // 切片分析结果，返回 type=2 时有效
    abnormalInfo: {
        // 异常信息
        tags: AIResultDigitalSliceTag[]
    }[]
    slicePath: string // 切片地址
    sliceStoreType: SliceStoreTypeEnum
}

export type AIResult =
    | {
          type: AIResultTypeEnum.RealViewImage
          img: AIResultRealViewImage
      }
    | {
          type: AIResultTypeEnum.DigitalSlice
          slice: AIResultDigitalSlice
      }

class AIApi {
    /**
     * 关于信息
     */
    about() {
        return ajax.get<AIAbout>('/miis/ai/about')
    }

    /**
     * 查询分析结果
     * @param id
     */
    getResult(id: string) {
        return ajax.post<AIResult>(
            '/miis/ai/getResult',
            { path: id },
            {
                hideLoading: true,
            },
        )
    }
}

export const aiApi = new AIApi()

```



## 问题

### 测试

+ 在jest测试时，文件路径不属于程序的运行目录，所以，你无法搜索到文件目录下的dll，你可以用process.cwd看一下。


