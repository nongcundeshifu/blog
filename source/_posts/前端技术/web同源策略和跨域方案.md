---
title: web同源策略和跨域方案
date: 2022-05-02 11:12:12
tags: 
  - 同源策略
  - 跨域
categories:
  - 前端技术
---


Web 的同源策略（Same-Origin Policy，SOP） 是浏览器最核心的安全机制之一，其核心目的是限制不同源的页面之间的资源访问与交互，防止恶意网站窃取数据、篡改内容或发起未授权操作，保障用户隐私和 Web 应用安全。

<!-- more -->

## 什么是同源策略

协议、主机（域名）、端口号一直，则被认为是同源。其中主域名和子域名是不同源。判断两个页面是否 “同源”，需同时满足以下 3 个条件，缺一不可：

- 协议（Protocol）相同：如均为 http、https 或 file（本地文件协议），`http://a.com` 与 `https://a.com` 不同源；
- 域名（Domain）相同：如均为 example.com，a.example.com（子域）与 example.com（父域）不同源，example.com 与 example.org（不同注册域）更不同源；
- 端口（Port）相同：如均为 80（HTTP 默认端口）或 443（HTTPS 默认端口），a.com:8080 与 a.com:80 不同源（即使协议和域名相同）。

## 浏览器同源策略影响什么？

同源策略并非 “一刀切” 禁止所有跨源交互，而是精准限制 “高风险操作”，主要涵盖以下 4 类核心场景：

### DOM 访问限制：禁止跨源页面操作 DOM

不同源的页面之间，无法通过 window.parent、window.opener 或 iframe.contentWindow 访问对方的 DOM（如读取 / 修改 document、Cookie、localStorage 等）。

- 例：a.com 的页面嵌入 b.com 的 iframe，a.com 无法通过 iframe.contentDocument 读取 b.com 的 DOM 内容，反之亦然；
- 例外：若 iframe 设置了 sandbox="allow-same-origin" 且与父页面同源，可解除该限制（需谨慎使用）。

### 数据存储访问限制：禁止跨源读取敏感存储

不同源的页面无法读取对方的本地存储数据，包括：

- localStorage / sessionStorage：完全禁止跨源访问，无法通过任何 API 读取或修改；
- Cookie：受 “域名 + 路径” 限制（详见 Cookie 同源规则），跨源页面默认无法读取，但可通过 SameSite=None 配置允许跨源携带（需配合 Secure）；
- IndexedDB / Web SQL：完全禁止跨源访问，每个源有独立的数据库空间。

### AJAX/Fetch 请求限制：禁止跨源发起未授权 HTTP 请求

默认情况下，浏览器禁止通过 XMLHttpRequest 或 Fetch API 向不同源的服务器发起请求（即 “跨域请求”）。即使请求成功到达服务器并返回响应，浏览器也会拦截响应，不允许前端读取数据，仅在满足 “跨域资源共享（CORS）” 规则时才放行。

- 例：a.com 的前端直接通过 Fetch 请求 b.com/api/data，服务器返回数据后，浏览器会因 “无 CORS 许可” 拦截响应，前端无法获取数据；
- 例外：`<img>、<script>、<link>` 等标签的 “被动跨源请求”（如加载图片、脚本）不受此限制，但仅能加载资源，无法`读取响应内容`（这也是 JSONP 跨域方案的原理）。

### 脚本执行与资源加载限制：部分跨源资源可加载但受约束

- 跨源脚本（如 `<script src="https://b.com/script.js">`）可加载并执行，但无法访问当前源的 DOM 或存储数据；
- 跨源样式表（如 `<link href="https://b.com/style.css">`）可加载并应用，但无法通过 getComputedStyle 读取样式值（避免泄露敏感样式信息）；
- 跨源图片、视频等媒体资源可加载，但无法读取其像素数据（如通过 Canvas 绘制跨源图片后，Canvas 会被 “污染”，无法调用 toDataURL() 等 API）。

## corb

[跨域读取阻止](https://www.chromium.org/Home/chromium-security/corb-for-developers/)

> 注意，这不是cors

## 跨域方案

同源策略虽保障安全，但也限制了合法的跨源功能（如前后端分离项目、第三方登录）。为此，浏览器提供了以下合规方案，允许在受控范围内突破同源限制

### 跨域资源共享：CORS，主流的 AJAX 跨域方案

简单点理解就是对于某一个资源的请求，服务端通过类似`Access-Control-*`的响应头来告诉浏览器，哪些资源允许被跨域访问。

核心响应头：

- `Access-Control-Allow-Origin`: `https://a.com`：允许 `a.com` 跨源访问；
- `Access-Control-Allow-Methods`: GET, POST, PUT：允许的请求方法；
- `Access-Control-Allow-Credentials` true：允许携带 Cookie 跨源请求（需前端配合设置 credentials: 'include'）。

注意：跨域资源共享是一种提供了安全跨域的方案，它不是预防跨站请求伪造的手段。

参考：

- [CORS](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/CORS)
- [为什么跨域请求要分为简单请求和非简单请求且需要OPTIONS？](https://www.zhihu.com/question/268998684/answer/344949204)

### 其他跨域手段

- jsonp：jsonp的原理在于，`<script>，<img>，<iframe>，<link>`等标签都不受同源策略限制，都可以跨域请求资源的特性，通过script的src请求一个接口，并将回调函数传递过去，然后后台返回一个调用该回调函数的js代码，并将数据填充进去，以此来达到获取数据的目的。
- 服务器代理：通常来说浏览器会有跨域限制，而服务端则没有跨域问题。所以，我们可以利用一个服务端来帮我们去请求跨域的资源，然后返回给我们前端。
- postMessage：跨窗口 /iframe 安全通信
  - 不同源的窗口（如父窗口与 iframe、window.open 打开的子窗口）可通过 window.postMessage 发送消息，接收方通过监听 message 事件处理消息。但是，需验证消息来源（event.origin）确保安全。

## 常见跨域问题

### AJAX（XMLHttpRequest 或 Fetch）请求跨域

当 AJAX（XMLHttpRequest 或 Fetch）请求遇到跨域问题时，浏览器会通过 控制台抛出明确的错误信息，且不同跨域场景（如未配置 CORS、预检请求失败、携带 Cookie 违规等）的错误提示存在差异。这些错误的核心是 “浏览器拦截了跨域响应”，而非服务器未返回数据。

所有跨域错误均会在浏览器 “开发者工具 → Console” 中显示，且错误信息包含 “CORS”“Cross-origin” 等关键词，便于快速识别。例如`Access to XMLHttpRequest at 'xxx' from origin 'xxx' has been blocked by CORS policy: No 'Access-Control-Allow-Origin' header is present on the requested resource.`。

#### 最常见原因：服务器未配置 CORS 头（无任何 CORS 响应）

当跨域请求发送后，服务器未返回 Access-Control-Allow-Origin 等 CORS 头时，浏览器会直接拦截响应，抛出 “缺少 CORS 许可” 的错误。浏览器判定该跨域请求 “未被授权”，拒绝将响应返回给前端。

#### 预检请求（OPTIONS）失败

对于 复杂跨域请求（如使用 PUT/DELETE 方法、带自定义请求头、携带 Cookie 等），浏览器会先发送 OPTIONS 预检请求，验证服务器是否允许该请求。若预检请求被拒绝或响应不符合要求，则会抛出错误。通常可能有两种情况：

- 预检请求被服务器拦截（返回 403/404）
- 请求方法未被服务器允许：服务器 Access-Control-Allow-Methods 未包含当前请求的方法（如只允许 GET/POST，却发起 PUT 请求）。

#### 携带 Cookie 时的 CORS 违规

当前端请求设置 credentials: 'include'（携带 Cookie），但服务器 CORS 配置不符合要求时，也会抛出错误。

- 服务端设置 `Access-Control-Allow-Origin 为 *` 且前端设置：credentials: 'include'（携带 Cookie）
  - 浏览器禁止 “携带 Cookie 的跨域请求” 与 `Access-Control-Allow-Origin: *`（允许所有源）同时存在（安全限制，避免 Cookie 被任意网站窃取），需将 Access-Control-Allow-Origin 设为具体域名（如 `https://your-domain.com`）。
- 服务器未开启 Access-Control-Allow-Credentials

#### AJAX跨域时的注意事项

- 拦截主体：错误均由 “浏览器” 抛出（而非服务器），提示中会明确 “blocked by CORS policy”；
- 响应状态：跨域错误时，浏览器不会将服务器返回的 HTTP 状态码（如 200/400）暴露给前端，前端获取的状态码通常是 0（Fetch 中为 TypeError，XMLHttpRequest 中 status 为 0）；
- 无网络错误：跨域错误不代表 “请求未发送到服务器”—— 多数情况下，请求已到达服务器并返回响应，但被浏览器拦截（可通过 “Network” 面板查看请求是否有响应）。
- 通常我们可以通过Network面板来协助我们排查问题
  - 找到目标跨域请求，查看 “Response Headers”是否符合我们预期
  - 对于复杂请求，查看 “OPTIONS 预检请求” 的响应

AJAX 跨域错误的核心是 “浏览器基于同源策略拦截了未授权的跨域响应”，不同场景的错误提示虽有差异，但均围绕 “CORS 配置缺失或违规” 展开。排查时需结合 Console 错误关键词 和 Network 面板的响应头，定位服务器 CORS 配置的问题（如缺少头、域名不匹配、未处理预检请求等），再针对性修复。

### 跨域请求被 CORB 拦截（特殊场景）

若跨域请求的响应是 JSON/HTML/XML 等敏感数据，且未通过 CORS 验证，浏览器会通过 CORB（跨源读取阻止） 主动拦截响应，抛出类似以下错误：`Cross-Origin Read Blocking (CORB) blocked cross-origin response xxxxxx`

CORB 是浏览器的额外安全机制，针对 “未通过 CORS 的敏感数据响应”，直接在内存层面阻止前端读取，比普通 CORS 错误更严格（即使响应到达浏览器，也不会暴露给前端）。参考：[跨域读取阻止](https://www.chromium.org/Home/chromium-security/corb-for-developers/)

### canvas跨域污染

#### 问题原因

默认情况下，浏览器对图片资源的处理遵循 “宽松加载、严格使用” 原则：

- 加载阶段：`<img>` 标签或 new Image() 可以跨域加载图片（如 a.com 加载 b.com 的图片），这是为了满足网页嵌入第三方图片的需求（如广告、头像）；
- 使用阶段：若将跨域加载的图片绘制到 Canvas 中，Canvas 会被标记为 “跨域污染（Cross-Origin Tainted）”—— 此时调用 toDataURL()（获取图片 Base64）、getImageData()（读取像素数据）等 API 会被浏览器拦截，报错类似：`Uncaught DOMException: Failed to execute 'toDataURL' on 'HTMLCanvasElement': Tainted canvases may not be exported.`

这是为了防止攻击者通过 Canvas 读取跨域图片的像素数据（如用户隐私图片、验证码），规避同源策略的安全保护。

#### 处理方案

图片资源的跨源限制可以通过 CORS 机制解除，要让 Canvas 绘制跨域图片后正常调用 toDataURL()，需同时满足以下两个条件：

1. 图片服务器配置 CORS 响应头（核心）：图片所在的服务器（如 b.com）必须在响应中添加 CORS 许可头，明确允许请求来源（如 a.com）访问图片的 “像素数据权限”。
2. 前端发起图片请求时声明 “跨域意图”：前端在加载跨域图片时，需通过 crossOrigin 属性明确告知浏览器：“此请求需要跨域权限，需验证服务器的 CORS 许可”。crossOrigin 有两个可选值：
   1. anonymous（推荐）：跨域请求不携带 Cookie，适用于无需身份验证的场景；
   2. use-credentials：跨域请求携带 Cookie，需服务器配合设置 Access-Control-Allow-Credentials: true。

注意：使用 new Image() 处理跨域资源时，必须 先设置 crossOrigin，再设置 src—— 若先设置 src，浏览器会默认以 “非跨域模式” 加载图片，后续添加 crossOrigin 无效。

**设置了crossOrigin="anonymous" 不生效？可能原因：**

- 图片 URL 存在缓存：若之前未加 crossOrigin 加载过该图片，浏览器会缓存 “非跨域模式” 的图片，后续添加 crossOrigin 也无效。解决方案：在图片 URL 后加随机参数（如 `https://b.com/image.png?t=123456`），强制刷新缓存；
- 服务器未正确返回 CORS 头：通过浏览器 “开发者工具 → Network → 图片请求 → Response Headers” 检查是否有 Access-Control-Allow-Origin，若没有，需重新配置服务器；
- crossOrigin 拼写错误：注意是 crossOrigin（驼峰式），而非 crossorigin（全小写，部分浏览器不兼容）。

**为什么 `<img>` 能跨域加载图片，但 Canvas 不能使用？**

主要是因为浏览器的 “分层安全策略”：

- 加载图片仅需 “资源获取权限”，浏览器允许跨域加载以满足网页功能需求；
- Canvas 读取像素数据需 “数据访问权限”，属于更高风险操作，必须通过 CORS 验证，确保服务器明确许可跨源访问 —— 避免攻击者滥用图片加载的宽松策略窃取敏感数据。
