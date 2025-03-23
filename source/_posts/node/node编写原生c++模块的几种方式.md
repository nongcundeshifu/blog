---
title: node编写原生c++模块的几种方式
date: 2022-10-22 18:23:40
tags:
  - node
  - node-gyp
  - node c++ addons
categories:
  - node
---

本次我们了解一下通常在编写node原生模块时，都有哪些方式，并简单说明一下它们的特点。

<!-- more -->

## 什么是node c++原生模块

node c++原生模块，也叫node插件，本质上它是用c/c++编写的动态链接库，可以被node的require方法引入并使用的node模块。插件提供 js 和 C/C++ 库之间的接口。

## node-gyp

node原生模块（.node文件）本质是使用c/c++代码编写的一个动态链接库，既然是c++编写的，自然就需要对其进行编译了，而目前基本上是使用node-gyp编译出来，而node-gyp本身是基于gyp这个工具而来的。

那gyp是什么？我们知道，编译c/c++项目的工具有很多，有跨平台的cmake，有各平台自己的工具，比如windows的vs，mac的xcode，gyp的最初是为chromium项目创建的生成工具，他通过一个`binding.gyp`配置文件来为chromium生成不同平台下的编译配置（不过现在用的是gn了）文件，比如在windows下，gyp会根据`binding.gyp`生成一份Visual Studio的编译配置文件，mac平台下回生成一份xcode编译配置文件。

而node-gyp基于gyp，自然也拥有gyp的这些功能，不过node-gyp它还做了一些额外的事情。比如，在我们编译一个 C++ 原生扩展的时候，它会去指定目录下（通常是 ~/.node-gyp 目录下）搜我们当前 Node.js 版本的头文件和静态连接库文件，若不存在，它就会去 Node.js 官网下载。并且node-gyp在编译时，会将这些头文件和静态链接库文件合并到我们事先写好的 binding.gyp 中，这样我们就能够在使用node-gyp编译的源文件中直接引用这些头文件。

对于一个node插件来说，它通常以npm包的形式分发，这个npm包中包含了c/c++的源码，并且会在安装后，调用node-gyp来编译这些源码，这样就能够编译出适合当前node版本的可用node原生模块。

## web桌面应用程序所面临的开发node原生c++模块的场景

- 对接第三方库或者驱动：通常来说当我们的electron应用需要对接一些硬件的时候，硬件厂商通常会提供c/c++的库或者驱动程序给我们，比如常见的相机，这时候，可能就需要使用c++来接入硬件的SDK，并暴露给js层去调用。
- 当我们需要使用c/c++部分来开发我们的业务时，通常出于性能因素的考量。

## 开发node原生c++模块的几种方式

参考：

- node各种开发方式的示例：[node-addon-examples](https://github.com/nodejs/node-addon-examples)

在开始讲解这几种方法之前呢，我们先要明确我们开发node原生c++模块的目的是什么，我们的目的很简单，就是为了使用c/c++来编写某些功能或者封装某些第三方c++库来能够提供给业务的js去使用，以达到满足性能需求或者使用某些c++提供的功能的目的。所以，你所编写的代码的最终目标就是让js中能够调用起c/c++的代码。而你所编码的范围也在这里。

而实现这个目的，很核心的一个内容就是：怎么包装c/c++的函数让js能够调用起来，怎么打通js和c/c++直接的数据转换，并且，处理好异步。而c++那块业务开发你完全可以交由c++开发人员去写，第三方库你只需要知道它有哪些接口，怎么使用就行，其他的不用关心。

很老旧的模式就不说了，主要重点提一下下面几种。

### 使用原生node.js、V8、libuv等库来编写插件

你可以理解为node项目提供了一个编写c++插件的机制，让开发者可以基于node开发c++插件来完成自己的业务功能。既然是编写插件，则node自然需要暴露出一些接口能力来给插件使用，通常插件开发者涉及到的一些组件和API包括：V8、libuv、一些node内部的c++ API以及一些其他静态库（比如OpenSSL、zlib）等。

插件开发者使用这些库和API，能够最大程度的利用node提供的功能来编写自己的node c++插件。

这是一个示例：

```c++
// library.cpp
#include "node.h"

namespace demo {
    using v8::FunctionCallbackInfo;
    using v8::Isolate;
    using v8::Local;
    using v8::Object;
    using v8::String;
    using v8::Number;
    using v8::Value;
    using v8::Context;
    using v8::Maybe;
    using v8::Exception;

    /*
     * 纯粹的js代码
     * */

    /*
     * 定义方法 等价于：function getPid() { return global.process.pid }
     * FunctionCallbackInfo类似于该函数被调用时的信息，比如参数，它的返回值等等
     * */
    void GetPid(const FunctionCallbackInfo<Value>& args) {
        Isolate* isolate = args.GetIsolate();
        // 获取v8实例上下文的Global对象，这里是node的global对象
        auto global = isolate->GetCurrentContext()->Global();

        auto pid = global->Get(String::NewFromUtf8(isolate, "process")).As<Object>()->Get(String::NewFromUtf8(isolate, "pid")).As<Number>();
        // 设置该函数被调用时的返回值
        args.GetReturnValue().Set(pid);

    }

    // 定义方法 等价于：function getVersion() { return this.version }
    void GetVersion(const FunctionCallbackInfo<Value>& args) {
        Isolate* isolate = args.GetIsolate();

        auto self = args.This();
        Local<Value> version = self->Get(String::NewFromUtf8(isolate, "version"));
        args.GetReturnValue().Set(version);
    }

    /*
     * c++业务代码
     * */
    double libAdd(double &x, double &y) {
        return x + y;
    }

    /*
     * 定义方法 等价于：
     * function add(x, y) {
     *     if (arguments.length < 2) { throw new TypeError('args.length must tow') }
     *     return x + y
     * }
     * */
    void Add(const FunctionCallbackInfo<Value>& args) {
        Isolate* isolate = args.GetIsolate();

        if (args.Length() < 2) {
            isolate->ThrowException(v8::Exception::TypeError(String::NewFromUtf8(isolate, "args.length must tow")));
        }

        auto x = args[0].As<Number>()->Value();
        auto y = args[1].As<Number>()->Value();

        double result = libAdd(x, y);
        args.GetReturnValue().Set(Number::New(isolate, result));

    }

    // 初始化函数
    void Initialize(Local<Object> exports) {
        // exports.As<Object>()->Set(String::NewFromUtf8(exports->GetIsolate(), "add"), v8::Function::New(exports->CreationContext(), Add).ToLocalChecked());
        NODE_SET_METHOD(exports, "getVersion", GetVersion);
        NODE_SET_METHOD(exports, "add", Add);
        NODE_SET_METHOD(exports, "getPid", GetPid);
    }

    // node提供的定义模块的宏
    NODE_MODULE(NODE_GYP_MODULE_NAME, Initialize)

}


```

上诉示例只是处理纯粹的js和c++代码的接口，这仅仅只涉及到了v8.h相关的，还并没有涉及到更复杂的比如事件循环、异步IO的libuv库。

使用这种方式开发node原生插件有很大的问题，其一就是你需要对node.h、v8等模块有一定的了解，但是更大的问题是，不同的node版本，其暴露的node.js、V8、libuv接口可能会变，它不保证这些接口的稳定性，而node更新、维护版本的速度非常快，所以，很容易就出现你基于某一个node版本的这些模块所开发的原生插件可以在该版本上运行，但是，有可能下一个版本，或者之前的版本就不行了，编译就无法通过了，这是最大的问题。

### 使用NAN

[nan](https://www.npmjs.com/package/nan)他是一个npm库，全称`Native Abstractions for Node.js`，即nodejs原生模块抽象接口，nan将nodejs不同版本所暴露出来的v8、libuv及其本身的API之间的差异全部都封装在nan的内部，并且它还提供了一些有用的方法来帮助我们简化插件的开发。node原生插件的开发者能够使用nan提供的这些宏和工具方法来抹平掉不同的node版本之间原生模块开发的差异。

他基本的实现很大一部分是在编译期间通过node的版本去判断，如果是xxx版本就返回xxx，它不是动态运行时的代码，这就导致了使用nan编写的插件，在不同的node版本下需要重新编译。

例如一个取默认事件循环实例的API：

```c++
// Nan::GetCurrentEventLoop
inline uv_loop_t* GetCurrentEventLoop() {
// 条件编译，在编译期处理
#if NODE_MAJOR_VERSION >= 10 || \
  NODE_MAJOR_VERSION == 9 && NODE_MINOR_VERSION >= 3 || \
  NODE_MAJOR_VERSION == 8 && NODE_MINOR_VERSION >= 10
    return node::GetCurrentEventLoop(v8::Isolate::GetCurrent());
#else
    return uv_default_loop();
#endif
}
```

### Node-API

Node-API以前叫做N-API，只不过现在名字变了，他是node官方的一个新规范接口，也是编写插件时，官方建议使用的规范接口，除非Node-API所暴露的接口无法满足需求。它封装了底层node的底层接口，将不同版本的node api接口和v8等等的接口都抽象化成为Node-API接口，并且它是符合ABI的接口，只要node所支持的ABI版本一致，那么只需要一次编译就可以在不同的node版本中使用。这是node项目内部维护的一个部分，随着node版本一起更新。

NAN也是封装了node本身提供的那些v8、libuv，而Node-API做的也是同样的事情，但是Node-API和NAN最大的差别在于，NAN更多的是在编译阶段去处理不同node版本API之间的差异，而Node-API本身更多的是作为node的一部分，将当前node版本需要暴露的所有API都转换为Node-API接口规范。

不过为了更加通用，Node-API是一个C接口的API规范，其实这个写起来也挺麻烦。

### node-addon-api

因为Node-API是一个c的api规范，而通常我们使用c++可能会更加方便一点。所以，napi-addon-api库基于Node-API封装出了面向对象的c++接口。这样，编写原生模块更加简洁方便。[node-addon-api](https://github.com/nodejs/node-addon-api)，这个库仅仅是将c风格的N-API接口包装为了c++面向对象风格，其内部仍然是使用的N-API，所以，它能提供ABI稳定性保证。

### 补充：Node-API (N-API) for Rust

参考：

- [github: napi-rs](https://github.com/napi-rs)
- [napi-rs文档](https://napi.rs/)

使用rust来编写node原生模块，而非C/C++，它同样在提供高性能的同时，还可以借由rust提供的安全特性来为原生模块提供更高的质量，兼容CommonJS和esm，而且可以生成d.ts文件。不过我倒是没有具体尝试过，感兴趣的话后面可以研究一下。

## 小结

本次主要讲解了我们在编写node原生c++模块时，可以使用哪些方式，也大致了解了使用这些方法编写原生c++模块是怎么样的一个风格，其实大致上来说，node官方维护了Node-API来保证插件开发者能够编写一次代码就能够在不同的node版本中编译运行，而如果不同node版本的Node-API的ABI版本是兼容的，那么连重新编译都不需要。而基于Node-API开发出来的node-addon-api，则在此基础之上，提供使用更加便捷的c++接口。通常来说，我们编写原生c++模块基本上是选用node-addon-api的，而在这些模块和API深处，我们可以更加深入的去探索node本身提供的那些模块，这样对于我们了解node、了解js也是一种很好的帮助。

- [node官方文档](https://nodejs.org/dist/latest-v18.x/docs/api/addons.html)
- [从暴力到 NAN 再到 NAPI——Node.js 原生模块开发方式变迁](https://cnodejs.org/topic/5957626dacfce9295ba072e0)
- [使用napi编写add方法以及一些细节](https://blog.csdn.net/ghosind/article/details/108132910)

