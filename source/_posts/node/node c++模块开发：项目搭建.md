---
title: node c++模块开发：项目搭建
date: 2022-05-12 17:26:47
tags:
  - node
  - node-gyp
  - node c++ addons
  - Clion 
categories:
  - node
---

## 背景

在你的业务中，通常在什么情况下会需要编写node c++ addons（即node原生模块或者c++模块）？

- 使用c++来提高提高性能
- 需要使用node来调用其他的c++库或者动态链接库

不管因为什么，编写一个node c++模块，所需要了解的知识可不少：

- 基本的node知识，这个不用多说
- 了解基本的c/c++语言，至少，你要能写c/c++代码，具体到什么程度就需要具体问题具体分析了。
- 了解node的[N-API](https://nodejs.org/api/n-api.html)或者[node-addon-api](https://github.com/nodejs/node-addon-api)库（推荐），这是架起node和c++之间的桥梁。

<!-- more -->

因为公司的一些需要，我研究了一下c/c++，并编写了一个封装第三方c++库的node c++模块，这这里我会记录一下我在编写这个node c++模块中学到以及思考的一些东西，我们这篇文章主要先尝试搭建一个node c++模块的项目。这里为了简化，它主要的功能是为了能调用一个第三方库中的API。

## 前期准备

### 开发环境

本系列文章中的项目在使用以下开发环境中编写和测试

- windows10（该项目不涉及到linux和mac）
- clion 2022.1（jetbrains出的c++开发工具）
- node 12.x
- c++项目环境(clion自带）：c++ 14版本，编译工具：msvs 142
- node-gyp和其编译环境：msvs 142，关于node-gyp环境的安装参考，参考之前的文章：[npm安装原生c++模块包的一些总结](https://www.ncdsf.com/2022/05/11/node/npm%E5%AE%89%E8%A3%85%E5%8E%9F%E7%94%9Fc++%E6%A8%A1%E5%9D%97%E5%8C%85%E7%9A%84%E4%B8%80%E4%BA%9B%E6%80%BB%E7%BB%93/)

### 说明事项

#### node c++模块编写方案

编写node c++模块，除了纯粹的c++代码之外，你还需要将你的c++代码提供的功能同node连接起来，主要有以下这几种方式：

- Node-API：以前叫做N-API，只不过现在名字变了，最新规范接口，也是建议使用的规范接口。它封装了底层js数据结构，并且它是符合ABI的接口，只要node所支持的ABI版本一致，那么只需要一次编译就可以在node中使用。即重点是ABI版本兼容。（因为它是一个C接口的API，所以其实这个写起来也挺麻烦）
- [node-addon-api](https://github.com/nodejs/node-addon-api)（推荐）：因为N-API是一个c的api规范，而使用c++可能会更加简单方便一点。所以，基于Node-API封装出了面向对象的接口。这样，编写原生模块更加简洁方便。这个模块仅仅是将c风格的N-API接口包装为了c++面向对象风格，其内部仍然是使用的N-API，所以，它能提供ABI稳定性保证。
- 如果N-API所提供的API接口，无法满足你的需求，那么你可以选择使用更加底层的V8、libuv、内部node等模块来编写你的c++模块。除此之外，更加推荐上面两种方式。
- NAN就不说了，已经弃了。

因node原生提供的Node-API对于目前我这个纯前端来说较为复杂，所以这里编写node c++插件选用的方案是`node-addon-api`库。

#### 使用的是clion而不是其他编辑器

这里我想要先强调一下这里所使用的开发环境，因为我使用的jetbrains的全家桶，且用的windows系统，所以，这个项目我会使用jetbrains的CLion来进行开发和调试，基本上它可以对本文中的所有功能做到开箱即用的程度。当然，你会想问，为啥不用visual studio？因为我不太熟悉他，而且，说是编写c++代码，但是我这里的项目搭建更多的是会偏向一个前端的node npm包的角度去考虑，而clion和visual studio搭建出的项目在工程上可能存在很大的不同，对于使用visual studio搭建项目的开发者来说本文的项目结构的参考性可能不大所以，所以需要提前强调一下，不过这里面对项目搭建的考量点应该还是可以借鉴的。

当然，这里还是还是要贴一下我Clion的构建设置的，防止后续埋坑。

使用visual studio进行编译：

![使用visual studio进行编译](https://image.ncdsf.com/2022/06/22/20220622155903.png)

构建自动reload CmakeLists.txt文件，且新增一个Release模式：

![使用Release模式](https://image.ncdsf.com/2022/06/22/20220622160111.png)

其他的差不多都是Clion的默认配置了。

### 简单的第三方库测试库：hello

我们编写一个最简单的用于测试的第三方hello库，只有一个最简单的API：根据传递的name字符串，获取“hello name“字符串。这主要是为了体现出使用了第三方库时的项目结构，比如c/c++的头文件放哪，如何引用lib/dll资源等等。

库的代码很简单，如下：

hello.h

```c++
#ifndef HELLO_HELLO_H
#define HELLO_HELLO_H

#include <iostream>
using namespace std;
string getHelloStr(const string& name);

#endif //HELLO_HELLO_H
```

hello.cpp

```c++
#include "hello.h"

#include <iostream>
using namespace std;

string getHelloStr(const string& name) {
    return "hello " + name;
}
```

然后使用msvs进行构建，得到一个hello.lib文件。后续，我们会在node c++模块中封装该库文件中的API，并暴露给node层面使用。

> 此处构建出的是Release x64位版本的。

## 项目搭建

所有的一切都准备好了，我们开始正式搭建我们的node c++项目吧。

> 注意，一些不重要的或者很常见步骤的细节方面，我会省略掉，比如需要安装某个npm包，我会直接说明安装包的名字，但是会省略掉安装步骤。

### 搭建c++项目

#### 创建c++可执行项目

我们使用Clion创建一个c++的可执行项目

![使用clion创建c++可执行项目](https://image.ncdsf.com/2022/06/21/20220621114706.png)

得到一个基本的项目结构，且可以直接运行：

![简单的可执行c++项目](https://image.ncdsf.com/2022/06/21/20220621133951.png)

然后记得初始化git仓库并连接到你们的git托管服务（不要忘了.gitignore)，这里就不展示细节了，现在我们来对它进行改造。

> 项目中的`cmake-build-*`是Clion自行构建出的项目中间产物。

#### c++相关内容的目录划分

我们创建一些诸如lib、include、addons等目录，并将一些资源进行简单的划分：

- addons：用于存放node c++模块相关的c++部分的源码。比如将main.cpp文件移动到该目录下了。
- lib：用于存放第三方c/c++的一些静态库（比如.lib文件）这里将我们之前用于测试的hello.lib库放进来了
- include：存放c/c++头文件，这里把用于测试的hello.h头文件放进来了

![c++相关内容的目录划分](https://image.ncdsf.com/2022/06/21/20220621140124.png)

然后，我们为了让项目可以使用我们编写的hello库，我们需要修改CMakeLists.txt文
件，以引用我们的头文件以及链接我们的hello.lib库文件:

CMakeLists.txt

```cmakelists
cmake_minimum_required(VERSION 3.22)
project(node_addons_template)

set(CMAKE_CXX_STANDARD 14)

# 添加一些头文件
include_directories(include)

# 设置需要链接的lib库文件所在的目录
link_directories(lib)

add_executable(node_addons_template addons/main.cpp)

# 设置需要链接的lib库文件，因为设置了link_directories，所以这里的目录可以不用以 lib\\ 开头
target_link_libraries(node_addons_template hello.lib)
```

正常来说，Clion会监听CMakeLists.txt文件的修改（如果你按照上面的Clion设置的话），否则你需要重新reload一下该项目，不然CMakeLists.txt的修改是不会生效的。

然后，我们修改main.cpp文件，使用我们的hello库：

```cpp
#include <iostream>
#include "hello.h"

using namespace std;

int main() {
    string hellString = getHelloStr("Li");
    cout<<hellString<<endl;
}
```

运行查看结果：

![运行结果](https://image.ncdsf.com/2022/06/21/20220621144249.png)

看输出，我们已经可以正确的使用hello库了。此时，c++部分的项目结构就已经搞定了，接下来，我们看一下我们熟悉的前端node部分的项目结构改怎么搭建吧。

### 搭建node项目

我们编写一个node c++模块，可不仅仅需要编写c/c++代码，还需要编写node部分的代码，且需要将我们编写的node c++模块正确的发布到npm仓库上，供其他项目使用，这里我们就来搭建node部分的项目结构。

#### 以npm工具包的角度去搭建项目

其实这一部分，和搭建一个npm工具包项目非常类似，其实对于node c++模块的使用者来说，它就是一个npm包，这里我们就按照npm工具包的形式去搭建即可。

同样在这个c++可执行项目中，执行以下步骤：

- 使用你喜欢的包管理器，初始化package.json，我这里用的pnpm。并补全相关的package.json信息（比如name，description，keywords等等）
- 创建src目录作为你编写JavaScript相关代码的根目录
- 添加相关的依赖
  - 使用的是ts编写JavaScript代码
  - 使用rollup来进行打包
  - 使用jest进行单元测试
  - 使用typedoc创建API文档
  - 使用eslint、prettier等规范代码（当然，你还可以添加husky、lint-staged等其他辅助包）
  - 安装node-addon-api、bindings库（用处后面会说到）

总之，这一部分你就可以看成在搭建一个npm工具库项目，他并没有太多的很新鲜的东西，因为对于node这一层的代码来说，你就是在用ts编写一个普通的npm库：

![以npm包的方式搭建](https://image.ncdsf.com/2022/06/21/20220621145310.png)

Clion的一个好处就是，它开箱即用的支持web开发相关的内容，无需额外做配置，至少对于开发node c++模块来说，Clion对web开发的支持已经足够了（毕竟此时你不用写react、vue这种）

至此，node c++模块的项目搭建，算是已经完成一半了，你还需要完成node代码和c++代码的连接。这一部分可就没那么容易了。

## 连接node和c++

我们在一开始，选用了node-addon-api库作为我们编写node c++模块的编写方案，关于这一个库的编写和使用，后续有机会我会编写另一篇文章进行介绍，这里只是简单的编写示例代码，毕竟，主要还是为了项目的搭建嘛。

### 使用node-addon-api编写暴露给node层的API接口

我们编写出来的c++代码，会被编译为一个`.node`文件，实际上它就是一个动态链接库，且他需要用到node-addon-api库的头文件或者库，我们需要对该项目的c++部分进行改造。

首先在之前的addons目录下新建一个`addons.cpp`文件，删除掉main.cpp文件，因为此时main.cpp以及相关的CMakeLists.txt中的内容已经无用了，因为这一部分代码不会通过这个CMakeLists.txt中定义的配置去构建。

然后，在addons.cpp文件中添加以下代码：

```cpp
#include <iostream>
#include "napi.h"
#include "hello.h"

// 声明一个函数，此函数封装了hello库的getHelloStr方法
Napi::String helloFn(const Napi::CallbackInfo& info) {
    Napi::Env env = info.Env();
    // 获取参数
    Napi::String name = info[0].As<Napi::String>();
    return Napi::String::New(env, getHelloStr(name.ToString()));
}

// node c++模块的入口函数
Napi::Object Init(Napi::Env env, Napi::Object exports) {
    // 将helloFn函数，作为exports对象的hello属性，暴露给node层，此时用户require()这个构建出来的.node文件后，就得到一个含有hello方法的对象。
    // 这和你在node中的module.exports = { hello: function () { xxx } } 是一样的。
    exports.Set("hello", Napi::Function::New(env, helloFn));
    return exports;
}

// "node_addons_template" 是模块名称
NODE_API_MODULE(node_addons_template, Init);
```

上面的代码利用node-addon-api库的API，将封装了hello库的getHelloStr方法的函数暴露给了node层，这样当你在通过require引用这个包时，export导出的API中就会一个hello的方法，js这边可以直接调用。

下面，我们就来看看，如何将这里的c++代码，编译为可被node导入的.node形式吧

> 此时，上面的代码可能会找不到一些定义，没关系，这是正常的，后面的优化中我们会提到。

### 编译c++代码并在node层调用

编译上面的c++代码，我们需要用到node-gyp库，这在上面也提到过，而这里，我们首先要做的是编写`binding.gyp`配置文件，这个配置文件是提供给node-gyp库来描述我们的c++代码的编译选项的。这也是为什么我们项目中的那个CMakeLists.txt文件的配置目前来说不需要的原因所在，我们最终编译出node c++模块其实是通过node-gyp和binding.gyp去实现的。

#### 配置binding.gyp

binding.gyp

```json
{

  "targets": [
    {
      "target_name": "node_addons_template",
      "sources": [
        "./addons/addons.cpp",
      ],
      "cflags!": [ "-fno-exceptions" ],
      "cflags_cc!": [ "-fno-exceptions" ],
      "include_dirs": [
        "include",
        "<!(node -p \"require('node-addon-api').include_dir\")"
      ],
      "defines": [
        "_HAS_EXCEPTIONS=1",
      ],
      "conditions": [
        ['OS=="win"', {
          "variables": {
            "PROJECT_ROOT": "<!(node -p \"process.cwd()\")"
          },
          "configurations": {
            "Release": {
              "msvs_settings": {
                "VCCLCompilerTool": {
                  "ExceptionHandling": 1,
                  "RuntimeLibrary": '2',
                },
              }
            },
            "Debug": {
              "msvs_settings": {
                "VCCLCompilerTool": {
                  "ExceptionHandling": 1,
                  "RuntimeLibrary": '3',
                },
              }
            }
          },

          "libraries": [
            "<(PROJECT_ROOT)\\lib\\hello.lib",
          ],
        }]
      ],

    }
  ],
}
```

binding.gyp文件，其实就是和CMakeLists.txt文件的作用差不多，都是描述c++编译选项的，而我们将c++编译为.node文件，其实靠的node-gyp以及binding.gyp配置选项来做到的，和CMakeLists.txt中的配置没有任何关系，比如，我们在binding.gyp中依然需要配置libraries来加载我们的hello.lib库，同时我们也配置了include_dirs来将我们的头文件包含进去，这和我们在CMakeLists.txt配置相关内容是一样的。

我们根本都不需要使用CMakeLists.txt来构建出库或者可执行文件。不过，我们后续还是会使用CMakeLists.txt并配置它，原因我们后面再说。

#### 添加相关的script命令

当你配置好了binding.gyp之后，你可以在package.json中添加脚本来编译他们：

```json
{
  "scripts": {
    "build": "rimraf dist && rollup -c rollup.config.js",
    "docs": "typedoc src/index.ts",
    "install": "yarn run addons-rebuild",
    "prepublishOnly": "yarn build",
    "configure": "node-gyp configure",
    "addons-build": "node-gyp build",
    "addons-rebuild": "node-gyp rebuild"
  }
}
```

这里的脚本命令有些多，其中build是编译ts用的，docs是用于API文档，install和prepublishOnly是发布和安装时的钩子，configure、addons-build、addons-rebuild则是用于构建node c++模块的命令了，他主要运行的是node-gyp中的命令。

通常来说，你直接运行addons-rebuild即可，这时候应该会构建成功，且在你的根目录下输出一个build文件夹，且里面的Release就包含了构建出来的.lib和.node文件：

![addons-rebuild编译结果](https://image.ncdsf.com/2022/06/22/20220622102233.png)

> 关于node-gyp的命令，请参考：[node-gyp](https://github.com/nodejs/node-gyp)

关于install这个script，他会在这个包的被安装时执行，以便使用node-gyp进行构建，但是，npm对此有一个特殊处理，如果你的包根目录下拥有binding.gyp文件，且你又没有定义自己的install或preinstall脚本，npm默认会帮你运行：`node-gyp rebuild`。

> 注：这里都是构建的Release版本的，你可以添加 --debug选项来使用Debug版本的

#### 引用编译后的.node模块

我们知道，node的require方法是可以直接引用.node文件的，这也是node提供给我们在js中加载c++模块的方法，这里我们在src目录中创建一个用于测试addons模块的测试文件夹test，并创建一个测试文件：

base.test.ts

```ts
import path from 'path'
// 获取c++模块所暴露出的接口（这里它应该暴露出了一个hello方法）
const addons = require('../../build/Release/node_addons_template.node')

describe('基础测试', function () {
    it('API导出测试', function () {
        expect(Reflect.has(addons, 'hello')).toBeTruthy();
    });
    it('测试hello方法', function () {
        const helloStr = addons.hello('Li');
        expect(helloStr).toBe('hello Li');
    });
});
```

然后运行该测试：

![测试结果](https://image.ncdsf.com/2022/06/22/20220622103516.png)

当然，这里的require获取.node文件不是那么的优雅，一大串的路径，这里我们可以使用一个名为[bindings](https://www.npmjs.com/package/bindings)的包，这个包专门用来帮你加装c++模块，我们将其抽离为一个工具方法：

src/utils/index.ts

```ts
import bindings from "bindings"

// 需要加载的c++模块名称
const ADDONS_NAME = 'node_addons_template'

/**
 * 加载封装的node addons sdk
 */
export function loadAddons() {
    return bindings(ADDONS_NAME);
}
```

然后替换上诉base.test.ts中的部分代码：

```ts

import { loadAddons } from "../utils";

const addons = loadAddons();

describe('基础测试', function () {
    // 略
});
```

至此，我们完成了一个基本的node c++模块的开发，下一步，则是准备将其发布到npm包中了。

### 发布npm包

我们之前配置的package script脚本中，有一个build和一个prepublishOnly命令，build则是打包我们编写的ts部分的源码的，而prepublishOnly则是在使用npm publish时，预先调用build命令构建出所需资源。

我们目前的src/index.ts下的代码很简单，就是直接导出加载的c++模块：

```ts
import { loadAddons } from "./utils";

export default loadAddons()
// import addon from '../index'
```

然后，你可以运行build命令进行打包，得到dist文件夹，而我们整个模块的main入口，则是设置为dist目录下的index.js，package.json配置为：

```json
{
  "main": "dist/index.js",
  "module": "dist/index-es.js",
  "types": "dist/index.d.ts",
}
```

最后，我们完善下`.npmignore`文件，把不需要发布到npm包中的资源给忽略掉，然后运行：`npm publish`即可（关于发布到npm包的一些细节，这里不再赘述，它和发布普通的npm包没啥差别）

## 总结

至此，我们从搭建c++项目，到搭建npm包项目，并逐渐将他们完善，以此完成了一个完整的node c++项目搭建，到这里，你可以获得一个可以跑起来的node c++模块项目了，但是，这远远不是最终答案，在下一篇[node c++模块开发：项目优化](/2022/06/22/node/node%20c++模块开发：项目优化/)文章中，我将解决我在编写c++模块时遇到的一些问题，并对相关的配置进行了一些优化，比如添加项目中CMakeLists.txt文件的配置、回答为什么要使用rollup和ts，以及当项目依赖很大的额外资源时如何处理npm包的发布和安装等等。

最后，整个项目的模板代码，我放到了github上了，可以作为一个参考：[node-addons-template](https://github.com/nongcundeshifu/node-addons-template)
