---
title: npm包项目-multirepo到monorepo的演进
date: 2024-4-22 10:43:40
tags:
  - 前端工程化
  - npm
  - monorepo
categories:
  - 前端工程化
---

随着业务的发展，公司开发的npm SDK业务越来越复杂，功能越来越多，以及在SDK被业务使用的过程中，也遇到一些较为棘手的问题，multirepo已经逐渐不再适用发展需求了，故尝试对SDK仓库进行改造，从multirepo转向了monorepo。

<!-- more -->

## 背景

前文参考：[npm包项目-前端工程化演进](/2024/03/22/前端工程化/npm包项目-前端工程化演进/)

在SDK项目最开始，它仅仅只作为一个普通的npm包项目，没有那么多要求，只要能正常的构建打包，并发布到npm仓库中被其他业务使用即可。

然而，随着业务的发展，其业务越来越复杂，功能越来越多，以及SDK在被业务使用的过程中，也遇到一些较为棘手的问题，multirepo已经逐渐不再适用发展需求了，故尝试对SDK仓库进行改造，从multirepo转向了monorepo。

## 遇到的问题

在改造之前，我们先看看看我们遇到的问题：

- 项目结构由于SDK代码的越发庞大也变得逐渐混乱，出现了循环依赖等问题，尤其是SDK的类型导出
- 由于SDK库中定义了大量类型并导出给业务使用，业务会依赖该类型从而引用SDK这个库，但是对于多页面打包的项目（例如PC）且**某些页面只是依赖了ts类型，就需要打包整个SDK库**，会增加其文件体积以及加载速度
- **import type**是一个好方法，但是对枚举来说无法生效，SDK的枚举通常也是值。
- **tree shaking**是个方案，但是需要业务自行处理。
- 由于该SDK依赖的IM库显式依赖window对象的接口，且在引入时就会进行执行，导致其在单元测试环境中需要单独mock掉该模块，且对于业务中的单元测试也可能会产生影响（如果业务引用了某个类型的话）
- SDK在开发过程中，通过编写demo进行自测，而这一部分是一个react应用，为了方便快速的测试开发中的功能，demo部分的代码和SDK的源码是放在一起的，导致SDK包本身包含了一些web项目才需要的依赖。而且这种情况下demo在使用SDK时无法保证和业务使用时是一致的。可能会无法发现业务使用时的一些问题
- 功能测试的引入导致了大量的测试代码以及对一些puppeteer等依赖的引入，同样如果不放到SDK仓库中，则在开发后的自动化测试则会较为麻烦，而如果直接放到SDK仓库中，则混合了不必要的依赖。
- 文档的完善：文档如果要发布的话，其实也是一个项目

## 迁移考量

我们看到，随着SDK功能越发的庞大以及越来越成熟和完善，和SDK库本身相关的内容越来越多，他们其实也可以放到一个multirepo仓库中进行维护，不过随之而来的是越来越庞大和复杂的目录结构，其维护成本也随之变高。而如果仓库的每个部分都较为独立，且可以分而治之，所以我认为monorepo相比于multirepo是一个更好的选择：

- 各种SDK相关的内容互相隔离（文档、示例、功能测试等）功能划分和目录结构更为清晰
- 和SDK相关的内容的公共依赖进行统一，将其放到一起可以更加方案的维护和互相引用
- monorepo本身经过多年发展也较为成熟
- SDK所面临的问题和需求也较为符合monorepo的使用场景。

## 风险考量

迁移存在一定风险，在正式迁移之前，需要对风险进行评估：

- SDK库本身的创建时间较早，所以其依赖都较为新，其只是工程架构更改为monorepo
- 从yarn变成pnpm的迁移，对于其依赖，由于存在lock文件，可以直接将yarn的lock变成pnpm的lock，不会则不会对其进行具体版本进行升级
- 架构和包的更新，对于业务的影响控制：由于在更新monorepo之前，其SDK的功能测试项目已经完成了大半，可以借助其自动化的功能测试，对更新项目架构的SDK进行自动化测试。

从上面的风险点看，其都在可控范围。故考虑可以进行迁移。

## 改造为monorepo

### 分包

按照上面我们遇到的问题，以及我们的需求，目前的SDK大致分为如下几个子包：

- SDK核心库
- SDK类型库
- example示例demo（用于开发过程中的功能自测）
- 功能测试：基于puppeteer的SDK功能自动化测试
- docs：SDK对外的使用文档（包括接口文档和使用文档）

除此之外，SDK本身设计了hooks和插件机制，后续在拆分模块或者插件化时，也可以再次以package为单位进行组织。不过注意，过犹不及，合适的才是最好的。故在最一开始，没有对SDK本身的功能模块进行包的拆分。

如此，就可以在同一个npm项目中，完成如下整个开发流程：

- SDK的逻辑开发
- 开发过程中通过示例demo进行简单的验证
- 再通过功能测试保证改动后整个模块其他业务的功能的稳定性
- 最后补充docs文档
- 更新依赖版本然后发布即可。

且业务在一些仅需要依赖SDK提供的类型的场景下（例如PC多页面、单元测试），也可以直接引用SDK类型库，而无需引入整个SDK的核心库，避免业务在打包时，打进了不必要的包。

> ps：独立的类型包，其实并非必要，因为不拆分也同样可以做到让业务引用单独的类型并且不包含业务逻辑。

### monorepo选型

改造为monorepo的选择一般有下面两种：

- yarn + lerna
- pnpm

考虑到pnpm本身的装包速度以及解决的幽灵依赖等特性，并且基本上其发展已经成熟了，并且我已经在SDK的功能测试仓库中提前试用过，其安装速度确实令人惊讶，了解到其pnpm本身的monorepo支持就已然非常丰富，对于目前我们的SDK的项目规模来说已然足够。

虽然还有一些像Rush和NX等方案，不过他们都感觉太重了，比如云端构建、提供整套的解决方案等，以目前的SDK项目规模来说，还没有大到这种程度，如果引入反而会给SDK带来不必要的复杂性和成本。

如此，SDK选用的是pnpm作为包管理器以及monorepo的管理，选择它有以下几个理由：

- pnpm包本身的性能，速度快，且已成熟
- pnpm的monorepo相比于lerna + yarn更为简单，且是包管理器本身自带的，包含了依赖管理和不错monorepo项目管理能力
- 在满足需求的情况下，无需再额外引入lerna

### 部署

从multirepo改造为monorep，对于npm的发布来说，影响到不大，只需要更新一下构建目标的命令即可。具体看公司的云端环境，这里不具有参考性，故忽略。

## 结语

如此我们就完成了SDK项目从multirepo到monorepo的迁移，其实这里重点表达的并非如何实现monorepo项目，而是在于如何基于实际的应用场景和遇到的问题，来决定我们是选择multirepo还是monorepo。