---
title: node c++模块开发：项目优化
date: 2022-06-22 17:26:47
tags:
  - node
  - node-gyp
  - node c++ addons
categories:
  - node
---

我在上一篇文章中[node c++模块开发：项目搭建](/2022/05/12/node/node%20c++模块开发：项目搭建/)文章中，一步一步搭建了一个基本的node c++模块项目。但是，那个项目还存在着一些问题，比如对c++部分的测试，ts类型定义（不然用ts干啥），API文档等等。这篇文章，我将对尝试解决这些问题，并对项目的配置做进一步的优化。

<!-- more -->

## Clion中的c++代码提示

上一篇文章中，我们说明了如何配置binding.gyp来让node-gyp构建我们的c++模块，同时我们也解释了项目中的CMakeLists.txt配置对于最终的node-gyp编译是不起作用的（因为node-gyp用的根本不是这个配置）。但是，我们仍然要配置CMakeLists.txt文件，主要是有以下几个原因：

一个是为了充分利用IDE的优势，这一点很重要。因为如果没有配置CMakeLists.txt，那么你编写的c++源码，IDE是无法进行语法提示，更不要说对其进行类型检查和错误分析的了，Clion并不能识别binding.gyp中的配置。所以，即使我们最终不使用CMakeLists.txt来进行编译，我们也要将我们的c++源码利用CMakeLists.txt进行配置。

另一个，则是对于c++部分的单元测试，比如我们自行封装的第三方库的那一部分代码，下文会添加google test来弥补这一部分的缺失，所以，这里必然还是需要配置CMakeLists.txt的。

下面我们就来配置CMakeLists.txt，让IDE能正确的识别c++源码。

## 在c++中，添加对node-addon-api库的支持

- 上文提到，我们的c++模块源码写在了addons/addons.cpp文件中，我们需要将其添加到CMakeLists.txt文件中，这里用add_library或者add_executable都可以。
- 其次，我们之前通过npm安装了node-addon-api库，而这个库中包含了一些头文件（在`node_modules\\node-addon-api`目录中），我们需要在CMakeLists.txt中引用这些头文件
- 最后，还需要将node-gyp中和node相关的一些头文件包含进来。

更新CMakeLists.txt

```CMakeLists
cmake_minimum_required(VERSION 3.22)
project(node_addons_template)

set(CMAKE_CXX_STANDARD 14)

# 添加一些头文件，TODO：这里的xxxxxxxxxx路径需要自行补齐。
include_directories(include C:\\Users\\xxxxxxxxxx\\AppData\\Local\\node-gyp\\Cache\\12.22.7\\include\\node node_modules\\node-addon-api)

# 设置需要链接的lib库文件所在的目录
link_directories(lib)

add_library(node_addons_template addons/addons.cpp)

# 设置需要链接的lib库文件，因为设置了link_directories，所以这里的目录可以不用以 lib\\ 开头
target_link_libraries(node_addons_template hello.lib)
```

> 注意，这里面的`C:\\Users\\xxxxxxxxxx\\AppData\\Local\\node-gyp\\Cache\\12.22.7\\include\\node`，其实是安装node-gpy时，它为当前的node版本生成的一些头文件，node-addon-api是需要它们的。通常指的是：`C:\\Users\\{user}\\AppData\\Local\\node-gyp\\Cache\\{node_version}\\include\\node` 这个目录

而binding.gyp中，则不需要添加这个node相关的类型定义，因为node-gyp应该是自行添加了。

## c++代码测试

我们在node层面，通过jest来测试我们的代码，这一部分的测试我们可以涵盖到：测试加载c++模块是否正常，测试c++模块暴露的API是否符合预期，测试其他js代码等等。但是，我们对于c++部分的测试，却无法涵盖，而且，如果仅仅只有node层面的测试，那么如果问题出现在c++层面，那么我们的调试以及定位问题就会变得非常麻烦，这里我考虑对c++部分添加google test，以实现对c++部分代码的单元测试。

在根目录下新建一个addons/test/bash.test.cpp文件，然后我们需要修改CMakeLists.txt文件，在文件下方添加如下代码：

```txt
# ------添加google test
# 下载并配置google test依赖
include(FetchContent)
FetchContent_Declare(
        googletest
        URL https://github.com/google/googletest/archive/e2239ee6043f73722e7aa812a459f54a28552929.zip
)
set(gtest_force_shared_crt ON CACHE BOOL "" FORCE)
FetchContent_MakeAvailable(googletest)

# 构建googletest代码，使用了google test，则不需要main文件了
enable_testing()

# 添加需要测试的测试文件
add_executable(addons_test addons/test/bash.test.cpp)

# 把需要链接的hello库添加进来，gtest_main是不能去掉的
target_link_libraries(
        addons_test
        gtest_main
        hello.lib
)

include(GoogleTest)
gtest_discover_tests(addons_test)
```

然后再bash.test.cpp文件中添加如下代码：

```c++
#include <iostream>
#include "gtest/gtest.h"
#include "hello.h"

TEST(baseTest, hello) { // NOLINT(cert-err58-cpp)
    auto helloStr = getHelloStr("Li");
    EXPECT_EQ(helloStr, "hello Li");
    std::cout<<"test pass"<<endl;
}
```

此时，就可以运行测试了：

![测试结果](https://image.ncdsf.com/2022/06/22/20220622144709.png)

> 注意，google test只能测试第三方库或者封装第三方库的那些c++代码，它`无法测试包含node-addon-api库的那些c++代码`，即和node相关的那些c++代码是无法进行测试的，因为无法同node一样提供相关环境。目前我还在尝试找到解决方案。

### 添加ts类型定义

我们之前通过bindings库来加载我们的node模块，此时获取到的是一个any对象，我们为了给其他人使用该库，还需要为我们node c++模块所暴露出的API，编写类型定义：

src/types/NodeAddonsTemplate.ts

```ts
/**
 * 为node c++模块暴露的API编写类型定义
 */
export type NodeAddonsTemplate = {
    hello: (name: string) => string
}
```

修改我们之前编写的loadAddons方法（src/utils/index.ts），添加类型

```ts
import bindings from "bindings"
import { NodeAddonsTemplate } from "../types/NodeAddonsTemplate";

const ADDONS_NAME = 'node_addons_template'

/**
 * 加载封装的node addons sdk
 */
export function loadAddons(): NodeAddonsTemplate {
    return bindings(ADDONS_NAME);
}
```

此时，我们在通过loadAddons方法加载我们的node c++模块时，其获取的就是一个`NodeAddonsTemplate`类型的对象，此时相关的API就具有了类型提示，且增强了重构能力，这也是为何单独为加载node c++模块添加一个方法的原因。

### 关于二次封装node c++模块的考虑

可能有些人会疑惑，为什么需要添加ts和rollup打包来增加项目的复杂度？有什么必要吗？

我在编写c++模块时，我发现了问题，因为我所编写的c++模块，其主要是封装某个第三方库的API，而我发现使用node-addon-api库所其暴露出的接口不太能满足我们业务的需求（比如直接封装的第三方库的API可能比较原始)，也许你可能想说，为什么不直接在c++那一层就根据业务去封装好呢？然而现实是，封装js代码比封装c++要简单（因为说到底，对于前端来说js肯定更熟悉），我可以接受从c++暴露出的API使用回调函数去处理异步（因为对于js的非回调异步，c++那边代码写起来并不那么容易理解），但是最好是将这些API在js那边再次封装成promise的形式。

基于上面的原因，node方面的代码肯定不是直接通过一个bindings去加载c++模块然后导出去就可以搞定的了。肯定需要额外的代码，那么ts和rollup就变得有必要，而且，顺带还可以把接口的类型定义给做了。并且，还可以通过typedoc生成相关的API文档。

当然，如果你不需要二次封装，那么可以不用rollup打包、不用ts，而是自己编写API文档，但是我仍然建议你为你的库编写类型定义文件（.d.ts），这对于库的使用者来说，可能很重要。

### 额外资源

我们在发布这个npm包时，是需要将我们依赖的lib提交到npm上面去的，或者lib太大，且包含了一些其他资源（比如动态链接库），那么你可以自定义脚本，通过install钩子脚本，在这个npm包被下载后，将相关的资源从远端拉取下来，放到这个包的目录中。

比如，我的lib下的资源太大，我发布npm包时，不将其提交到npm上，我将这些lib资源作为压缩包放到我的服务器上，然后我在项目根目录下新建一个`install/loadSources.js`文件，然后在该文件下编写你拉取相关资源的代码。最后，在install这个script脚本中跑起来即可：

package.json

```json
{
  "scripts": {
    // ...
    "install": "yarn run addons-rebuild && node install/loadSources.js",
  },
}
```

## 总结

关于项目本身的优化，目前就想到了这么多，主要都是我在编写node c++模块过程中，遇到过的，当然，关于编写node c++模块远远没有那么简单，也肯定还有很多问题和挑战存在，只不过自己还没有涉及罢了，任重而道远呐。
