---
title: npm包项目-文档编写
date: 2024-5-03 09:13:40
tags:
  - 前端工程化
  - npm
  - 文档
categories:
  - 前端工程化
---

作为一个npm包，文档对于其业务的引用者来说，自然是重中之重，尤其是npm包的开发者和业务的使用者不是同一个人，甚至不是同一个业务。故能有一份完善的文档，对于npm包的使用者来说是最安心不过的了。

<!-- more -->

## 前言

前文参考：[npm包项目-前端工程化演进](/2024/03/22/前端工程化/npm包项目-前端工程化演进/)

作为一个npm包，文档对于其业务的引用者来说，自然是重中之重，尤其是npm包的开发者和业务的使用者不是同一个人，甚至不是同一个业务。故能有一份完善的文档，对于npm包的使用者来说是最安心不过的了。

项目的开发文档可以没有那么急，但是使用文档绝对不能少，不然第三方在接入你的npm包时，两眼一抹黑，无从下手，相信你自身在使用别人的npm包时，看到没有文档，肯定也会觉得比较难受。

## 背景

目前来说，在经过了大半年的开发后，其公司SDK的整体架构都已经区域稳定，后续则是基于产品需求补充相应的业务功能。而这个时期的开发节奏都比较快，在保证效率和质量的前提下，一些本不该忽略的东西，也是时候给补上了。

且作为互动应用中（PC、APP），最核心的互动业务的SDK，也需要团队中的其他人熟悉了解，并在必要时能够承担相应的开发任务，故是时候将SDK相关内容的文档给补齐了。

## 现状和需要解决的问题

- 在快节奏的开发中，SDK文档逐渐缺失和过时，没有文档都要好过错误的文档。所以，为了方便后续开发人员的理解和使用，减少沟通成本、提高开发效率，需要补齐缺失的文档以及更正过时的文档
- SDK初始时所使用的文档构建工具和构建流程太过冗余和复杂，整个文档和SDK源码部分有一定耦合，开发者难以维护和更新。故我们需要找到更为简单和高效的文档工具。

## 文档类型

通常npm包的文档主要分为以下几个部分：

- 使用文档：基本介绍，npm包的功能、接入方法、示例、注意事项等，这个通常由开发者编写
- 接口文档：整个npm包的接口定义和类型定义，通常可以直接生成
- 开发文档：这一部分着重于npm库的架构、设计、项目结构、模块之间的协作、约定等等，通常是给库的开发者使用的。

## 文档构建工具

在一开始SDK的multirepo仓库阶段，我们利用tsdoc配合api-extractor、api-documenter以及docsify来生成API文档以及文档站点，且将其部署到了云端，以便用户进行查看，但是其配置和构建都太过复杂，且命令众多，理解和维护成本都太高。

我们所需要的是一个API文档和一个静态站点生成器，因为我们除了API文档外，我们还有changelog、使用指南等文档，这些需要部署到服务器上供其他人浏览。后来在调研了一些文档生成工具后，最终我们选择了如下的构建工具来作为我们的文档构建工具：

- typedoc：仅仅只是API文档生成器，他支持tsdoc的文档注释，配合ts本身的类型，它能够通过简单的配置就生成出一个非常不错的文档站点或者markdown文件。
- vitepress：一个静态站点生成器：<https://vitepress.dev/zh/guide/what-is-vitepress>。其基于vite的使他的速度和热更新都非常快，对开发和编写文档的体验非常好，且配置简单

## 使用文档

这部分的文档，基本上是由开发者开发完毕后自行编写，目的是为了让第三方接入者基于这个文档，能够知道如何这个库是干什么的， 有什么功能，如何使用它，反正将自己带入一个需要接入这个库、但是对这个库一无所知的开发者的视角，以这个视角来审视自己要接入SDK期望知道什么。

常用的文档大纲：

- 简介
- 接入方法
- 核心功能说明
- 特殊约定和技巧
- 示例
- 注意事项
- 其他额外信息
- changelog

### 术语表

通常函数、类型、参数的命名都尽量要求符合语义，这样能减少注释的工作量，且一些常用的函数、参数在良好的语义化命名下都不需要写注释。例如userUid、phone这种

但是，如果是项目独有、但是又有具体含义，且多次出现的，则可以提供统一的术语表，这也算是一种约定俗成。统一心智负担。

不过原则还是那个，如果你是开发者，你希望这个文档里面有什么，然而，这里面的尺度需要自行把握了：事无巨细则意味着时间和成本，但也意味着文档好用，这本身也是一个权衡。

> ps：通常使用文档作为对外暴露的文档入口，会包含API文档部分。

## API文档

API文档通常也叫接口文档，更加注重于开发过程中的编写，在他们通常基于对代码的注释来自动生成API文档，而其中的集大成者应该算 jsdoc 吧，符合jsdoc语法，甚至于你可以利用jsdoc + js来代替Typescript的类型系统，也就是通过注释来实现ts的类型约束，据说svelte4.x开始这么干了。

当然，使用Typescript的话一般只需要使用基于 tsdoc  的规范去编写注释即可，而大多数的ts文档生成器，都兼容tsdoc规范。不过也可以使用jsdoc的标签（如果tsdoc没有的话），通常来说现代编辑器都支持识别jsdoc和tsdoc的注释语法。

所以，这一部分的重点在于开发在编写代码的过程中，针对代码的逻辑编写较为详细的注释，以便在生成API文档时，其文档能够尽量的详尽。

这里要多说一嘴，代码的注释，不仅仅体现在文档的生成，在你使用这个库或者自己在开发这个库时，其注释在编写代码时给与你提示，包含字段含义、接口参数等等，加快你的开发和对接成本

通常，为代码增加注释自然也是需要耗费不小的精力的，要说最规范的注释文档，你翻一下tsdoc或者jsdoc，他们有非常详细的标签：<https://www.jsdoc.com.cn/tags、https://tsdoc.org/pages/tags/alpha/>，你翻一翻这些标签的含义，基本上就能大致知道，他们的文档注释规范中，包含了哪些内容，例如：

- 介绍和详细说明
- 示例
- 默认值
- 是否内部使用
- 是否废弃
- 参数的含义和说明
- 返回值说明
- 抛出异常
- 外链
- API发布阶段：beta、alpha

还有很多，只要你想到的文档注释要包含的内容，基本上这些规范都帮你想到了，你想要真的写的完美、完善，只要按照jsdoc或者tsdoc的规范和标签去尽可能得补齐注释标签，这样必然是属于一份好的文档。

不过，我认为更重要的是**按照自身团队的规模和需求去选择性的补充需要补充的文档注释**，在SDK中，我认为有以下几个点是需要关注的：

- 区分内部模块、类型和方法，让文档只展现出对接者需要关注的内容，例如SDK中的工具方法自然就没有必要暴露以及出现在API文档中。@internal
- 对外暴露的class、方法、数据字段、事件、枚举通常都需要补充其简短描述
- 对于特殊方法、业务、使用方式等，需要补充较为详细的说明，也可以补充为何如此，以及一些对接者需要注意的事项 @remarks
- 标记废弃 @deprecated
- 得益于Typescript，其参数、返回值的类型我们是无需关心，而如果需要对参数字段和返回值做一些特殊说明，则可以添加@param（或者直接为参数添加简单描述）、@returns
- 如果是比较复杂的功能、模块或者机制以及示例，则会单独在使用文档或者开发文档中进行详细的描述，不太适合在注释中编写。

关于@example示例标签，我认为简单的没必要加，复杂的不适合加，给个示例链接反而更好。如果有合适的例子，后续再补充。

## 开发文档

这部分文档，通常是给开发人员了解整个项目用的，通常包含如下部分：

- 开发规范
- 架构设计
- 规范、一些约定
- 打包部署等

目的是为了让开发人员了解项目，以便进行对项目进行开发。这部分文档，放在项目中即可，无需对外暴露。

这个其实就是你平常写的设计文档啊、你进行评审的一些文档。

## 总结

如果你想让用这个库的人，更容易理解和使用这个库，你需要完善你的使用文档。

如果你想要让开发这个库的人，更容易上手开发，你需要完善你的开发文档，不过开发这种事，只有真正去写了代码，才算真正作为这个项目的开发者。

API文档是方便了开发同时也方便了使用这个库的人，现在基于ts可以很简单的生成出来，算是最容易写的，你只需要为你的接口方法和类型加个注释，就可以生成一份看起来还不错的API文档了。
