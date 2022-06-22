---
title: node c++模块开发：问题记录
date: 2022-05-22 17:26:47
tags:
  - node
  - node-gyp
  - node c++ addons
categories:
  - node
---

这里记录了一些，我编写node c++模块时，所遇到的一些问题，所以记录下来，而且肯定会不定期更新。

<!-- more -->

## node-gyp编译错误

### 使用 /GL 编译的模块警告

警告：`找到 MSIL .netmodule 或使用 /GL 编译的模块；正在使用 /LTCG 重新启动链接；将 /LTCG 添加到链接命令行以改进链接器性能`

解决：如果是用的msvs构建出来的，那么可以在visualstudio的解决方案中，点击属性，在 c/c++ 选项中的优化选项面板中，找到全局程序优化，从`是(/GL)`设置为否

### RuntimeLibrary不匹配错误

错误：报错 error LNK2038: 检测到“RuntimeLibrary”的不匹配项: 值“MD_DynamicRelease”不匹配值“MT_StaticRelease”

解决：如果是类似错误，则，你可以在visualstudio的解决方案中，点击属性，在 c/c++ 选项中的代码生成选项面板中，找到运行库，然后设置他们。但是通常来说，库这边全部使用默认的即可。你只需要在binding.gyp配置文件中，修改一下配置即可，例如：

```json
{
    "conditions": [
      ['OS=="win"', {
        "configurations": {
            "Release": {
                "msvs_settings": {
                "VCCLCompilerTool": {
                    "ExceptionHandling": 1,
                    "RuntimeLibrary": '2', // 主要是这个
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
        }]
    ],
}
```

### debug引用了release或者release引用了debug的库

错误：error LNK2038: 检测到“_ITERATOR_DEBUG_LEVEL”的不匹配项: 值“0”不匹配值“2”

这本质是以Debug模式运行的代码引用了Release构建的库，或者反过来（看值的匹配），这里只需要正确的引用同样模式的库即可，即Debug引用Debug构建的库，Release引用Release构建的库。

### c++模块引用动态链接库

### bindings错误： The specified module could not be found

如果你的原生c++模块中加载了动态链接库，而这个动态链接库搜索不到时，则可能会抛出这个错误。可以使用检查dll的方式来检查`.node`文件，工具：[dependencywalker](http://www.dependencywalker.com/)

## windows下构建问题

### 换行符

如果在构建时，拉取下来的代码换行符风格不一致，比如在windows下，c++源文件是unix风格的，那么编译时就会有问题。

### 缺失Microsoft Visual C++ Redistributable

报错：

`The application has failed to start because its side-by-side configuration is incorrect. Please see the application event log or use the command-line sxstrace.exe tool for more detail.`

可能是因为你所引用的库依赖了Microsoft Visual C++ Redistributable库，但此电脑上缺失，你可以通过windows应用程序界面，搜索`Redistributable`如下：

![本机安装的Redistributable库](https://image.ncdsf.com/2022/05/19/20220519175223.png)

然后去下载相关的[Redistributable](https://docs.microsoft.com/en-US/cpp/windows/latest-supported-vc-redist?view=msvc-170)，然后安装即可。
