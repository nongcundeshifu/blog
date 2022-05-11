---
title: npm安装原生c++模块包的一些总结
date: 2022-05-11 14:53:24
tags:
- node
  - node-gyp
  - node c++ addons
categories:
  - node
---

我们知道，npm中有些包是使用了原生的c++进行编写，在安装这类包时，都会使用node-gyp库来对这些c++模块进行编译以便可以使其在node中使用。而我们大部分遇到这类原生c++模块包的安装时，都会非常头疼，因为太容易安装失败了。所幸，在吃了这么多亏之后，终于将安装这类c++模块的解决方案大致摸清楚了，所以记录下来，希望能帮到别人。

## 前置说明

- 一些基本的node、npm等相关背景知识，比如npm如何安装包这种知识在此就不再赘述了。
- 因为我自己的开发环境是windows，所以，这篇文章的内容只涵盖了在windows上安装原生c++模块的方案

如果想要在安装原生c++模块时，有较好的体验（即大部分原生c++模块都能安装成功），除了需要安装好node本身的node-gyp及相关的编译工具之外，以下几点也是我认为比较重要的：

- 你需要准备一个代理，我用的代理工具是 [ShadowsocksR](https://github.com/HMBSbige/ShadowsocksR-Windows)，如果没有代理进行科学上网，下面的步骤可能都会卡主，而如果再配上[proxifier](https://www.proxifier.com/)这种全局代理工具，基本网络这块不是阻碍了。
- 配置一些原生c++模块的安装源

> ps：安装相关的编译工具比较耗时，下面这几种方式各位需要对其有点耐心。

## 安装node-gyp及相关的编译工具

在此之前，我一直都怕了安装需要node-gyp模块编译的npm包了，同时也怕了安装node-gyp本身了，配一个node-gyp能正常工作的环境挺难麻烦的，但是，该装的还是要装的，环境该配的还是要配的，因为编译c++模块就靠它了。

截止目前（2022.5月），我所了解的配置好一个node-gyp编译环境，可以通过以下几个方法：

- 通过“适用于 Windows 的官方 Node.js 安装程序”来安装node-gyp相关的编译环境
- 通过`windows-build-tools`包（注意：从仓库文档上来看，也不推荐了）
- 手动安装依赖和配置环境：手动安装Visual C++ 构建工具、python并对其进行相关的配置

更多安装信息，可以参考：[node-gyp官方文档](https://github.com/nodejs/node-gyp)

### 通过“适用于 Windows 的官方 Node.js 安装程序”来安装node-gyp相关的编译环境

如果你是第一次为你的电脑安装node，且没有使用像`nvm`、`n`这些node版本管理器来安装node的话，那么这是目前最方便的方案。只需要下载适用于 Windows 的官方 Node.js 安装程序（比如msi安装程序），并且在安装时，`勾选“Automatically install the necessary tools. xxx”`选项即可：

![适用于 Windows 的官方 Node.js 安装程序](https://image.ncdsf.com/2022/05/07/20220507152000.png)

![勾选Automatically install the necessary tools](https://image.ncdsf.com/2022/05/07/20220507153417.png)

后续node安装程序会自动弹出一个命令行程序来安装node-gyp相关的必要工具（python、Visual studio build tools），此外可能还会包含一些其他的工具：

![命令行程序安装过程](https://image.ncdsf.com/2022/05/07/20220507152757.png)
![命令行程序安装过程](https://image.ncdsf.com/2022/05/07/20220507152804.png)
![命令行程序安装过程](https://image.ncdsf.com/2022/05/07/20220507152812.png)

安装完毕后，命令行程序会自动关闭。此时，你可以打开一个新的powershell，尝试输入`node -v`和`npm -v`来验证node是否安装正常。当然我们重点还是要验证node-gyp的。但是node-gyp还是需要你自己安装的，运行：`npm i -g node-gyp`，等待安装完毕。

验证：尝试安装一个node原生模块，看node-gyp相关的编译环境是否正常，比如：

- `npm i -g ffi-napi`

此时，你应该可以得到以下结果：

![安装node原生模块](https://image.ncdsf.com/2022/05/10/20220510114926.png)

安装上面两个包成功，基本上说明你已经拥有一个完好的node-gyp编译环境了。

如果你已经安装过node了（通过官方node安装程序来安装的），那么你可以再次运行该安装程序，并且，选择点击"change"按钮来重新修改安装选项，然后勾选“Automatically install the necessary tools”来安装相关工具：

![change](https://image.ncdsf.com/2022/05/09/20220509111755.png)

![勾选Automatically install the necessary tools](https://image.ncdsf.com/2022/05/07/20220507153417.png)

> 注意：如果你本地安装的node版本和新的node安装程序版本不一致，比如下载了个更加新的版本，则等于重新安装node了，比如我本机安装了12.x的，打开一个14.x的msi安装程序，则等价于重新安装node为14.x版本的。此时，你仍可以勾选“Automatically install the necessary tools”来安装相关node-gyp编译环境。

#### 使用nvm来管理node版本时借助官方node安装程序来安装相关node-gyp编译环境

如果你使用的是nvm这种node版本管理器工具来安装的node，那么你仍然可以借助windows 官方node安装程序来安装相应的node-gyp编译环境，对于nvm-windows来说，你可以通过`nvm off`和`nvm on`来关闭和打开node版本管理（以nvm-windows-v1.1.9作为测试），在关闭nvm的情况下，你可以使用官方node安装程序来安装node的同时，安装相关node-gyp编译环境，在安装相关node-gyp编译环境结束后，通过官方node安装程序卸载本机的node，然后重新打开nvm即可。

> 对于其他的node版本管理器来说，应该也可以应用同样的方式，先关闭掉node版本管理器使用的node，然后借助官方node安装程序来安装node-gyp相关的编译环境后，之后卸载node，重新使用node版本管理器来启用node。反正你的目的是为了安装node元素模块依赖的编译环境。

### 通过windows-build-tools包安装相关环境

更多安装细节，请参考：[windows-build-tools github](https://github.com/felixrieseberg/windows-build-tools)

因为目前适用于 Windows 的官方 Node.js 安装程序已经可以自动安装node-gyp所需要的工具了，所以，目前该仓库已经被作者存档了。不过，在官方安装程序支持之前，该仓库一直都是安装复杂的node-gyp编译环境的最好方式。使用它很简单，只需要打开`管理员`权限的powershell，然后运行：`npm install -g windows-build-tools`等待安装完毕即可：

![npm install -g windows-build-tools](https://image.ncdsf.com/2022/05/09/20220509105239.png)

而且，当你已经安装过node，且使用的是像nvm、n这种包管理器来管理你的node版本时，那么这个工具仍然是有用的，不过你需要注意的是，该仓库已经被`归档`了。而且，你可能会安装失败，我重试了几次才成功。所以，这种方式现在不是那么推荐了。

### 手动安装node-gpy编译环境

如果你需要开发node c++原生模块，那么手动安装相应的开发环境也是个不错的选择。而且，它基本上，不会怎么出错。其实我个人更加推荐这种方式的。

#### 安装Visual Studio C++构建环境

node-gpy只是用于编译出能在node中运行的c++模块，编译c++这活还得c++构建环境去做的。

1. 下载官网 [Visual Studio 安装程序](https://visualstudio.microsoft.com/zh-hans/thank-you-downloading-visual-studio/?sku=Community)，默认会下载最新版本（应该是2022），而我的windows 1903似乎不兼容该版本，所以，我这里以2019为例（2017、2019都可以）。下载[Visual Studio Community 2019](https://www.techspot.com/downloads/7241-visual-studio-2019.html)
2. 打开安装程序，通常来说建议选择Visual Studio Community，即社区版，因为社区版的Visual Studio编辑器免费，而开发环境则需选择：`使用c++的桌面开发`。然后安装默认环境并等待安装完毕即可。如下图：
3. 待安装完毕后，设置npm配置： `npm config set msvs_version 2019`。

下载Visual Studio Community 2019：

![Visual Studio Community 2019](https://image.ncdsf.com/2022/05/10/20220510110127.png)

安装：

![使用c++的桌面开发](https://image.ncdsf.com/2022/05/10/20220510093958.png)

安装完成：

![安装完成](https://image.ncdsf.com/2022/05/10/20220510100544.png)

#### 关于python

node-gpy可能需要你安装python的兼容版本，你可以直接把[python3.10](https://www.python.org/downloads/release/python-3104/)装上就行。安装时你记得选择将其添加到环境变量。

测试python是否安装完成，打开控制台，输入：`python`，看是否输出正常。

测试成功后，此时你就可以尝试安装一个包来测试你的node-gyp编译环境是否正常了：`npm i -g ffi-napi`

> 如果仅仅安装python并且添加环境变量后安装c++原生模块包还是失败，则尝试执行：`npm config set python /path/to/executable/python`，其中`/path/to/executable/python`指你python可执行程序所在的路径

## 其他总结

现在有些c++原生模块的npm包，他在发布新版本时，就会将各平台的二进制node文件先提前编译出来，然后在用户安装时，直接下载相应的二进制文件即可，这样用户在安装时避免了其重复编译，比如`node-sass`这个包我试了下发现就是类似这样操作的。

### npm包安装很慢

有些npm包安装时，速度很慢，这除了npm安装源的问题，还可能是由于有些包在安装过程中，会下载一些其他资源，比如electron、canvas、sharp这些包，他们在安装过程中，都会额外下载这些包自身依赖的其他资源，而这些下载链接呢，可能存在资源404，或者因为网络问题，资源下载失败的情况，这些都会导致包安装失败。

当然，出现这种情况，大部分流行的包都有解决方案了，要么配置npm源为国内的淘宝源之类的，要么使用类似：[mirror-config-china](https://github.com/gucong3000/mirror-config-china)这种包来修改.npmrc或者.yarnrc配置，将一些包依赖的下载链接配置成可用的或者国内的链接。有兴趣的，可以了解下mirror-config-china这个仓库做了什么。

## 总结

本文总结了安装node-gyp编译环境的各种方案，通过上述的方案，相信你能不再为安装node原生c++模块而烦恼。

本文完！
