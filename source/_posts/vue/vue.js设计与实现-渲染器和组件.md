---
title: vue.js设计与实现-渲染器和组件
date: 2026-03-06 14:36:22
tags: 
  - vue
categories:
  - vue
---

仍然是最近几个月闲来无事，把js和css又重新过了一遍之后，又把目光又放到了vue3上面，之前几年都是用的react比较多，vue2只在大学毕业那一两年用过，vue3还是最近的工作中才有机会去使用的，同样，为了较为体系的去了解vue3，除了官方文档之外，还找了本《Vue.js设计与实现》来作为参考，这本书不厚，干货确实不少，对于vue了解不深的人来说，会有不小的收获。同时在学习的过程中，也会动手去自行实现，最终跑出了一个感觉还不错的vue框架示例。

<!-- more -->

## 前言

上篇文章我们了解了vue3的响应式系统，这次我们接着来看看vue3的渲染器和组件。渲染器和组件在我的第一篇vue文章中有提到，不了解的可以先去看看：[vue.js设计与实现-开始：初识渲染器](/2026/03/04/vue/vue.js设计与实现-开始/)

相关代码参考：[github：vue-design](https://github.com/nongcundeshifu/vue-design)

## 渲染器的设计

- 在 Vue.js 中，很多功能依赖渲染器来实现，例如 Transition 组件、Teleport 组件、Suspense 组件，以及 template、ref 和自定义指令等
- 渲染器也是框架性能的核心，渲染器的实现直接影响框架的性能。
  - Vue.js 3 的渲染器不仅仅包含传统的 Diff 算法
  - 它还独创了快捷路径的更新方式，能够充分利用编译器提供的信息，大大提升了更新性能。

### 渲染器与响应式系统的结合

- 渲染器是用来执行渲染任务的。在浏览器平台上，用它来渲染其中的真实 DOM 元素。渲染器不仅能够渲染真实 DOM 元素，它还是框架跨平台能力的关键。
- 渲染器：将一个源内容（其形式可以是字符串，也可以是一个对象（虚拟dom））渲染到目标内容上的东西。例如将字符串，渲染到浏览器的真实dom，只要能够实现这个目标，那就是一个渲染器。

既然渲染器用来渲染真实 DOM 元素那么严格来说，下面就是一个渲染器

```js
function renderer(domString, container) {
  container.innerHTML = domString
}
renderer('<h1>Hello</h1>', document.getElementById('#app'))
const count = 1
// 也可以是一个带变量的字符串
renderer(`<h1>${count}</h1>`, document.getElementById('#app'))
```

如果上面代码的count是一个响应式对象呢？这让我们联想到副作用函数和响应式数据。利用响应系统，我们可以让整个渲染过程自动化

```js
const count = ref(0)
watchEffect(() => {
  renderer(`<h1>${count}</h1>`, document.getElementById('#app'))
})
count.value++
```

在这段代码中，我们首先定义了一个响应式数据 count，它是一个 ref，然后在副作用函数内调用 renderer 函数执行渲染。副作用函数执行完毕后，会与响应式数据建立响应联系。当我们修改 count.value 的值时，副作用函数会重新执行，完成重新渲染。

这就是响应系统和渲染器之间的关系：我们利用响应系统的能力，自动调用渲染器完成页面的渲染和更新。`这个过程与渲染器的具体实现无关`

### 渲染器的基本概念

实现参考：renderer-1.ts文件

- 我们通常使用英文 renderer 来表达“渲染器”。千万不要把 renderer 和 render 弄混了，前者代表渲染器，而后者是动词，表示“渲染”。
  - 渲染器是更加宽泛的概念，它包含渲染。渲染器不仅可以用来渲染，还可以用来激活已有的 DOM 元素，这个过程通常发生在同构渲染的情况下(服务器渲染和客户端渲染或uniapp等)
  - 即一个渲染器，可能包含普通的浏览器渲染，也可能包含服务端的渲染，甚至还可以是其他的渲染
- 渲染器的作用是把虚拟 DOM 渲染为特定平台上的真实元素。在浏览器平台上，渲染器会把虚拟 DOM 渲染为真实 DOM 元素。
- 虚拟 DOM 通常用英文 virtual DOM 来表达，有时会简写成 vdom。
- 虚拟 DOM 和真实 DOM 的结构一样，都是由一个个节点组成的树型结构。所以，我们经常能听到“虚拟节点”这样的词，即 virtual node，有时会简写成 vnode。虚拟 DOM 是树型结构，这棵树中的任何一个 vnode 节点都可以是一棵子树，因此 vnode 和 vdom 有时可以替换使用。
- 渲染器把虚拟 DOM 节点渲染为真实 DOM 节点的过程叫作挂载，通常用英文 mount 来表达。
- 渲染器通常需要接收一个挂载点作为参数，用来指定具体的挂载位置。这里的“挂载点”其实就是一个 DOM 元素，渲染器会把该 DOM 元素作为容器元素，并把内容渲染到其中。我们通常用英文 container 来表达容器。

### 渲染器的功能

- 我们首先调用 createRenderer 函数创建一个渲染器，接着调用渲染器的 renderer.render 函数执行渲染（这里表示浏览器的dom渲染，先不考虑其他的例如服务端渲染等）。当首次调用 renderer.render 函数时，只需要创建新的 DOM 元素即可，这个过程只涉及挂载。
- 而当多次在同一个 container 上调用 renderer.render 函数进行渲染时，渲染器除了要执行挂载动作外，还要执行更新动作
  - 由于首次渲染时已经把 oldVNode 渲染到 container 内了，所以当再次调用 renderer.render 函数并尝试渲染 newVNode 时，就不能简单地执行挂载动作了。
  - 在这种情况下，渲染器会使用 newVNode 与上一次渲染的 oldVNode 进行比较，试图找到并更新变更点。这个过程叫作“打补丁”（或更新），英文通常用 patch 来表达（即一个patch函数）。但实际上，挂载动作本身也可以看作一种特殊的打补丁，它的特殊之处在于旧的 vnode 是不存在的。
  - patch函数，用来将两个新旧vnode对比（最开始旧的vnode是空），找出差异，并更新挂载
- 在对比两个新旧节点后，发现旧节点已经不需要了，则还需要将其进行卸载掉

### 挂载和自定义渲染器

- 正如一直强调的那样，渲染器能够把虚拟 DOM 渲染为浏览器平台上的真实 DOM。这一步叫做挂载
- 同时我们通过将渲染器设计为可配置的“通用”渲染器，即可实现渲染到任意目标平台上。
- 我们将以浏览器作为渲染的目标平台，编写一个渲染器，在这个过程中，看看哪些内容是可以抽象的，然后通过抽象，将浏览器特定的 API 抽离，这样就可以使得渲染器的核心不依赖于浏览器。
- 自定义渲染器并不是“黑魔法”，它只是通过抽象的手段，让核心代码不再依赖平台特有的 API，再通过支持个性化配置的能力来实现跨平台。
- 通过将浏览器dom的操作：创建、插入、删除等逻辑抽离为一个独立的API（这里将其命名为ElementAPI），并在创建渲染器时作为参数传递进去，从而实现逻辑分离。
- 这里的挂载不只是将虚拟dom挂载真实的页面上去，也同样包含递归子节点时，将虚拟dom挂载到已经创建的节点中（节点插入）。

### 卸载

卸载需要考量的点：

- 容器的内容可能是由某个或多个组件渲染的，当卸载操作发生时，应该正确地调用这些组件的 beforeUnmount、unmounted 等生命周期函数。
- 即使内容不是由组件渲染的，有的元素存在自定义指令，我们应该在卸载操作发生时正确执行对应的指令钩子函数。
- 使用 innerHTML 清空容器元素内容的另一个缺陷是，它不会移除绑定在 DOM 元素上的事件处理函数。（存疑）
- 正确的卸载方式是，根据 vnode 对象获取与其相关联的真实 DOM 元素，然后使用原生 DOM 操作方法将该 DOM 元素移除。为此，我们需要在 vnode 与真实 DOM 元素之间建立联系
- 并且在此卸载过程中，我们可以根据节点类型，来调用相应的组件生命周期或者dom中的指令的生命周期

### 更新

- patch函数会对比两个vNode之间的差异，进行打补丁，但是如果在它们两个vNode本身不相同，则不需要进行对比和打补丁了，所以在具体执行打补丁操作之前，我们需要保证新旧 vnode 所描述的内容相同。
  - 如果两个vNode本身类型都不一样，其大部分情况下，基本上是整个vNode树都被替换掉了，此时进行打补丁的操作，不如重新直接卸载旧的，创建新的。
- 即使它们描述的内容相同，我们也需要进一步检查它们的类型，即检查 vnode.type 属性值的类型，据此判断它描述的具体内容是什么。如果类型是字符串，则它描述的是普通标签元素，这时我们会调用 mountElement 和 patchElement 来完成挂载和打补丁；如果类型是对象，则它描述的是组件，这时需要调用 mountComponent 和 patchComponent 来完成挂载和打补丁。
- patchElement中会先处理属性更新，然后再处理子节点的更新，且判断新旧的子节点是否是数组，只有都是数组时才会调用diff算法进行更新
  - 子节点可能存在的情况
    - 没有子节点
    - 文本子节点
    - 子节点数组：文本、组件、普通子节点
  - 只有子节点都是数组类型的时，才需要进行diff算法
- 下文会介绍三种diff算法，分别是简单diff、双端diff和快速diff（vue3目前使用的是快速diff算法）

### HTML属性处理

- HTML 标签有很多属性，其中有些属性是通用的，例如 id、class 等，而有些属性是特定元素才有的，例如 form 元素的 action 属性。实际上，渲染一个元素的属性比想象中要复杂
  - 并不是所有 HTML Attributes 都有与之对应的 DOM Properties
  - 类似地，也不是所有 DOM Properties 都有与之对应的 HTML Attributes
  - 例如value、checked 属性的同步问题
  - HTML Attributes 与 DOM Properties 之间的关系很复杂，但其实我们只需要记住一个核心原则即可：`HTML Attributes 的作用是设置与之对应的 DOM Properties 的初始值。`
    - HTML Attributes：仅少数特殊属性（如 id/type）修改后会同步到 property；大部分属性（如 value/checked）不同步
    - DOM Properties：修改 property 几乎不会同步到 attribute（除非手动调用 setAttribute）
- 这意味着：浏览器解析 HTML 代码后，会自动分析 HTML Attributes 并设置合适的 DOM Properties。但用户编写在 Vue.js 的单文件组件中的模板不会被浏览器解析，这意味着，原本需要浏览器来完成的工作，现在需要框架来完成（只读属性、值不同步属性、布尔属性等）
  - 例如disable属性，使用dom属性设置或者使用Attributes设置，都存在不同场景的问题，对于这种，我们就需要特殊处理。
  - 例如value属性，其作为html的属性时，其同步的其实是defaultValue的dom属性，而不是dom的value属性。而用户在输入框中修改时，其修改的值是dom.value属性，其Attributes是不会更新的。（非vue组件中）
  - 而在vue组件中，且value属性会绑定到html的value属性上，而不是dom.value属性。而浏览器的input输入框更新时，这是浏览器行为，他会更新dom的value值，此时我们需要在这个change事件中去更新我们的ref值（这同时也会触发渲染，来更新浏览html中的value Attributes）
- class属性处理：为什么需要对 class 属性进行特殊处理呢？这是因为 Vue.js 对 class 属性做了增强。
  - 因为 class 的值可以是多种类型，所以我们必须在设置元素的 class 之前将值归一化为统一的字符串形式，再把该字符串作为元素的 class 值去设置。
  - 我们需要封装 normalizeClass 函数，用它来将不同类型的 class 值正常化为字符串
  - 同时使用 setAttribute、el.className 或 el.classList这三种方式设置class，其el.className的性能最好，故还是需要特殊处理class的属性设置。
- style属性：除了 class 属性之外，Vue.js 对 style 属性也做了增强，所以我们也需要对 style 做类似的处理。
- 通过对 class 的处理，我们能够意识到，vnode.props 对象中定义的属性值的类型并不总是与 DOM 元素属性的数据结构保持一致，这取决于上层 API 的设计。Vue.js 允许对象类型的值作为 class 是为了方便开发者，在底层的实现上，必然需要对值进行正常化后再使用。另外，正常化值的过程是有代价的，如果需要进行大量的正常化操作，则会消耗更多性能。

### 事件处理

- 事件可以视作一种特殊的属性，因此我们可以约定，在 vnode.props 对象中，凡是以字符串 on 开头的属性都视作事件（注意，是虚拟dom中，不是组件的props，组件的props没有这个约定）
  - 将事件作为一个特殊的vnode.props进行处理，并将事件处理函数作为dom对象的一个自定义属性（存储为key、value来处理多个事件）进行存储，这样只需要使用addEventListener添加一次事件，后续更新只需要更新事件处理函数的值即可，销毁时也只需要处理一次。
- 事件冒泡与更新时机问题
  - 由于事件冒泡的事件处理函数执行机制和微任务的协同，导致冒泡的每个事件的的处理执行都会清空微任务，从而导致整个事件传播过程中，会触发那些组件更新后被绑定的事件（通常vue的更新是在微任务中的）这可能并不符合预期。
  - 解决方案则是通过判断事件触发的时间和事件的绑定事件，跳过那些事件绑定时间在事件触发时间之后的哪些处理函数。（事件的event对象有一个timeStamp标识事件触发的时间戳）

### 简单diff算法

实现参考：renderer-2.ts文件

简单 Diff 算法的核心逻辑是，拿新的一组子节点中的节点去旧的一组子节点中寻找可复用的节点。如果找到了，则记录该节点的位置索引。我们把这个位置索引称为最大索引。在整个更新过程中，如果一个节点的索引值小于最大索引，则说明该节点对应的真实 DOM 元素需要移动。找不到则为新增节点，并且在最后遍历旧节点，移除那些不存在于新节点的哪些（使用key判断）

当新旧 vnode 的子节点都是一组节点时，为了以最小的性能开销完成更新操作，需要比较两组子节点，用于比较的算法就叫作 Diff 算法。我们知道，操作 DOM 的性能开销通常比较大，而渲染器的核心 Diff 算法就是为了解决这个问题而诞生的

最简单的长度不一致时的更新（不考虑顺序，即不移动）

- 新旧的长度一致，使用patch更新两个节点
- 新节点长度大于旧节点长度，需要添加新节点
- 新节点长度小于旧节点长度，需要删除多余的旧节点

#### 思考：非for循环的子节点列表更新是否有必要使用diff算法？

如果是非循环下的子节点之间的更新，例如一个组件的根div中，有一个h1和p节点，那么它们算是一个子节点列表，此时进行更新时，还有必要进行复杂的diff算法的比较更新吗？

我认为是没有必要的，直接按照最简单的长度一致并一一对应的使用patch进行节点更新即可（vue貌似就是这么做的），为什么？

通常来说，如果不是循环的子节点：

- 首先它们在组件中长度是固定的。
- 即使使用v-if这种移除元素的情况，那么vue中`可以会注释节点来代替移除dom元素的情况，这样可以保证新旧节点的子元素长度是不变的。`，但是，在vue中貌似并不会以此来不进行diff算法（也就是基于长度一致，直接对比）if的注释节点更多是为了确定元素位置的锚点以及在列表diff算法时，防止元素错位，但其实也没有错位）
- 且在大部分情况下，其子元素的顺序是固定的，不会变化
  - 除非在下面这种情况：div中有A、B两个节点，然后再更新后，变成了B、A的顺序。那么你需要使用v-if来实现：`<A v-if /><B /><A v-if />`，此时同一时间只有一个A节点，那么会存在顺序的变化。而这种情况，使用上面的方案，就无法复用A节点了。但是这种情况，第一个A和第二个A真的可以复用？且真的复用的情况有多少，能影响多少性能？
- 且即使出现这种情况，那么你需要手动添加key，且保证它们不会同时出现，那么在编译阶段，我可以基于key来做特殊优化，让这个子节点列表进行深度diff算法。
  - 虽然vue中没有考虑这种情况，即使你使用key，也不会复用，而是直接移除再重新创建。

注意：vue3本身基于编译器来实现基于块的更新，即它有一个动态子节点，使用patchBlockChildren来快速对比和更新（直接遍历列表，按照顺序）所以，这里仅仅是在实现过程中的一些想法。

例如如下节点：

```html
<div class="test-2">
    <div @click="name = 'zhou-2'">name: {{name}}</div>
    <div v-if="name === 'zhou-1'" class="div-1">{{name + '-1'}}</div>
    <div class="div-2">{{name + '-2'}}</div>
</div>
```

那么vue在生成的VNode中，会存在一个dynamicChildren，那么在进行diff更新时，则优先基于这个字段调用patchBlockChildren来更新，这个更新就很简单了，直接遍历新节点列表，按照顺序一一对应进行diff（此时上面那个if如果切换了，那么就是注释节点和div虚拟dom节点之间的比较。更多的优化可以看后续编译器相关的介绍。

#### 使用key来复用子节点

从上一节的思考中我们知道，在非循环中，其实基本上不会出现dom节点的移动，即使使用v-if，导致子节点长度不一致，vue也会使用注释节点来代替节点的移除，从而保证节点长度。那么：

- 真正可能出现节点移动的就只有在for循环中创建的子节点。
- 而在这种情况下，为了标识新旧节点是否是同一个节点从而判断是否复用旧节点来进行移动，我们就为for循环中的子节点引入了key来作为标识符，它的含义为：如果是同一个数据渲染出来节点，那么它的key是一样的。
- 如果在新旧节点中存在key一致的元素，那么表明它们是相同的节点，则新的虚拟dom会复用旧的虚拟dom中的已经创建的dom来进行更新，而非重新创建，只是移动并（如果需要的话）使用patch进行更新。注意：如果在更新中发现复用的（key相同的）两个元素类型不一致，则patch中会自行重新卸载并创建新的。
- 因为`diff算法的最终目标，是为了在一个子列表中找到合适的、更新最小、更新性能最好的两个节点进行patch补丁更新`
- 所以需要注意的是，`使用key找到DOM 可复用并不意味着不需要更新，而是由于key找到的新旧两个进行更新的dom被认为是“同一个”dom，所以对比它们的更新通常来说更新内容会更少，性能会更好。`
- 而且，如果你在一个父节点的子节点中同时使用for和其他子节点来渲染，那么vue在编译阶段会将v-for指令的列表用一个Fragment包裹起来，以便diff算法确保会在包含key的列表中去对比。

简单diff算法的实现方式：

- 我们通过遍历新节点，找到旧节点中，key相同的旧节点，如果找到了
  - 那么复用旧节点的dom，并且用找到的旧节点和新节点进行patch补丁更新
- 如果没有找到，则创建新的节点并插入到合适的位置
- 最后再遍历旧节点，找到不存在于新节点列表的元素，删除掉

优化：

- 建立key的索引map，避免后续的for循环嵌套

#### 处理节点更新、新增和删除

即使使用key复用了节点，但是节点顺序可能变化（新节点列表的顺序变化了），那么我们需要处理节点的移动

- 找到新节点复用时，找到的旧节点列表索引的，并记录找到的旧节点列表的索引的最大值。
- 找到复用的旧节点时，判断如果找到的旧节点索引比记录的旧节点列表索引最大值要小，则它是需要移动的，此时利用dom API将其移动到上一个新节点所在的dom的后面，保证其顺序，完成移动。
- 新增时同样如此。
- 而删除旧节点列表不需要的节点，则在最后再次遍历旧节点列表，找到旧节点的key在新节点列表中不存在的哪些，将其移除。

### 双端diff算法实现（vue2）

实现参考：renderer-3.ts文件

简单 Diff 算法仍然存在很多缺陷，这些缺陷可以通过本章将要介绍的双端 Diff 算法解决（这是vue2中使用的diff算法）

`简单 Diff 算法的问题在于，它对 DOM 的移动操作并不是最优的`。例如，最后一个元素变成了第一个元素时，简单diff算法的移动，需要移动列表长度-1的次数，但是实际上只需要将最后一个元素移动到第一个元素即可。

- 顾名思义，双端 Diff 算法是一种同时对新旧两组子节点的两个端点进行比较的算法。因此，我们需要四个索引值，分别指向新旧两组子节点的端点
- 思考一下，diff算法本身的目的是什么？找到合适的新旧节点进行patch补丁更新，那通过什么找到合适的节点呢？使用key。然而，这只是diff算法的第一步。这一步你可以理解为有新旧两个列表，以新列表为基准，找到旧列表中也存在的元素，同时新增不存在的元素，最后找出旧列表中不在新列表出现的元素。
- 然而除了列表元素的更新之外，还有一个就是减小元素的移动操作，因为列表带有顺序，且需要保持顺序。
- 双端diff就是为了减少移动操作而优化的算法

实现思路：

- 基于4个端点：两个列表节点的开始和结束
- 循环比较新旧节点的头尾节点，直到有一方遍历完为止
- 以新节点列表的两端组成一个范围，并以开头和结尾两个节点，去判断旧节点列表的头尾节点是否命中，从而识别出开头变结尾，结尾变开头的情况（在所在的范围）
- 从而只移动开头和结尾的节点，然后两边范围缩小后再次对比，从而解决可能由于一个元素从开头变成结尾或者从结尾变成开头所导致的元素错位问题，避免无效移动
  - 它等于能够把错位的元素给恢复过来进行对比。
- 而如果新节点列表范围的开始和结束节点都没有对比成功，则按照简单diff算法的方式从遍历去找旧节点列表是否存在一样的key（以新节点列表范围的开头为基准），此时缩小的只有新节点列表的范围，旧节点列表的范围不会变化
  - 且不管有没有找到，对于旧节点列表所在的范围内的节点来说，其真实的dom的相对顺序都不会改变
  - 因为如果找到了，也会将对应的旧节点列表的节点所在的真实dom给移动到这个范围之外，并把这个找到的旧节点给设置为undefined，不会出现在后续的对比中
  - 可以以新节点列表范围的开头为基准去查找，也可以以新节点列表范围的结尾为基准去查找，两次都可以在同一个循环中进行，因为新节点列表范围的结尾已经对比过，如果保留，在下一个4次对比中会重复判断且同样不会成功，所以同时查找新节点列表范围的开头和结尾节点，同时缩小范围会比较好。
- 而且旧节点列表的所在的端范围所对应的真实dom的顺序性在其范围的迭代中是不会更改的，所以，在实现移动时，可以以旧节点列表的端点范围为参考进行移动

问题：

- 需要进行多次的比较（即判断比较的逻辑比较多）

### 快速diff算法

实现参考：renderer-4.ts文件

在 DOM 操作的各个方面，快速 Diff 算法的性能都要稍优于 Vue.js 2 所采用的双端 Diff 算法。利用了预处理和最长递增子序列进行优化

不同于简单 Diff 算法和双端 Diff 算法，快速 Diff 算法包含预处理步骤，这其实是借鉴了纯文本 Diff 算法的思路。在纯文本 Diff 算法中，存在对两段文本进行预处理的过程，即找到文本相同的部分，这一部分是不要进行diff算法的。

其实无论是简单 Diff 算法，还是双端 Diff 算法，抑或本章介绍的快速 Diff 算法，它们都遵循同样的处理规则：

- 判断是否有节点需要移动，以及应该如何移动；
- 找出那些需要被添加或移除的节点（上一步处理完成后处理遗漏的）

实现：

- 第一步：预处理步骤
  - 借鉴了纯文本 Diff 算法的思路，先从开头和结尾分别对比出不需要改变的（即顺序一致的前缀和后缀节点），直接进行更新，此时不需要进行移动
- 第二步：如果清除完成后，新旧列表节点同时还存在未处理的节点，那么以这两个列表中未处理的节点作为两个需要处理的数组列表。
  - 那么以新节点列表的的未处理节点区间为主体，构建source数组，表示其这个列表其对应的旧节点，如果存在，则为旧列表节点的索引位置，说明可以复用，否则值默认为-1，表示需要新创建
  - 遍历旧节点列表来找到新节点列表中key相同的节点（使用map映射避免嵌套循环查找），同时更新source数组，且在其过程中进行patch和卸载
  - 同时会判断列表本身是否需要移动，如果需要移动则继续执行
- 第三步：以source构建最长递增子序列lisSource，来标识其不需要移动的新节点
  - 同时使用两个索引，倒序遍历source和lisSource，判断其是新增、需要移动还是不需要处理。

其首先基于预处理来排除了相同的前缀和后缀。同时在第二步中也会判断剩余列表是否需要移动，如果不需要移动，则也不需要构建最长递增子序列。如果实在需要，则通过最长递增子序列来优化移动的步骤，提高性能。

最长递增子序列的性能：基于贪心和二分法，将其复杂度变为O(N * LogN)，且最长递增子序列越长，其性能越低（因为二分法需要的步骤越多），但是其dom移动的次数也会降低。其是一个综合性能更优，且复杂度更稳定的方式。

列表越长、乱序程度越高，LIS 优化的效果越明显，双端diff在一些列表乱序的情况下，会产生非常多的对比，且对比失败则转变为查找进行移动，从而其操作dom的次数并非最优

快速diff对比双端diff优势例子：

```text
旧列表：[A, B, C, D] → 索引 [0, 1, 2, 3]
新列表：[B, D, A, C] → 旧索引对应 [1, 3, 0, 2]
```

- 双端 Diff：会逐个判断节点位置，发现 B 在新列表的第 0 位（旧索引 1）、D 在第 1 位（旧索引 3）、A 在第 2 位（旧索引 0）、C 在第 3 位（旧索引 2），最终会触发 3 次 DOM 移动；
  - 这里的点在于：双端diff中如果出现了顶部和底部的匹配，此时仍然会触发移动。无法顾忌不同节点之间的相对顺序进行优化
- 快速 Diff：计算可复用节点旧索引序列 [1, 3, 0, 2] 的 最长递增子序列（LIS） 为 [1, 3]（对应节点 B、D）。
  - LIS 序列的节点：相对顺序未变，无需移动；
  - 非 LIS 序列的节点（A、C）：只需插入到 LIS 序列的对应位置，仅需 2 次 DOM 移动。

### 组件的diff说明

- 组件的diff算法其本质上和普通dom的对比没有任何差别，仍然是基于key来进行复用，并进行组件的更新
- 然后这里有一个重要的差别就是，组件本身的children是没有的，也就是说，组件的更新是不会和dom那样去更新children子元素，因为组件的children被用作了slot插槽了，所以组件不需要更新children，仅需要更新props，然后再更新组件的props的过程中，去判断props是否有改变，如果有改变，则直接触发组件的effect副作用函数，重新调用渲染函数，然后再patch更新渲染函数返回的虚拟dom。
- 唯一需要注意的就是，组件进行diff时，如果要更新组件的位置，其anchor的获取（因为组件的虚拟dom中是没有对应的el值的，而是要获取其subTree的el（其subTree中的根节点可能是个Fragment也可能是个组件，所以需要特殊处理）
  - Fragment的节点的el是一个空的文本节点，且会被插入到dom中，而在diff过程中，如果遇到了Fragment节点，那么获取到的el就是这个空的文本节点，其类似于：text div1 div2 text这种顺序。其中两个text包裹住的dom节点就是Fragment组件的children。这样就可以正确获取Fragment VNode的节点作为锚点。
    - 注意：只有双端diff和简单diff中使用了nextSibling这个属性，因为Fragment节点的el.nextSibling是div1（上文的例子），所以以此来插入是不正确的，需要特殊处理。所幸快速diff算法没有使用该属性。
    - 同时在Fragment中的前后插入两个文本节点（同时标记Fragment的头尾范围），并且在VNode中以el和anchor属性存储，这样可以方便删除Fragment，即使后续有需要，也可以通过这个两个位置节点获取真实的dom锚点（例如如果要插入Fragment所在的位置，那么就可以通过anchor的nextSibling来获取一整个Fragment块的下一个兄弟元素作为anchor锚点，如果插入到Fragment区块之前，那么就直接获取其el即可）
- 组件的VNode的el节点其实就是组件的subTree的el，同时组件的更新需要是同步的，而不是异步的，且它是递归更新的，即父组件更新时，更新到其子节点中的子组件时，需要同步更新子组件后，父组件才会继续更新。因为很重要的一点就是diff算法过程中，如果diff的是一个组件，那么子组件没有更新完成的话，那么diff中节点移动时，就无法获取到正确的锚点，从而导致插入位置出现异常。

## 组件

渲染器主要负责将虚拟 DOM 渲染为真实 DOM，我们只需要使用虚拟 DOM 来描述最终呈现的内容即可。但当我们编写比较复杂的页面时，用来描述页面结构的虚拟 DOM 的代码量会变得越来越多，或者说页面模板会变得越来越大。

这时，我们就需要组件化的能力。有了组件，我们就可以将一个大的页面拆分为多个部分，每一个部分都可以作为单独的组件，这些组件共同组成完整的页面。组件化的实现同样需要渲染器的支持

### 组件和渲染器

渲染器，可以渲染真实dom，同样他也可以渲染组件，我们使用type来标识dom或者一些特殊节点，例如Text、Fragment等，渲染器针对组件，也需要进行特殊处理。同时，组件本身是对页面内容的封装，它用来描述页面内容的一部分。因此，一个组件必须包含一个渲染函数，即 render 函数，并且渲染函数的返回值应该是描述页面内容的虚拟 DOM。通过这个渲染函数，此时渲染器和组件的内容就可以联系起来：

- 组件通过渲染函数render返回它所描述的页面的虚拟dom节点
- 而渲染器通过这个渲染器获取组件的虚拟dom节点并且进行渲染组件内容到真实页面上。

### 组件和响应式

组件本质上渲染的内容其实和渲染器一样，即组件有一个渲染函数render，它会访问响应式数据创建出虚拟dom，再交由渲染器渲染。并且对于组件来说，它的渲染函数会被执行在一个类似于effect的函数中（你也可以理解为是一个watchEffect）：

```js
effect(() => {
    // 组件的渲染函数，你可以理解为它包裹在一个effect中，那么此时渲染函数中的引用的响应式数据更新时，自然就会重新执行渲染函数来生成最新的虚拟dom了。
    Component.renderer(context)
})
```

但是非常需要注意的是，组件在创建的过程中，例如setup函数的执行，他会创建一些自己的响应式数据并访问一些响应式数据，此时，`在setup函数中访问一些响应式数据时，需要特别小心依赖是否被意外收集，尤其是组件嵌套渲染时`。例如A组件中会渲染B内容，那么此时A执行渲染函数去创建B组件，那么此时A的setup函数执行是处于一个effect副作用函数中的，那在setup函数中访问的响应式数据不应该作为A组件的副作用函数的响应式依赖。`故在vue3中会有一个依赖收集暂停和回复机制。`

例如：

```js
// 如果不暂停收集
// 父组件vNode
const vNode = {
  type: 'div',
  children: [
    { type: Item }
  ]
}
// 子组件存在一个计算属性，是一个响应式对象，并且setup中直接访问了这个响应式数据
cosnt Item = {
  setup(props, ctx) {

        const context = computed(() => {
            // 其他代码
            return 'xxx'
        })

        context.value

        return function () {
            return {
                type: 'div',
            }
        }
    },
}
```

上面的代码中，父组件在最开始渲染时，会启动一个effect，然后渲染子组件的内容（这一步是同步的），子组件会创建并执行setup函数，此时子组件还未开启effect并执行渲染函数，但是，由于子组件中的setup创建并访问了响应式数据（context数据），此时进行依赖收集时，会发现副作用栈中存在一个effect，这是父组件开启的effect函数，此时会将这个父组件的effect函数作为这个context响应式数据的依赖进行收集。这并不符合我们的预期。所以，在vue中，执行组件的setup函数时，会暂停依赖的收集（不是暂停所有，而是暂停当前副作用栈之前的那些副作用函数的依赖收集），如果子组件调用watchEffect函数，那么还是会正确进行依赖收集这个watchEffect函数的（用了一个trackStack，它是一个布尔类型的，应该对应的是activeEffectStack栈是否需要进行依赖收集）

### 组件的设计

> 注：由于在这部分实现时，还未实现编译器相关内容，故这里面的代码都使用的是虚拟dom的形式来表示组件的内容。

实现参考：renderer-5.ts文件

渲染器有能力处理组件后，下一步我们要做的是，设计组件在用户层面的接口。这包括：

- 用户应该如何编写组件？
  - SFC：单文件组件
- 组件的选项对象必须包含哪些内容？
- 以及组件拥有哪些能力？等等。
  - 一个是对页面内容的封装
  - 另外一个是基于业务逻辑，让组件内容“可变”，即动态变化
- 实际上，`组件本身是对页面内容的封装，它用来描述页面内容的一部分。因此，一个组件必须包含一个渲染函数，即 render 函数，并且渲染函数的返回值应该是虚拟 DOM。`
  - 他也可以渲染一个空的节点，并且将组件视为逻辑的封装，这在react早期的高阶组件中很常见，不过现代开发中，都使用hooks或者组合API来封装复用的组件逻辑，这种用组件单独封装逻辑的情况已经基本上很少了
- 组件实例与组件的生命周期
  - 组件实例本质上就是一个状态集合（或一个对象），它维护着组件运行过程中的所有信息，例如注册到组件的生命周期函数、组件渲染的子树（subTree）、组件是否已经被挂载、组件自身的状态（data），等等。
  - 那么在组件挂载方法中（mountComponent中），则会创建组件实例，并执行相应的生命周期方法
  - 在最开始创建组件时，子组件的生命周期永远在父组件的beforeMount执行之后，mounted执行之前完成。因为父组件的虚拟dom的patch方法，在这两个生命周期之间执行。
  - 且在组件创建后，执行完成渲染函数后，对组件内容进行挂载，当完成挂载后会触发mounted生命周期。
    - 注意这里的mounted挂载方法中可以访问dom，由于是递归完成的虚拟dom的处理，当子组件完成patch函数时，它仅仅代表子组件的内容，插入到了父组件的dom中，并不一定能挂载到了真实的页面上。
- 组件属性
  - 我们将组件选项中定义的 MyComponent.props 对象和为组件传递的 vnode.props 对象相结合，最终解析出组件在渲染时需要使用的 props 和 attrs 数据。
  - 这里需要注意：在 Vue.js 3 中，没有定义在 MyComponent.props 选项中的 props 数据将存储到 attrs 对象中。
- props是一个对象，其是浅层的响应式的，SFC中的代码都会转换为props的访问（即使你解构了，在编译时也会转换）
- props 本质上是父组件的数据，当 props 发生变化时，其实就是父组件的数据变化了，则会触发父组件重新渲染。
  - 当父组件响应式数据发生变化时，父组件的渲染函数会重新执行，在更新过程中，渲染器发现父组件渲染的虚拟dom中包含组件类型的虚拟节点，所以会调用 patchComponent 函数完成子组件的更新
  - patchComponent 函数用来完成子组件的更新。我们把由父组件自更新所引起的子组件更新叫作子组件的被动更新。当子组件发生被动更新时，我们需要做的是：
    - 检测子组件是否真的需要更新，因为子组件的 props 可能是不变的；
    - 如果需要更新，则更新子组件的 props、slots 等内容。并触发子组件的update更新
- 为了方便render函数使用组件的数据，需要为组件封装一个渲染上下文传递到render函数中，包括组件state和props等。它实际上是组件实例的代理对象。在渲染函数内访问组件实例所暴露的数据都是通过该代理对象实现的（不是setup上下文，渲染上下文是指直接在渲染函数中通过this来访问组件实例上的props、state、emit方法等数据）
- 组件的卸载：
  - 组件的卸载需要注意嵌套组件，通常只会递归的卸载组件（基于组件的subTree的dynamicChildren进行递归卸载，在编译时确定），但是dom不会递归卸载，而是在最开始时用一个标记来标记需要卸载的根dom，后续的子树中的dom节点则不需要进行dom删除操作，会在一开始的根dom那里将整个dom树移除。

#### props：父组件的更新和子组件的更新

父组件通过组件属性给子组件传递props时，其本质上子组件获取到的props对象就是一个响应式对象，其属性就是子组件上定义的属性。

```html
<div>
  <MyComponent :title="title" :id="id" />
</div>
```

如上，父组件给子组件定义了两个属性，那么子组件接受的props对象就是一个`{ title: xxx, id: xxx }`对象，该对象是只读，且是一个浅层次的响应式对象。也就是对子组件来说，只有title和id属性是响应式的。且vue在父组件更新时，通过props来判断是否需要更新子组件时，该判断是一个浅比较（你可以自定义）

然而有一个很有意思的例子在于，如果子组件有一个list属性（number数组），其值是一个响应式对象（数组类型的）那么，我在父组件的点击事件中去更新list数组的第一个值，此时，你觉得父组件需要被更新吗？如下例子：

```vue
<!-- 这是父组件 -->
<template>
    <div>
        更新随机数{{Math.random()}}
        <button @click="handle2">更新list2中第一个元素的name</button>
        <TestChildren :list2="list2" />
    </div>
</template>

<script setup lang="ts">

import TestChildren from "./TestChildren.vue";
import {ref} from "vue";

const list2 = ref([
    1,2,3
])
function handle2() {
  // 仅更新list2的第一个值
    list2.value[0] = 10
}
</script>

<!-- 这是子组件 -->
<template>
    <div>
        更新随机数{{Math.random()}}
        <div>{{ list2[0] }}</div>
    </div>
</template>

<script setup lang="ts">
import {watchEffect} from "vue";

const { list } = defineProps<{
    list2: number[]
}>()

</script>
```

从感觉上，父组件点击按钮更新了它组件自己的响应式数据的值，那么它应该被更新，并且连带着子组件也更新才对，然而，其实并非如此，事实上，上面的实例在执行时，只有子组件被重新渲染了，父组件并不会重新渲染。为什么？我来解释

- 首先，对于父组件的渲染函数中，它依赖的响应式数据是list2这个ref本身，也就是这个list2.value更新时，父组件才会重新渲染，这是响应式依赖收集的细节，父组件的渲染函数仅仅依赖了list2.value这个响应式数据，所以，你更新`list.value[0]`的值是不会触发父组件更新的副作用函数的。
  - 所以一个组件的所谓更新，其实就是重新执行那个调用了组件渲染函数的副作用函数，此时只有template里面的代码，这里面所使用的响应式数据，才是父组件`重新渲染`的响应式依赖。即使你的父组件创建了再多的响应式数据，只要template中没有依赖或者访问，那么这些响应是数据无论如何更新，都不会触发父组件的`重新渲染`这个概念。
- 其次，对于子组件来说，它接受的props是一个`{ list2: number[] }`对象，这个对象是一个仅读的浅响应式对象，子组件在template或者watchEffect中访问props.list2是会被其进行依赖收集的，同时，list2的值本身也是一个响应式对象，只不过该对象是在父组件创建的。那么子组件访问了`props.list2[0]`，那么子组件的template渲染函数会存在两个依赖收集：`props.list2和list2[0]`。当这两个响应式数据更新时，则会直接触发子组件的渲染函数更新副作用函数的重新执行，即子组件重新渲染。
- 还有一个点需要注意的时，父组件更新时，即使子组件没有更新，那么其子组件拿到的props的值仍然是最新的（因为父组件重新渲染时，如果子组件需要更新，则也会更新子组件实例中的props属性，理论上子组件不需要重新更新，那么子组件的props说明没有变过，就是最新的）
- `当父组件对比重新渲染，发现子组件的props变化了时，即使子组件本身没有使用任何的props（例如渲染函数里面或watchEffect里面），那么子组件仍然会被重新渲染，从而执行渲染函数`
  - 注意，这里仅针对组件定义过的props，即defineProps中定义的属性，如果是defineEmits中定义的事件处理函数变化了，子组件是不会触发重新渲染的(不过其子组件触发事件时调用仍然是最新的事件处理函数)
  - 而vue的模板编译时，存在非常多的优化，例如事件，使用一个箭头函数，那么其编译成的渲染函数会将函数缓存，不会每次重新渲染时都生成新的箭头函数，其次如果是表达式，vue3在编译时同样会做缓存，防止因为这种情况导致不必要的更新。
  - 但是，注意，这里貌似只针对事件的函数进行优化，如果定义的一个属性是函数，那么vue不会对其进行函数缓存的优化。
    - `说明：Vue 3 编译器仅对「事件绑定（@xxx）」的箭头函数做缓存优化（且仅对事件的表达式进行函数包装），而对「props 传递（:xxx）」的箭头函数不优化`
    - 这本质是「事件系统的设计约定」和「props 语义的不确定性」导致的：事件绑定的函数是 “回调语义”（稳定且仅用于触发），而 props 函数是 “属性语义”（可能被主动调用 / 多次读取，缓存会破坏语义）。
    - 还有一个例外就是v-for循环中的箭头函数事件：v-for 场景下，事件回调的箭头函数无法被稳定缓存，本质是「循环的动态上下文（如 index/item）会导致函数语义不唯一」，编译器无法保证缓存的安全性，因此放弃自动缓存。
  - 所以在给组件绑定事件时，通常都不会导致其重新渲染，因为函数通常都不会变化，那么在对比props时都不会是导致更新的原因。但是有一个特例就是for循环中
- 父组件更新时，是一个递归更新的过程，如果子组件需要更新，那么子组件的更新是同步的。A - B - C时，那么更新时机是嵌套的（A（B（C）））
- `在vue的实现中，父组件中更新子组件时，他会直接调用子组件的update方法，其实就是那个类似watchEffect函数返回的effectFn方法来执行重新渲染，而不是通过设置子组件的props来隐式的触发子组件的重新渲染。`
  - 而在子组件的udpate方法中，他会更新自己的props，`且在更新组件自己的props时，是会暂停一些effect副作用函数的收集之类的。`
  - 这里是存在嵌套effect，但是注意，它不像你直接嵌套两个watchEffect这样，他所谓的嵌套，是在一个watchEffect中，去执行一个已经存在的由watchEffect函数返回的effectFn，所以它并不会重新通过watchEffect来创建一个新的副作用函数，所以在这个嵌套的effect中，他可以正常的清理旧的依赖而不会多次进行依赖收集。

所以，对于组件和响应式数据，你需要将他们分开来看待，组件的渲染函数（template）会被包装到一个类似watchEffect的函数中，那么渲染函数中使用到的响应式数据更新时，组件才会被重新渲染，这和响应式数据的来源，例如是不是组件创建的没有任何关系。而组件内部的watchEffect等方法的执行也和组件是否重新渲染没有任何关系，它同样是需要依赖的响应式数据更新时，它才会被重新更新。而组件的整个setup函数，它仅在组件创建时被执行，用来获取一些数据，后续的更新之类的操作，和这个函数本身没有任何关系。

还有，setup函数仅会在组件创建时被创建，那么我们很容易知道，组件中获取到的props对象它本身不会变，即使组件重新渲染了，props对象的引用都是同一个对象。不过好像变不变，对于我们来说其实也不重要。但是注意，props变量本身不具有响应式。

```ts
const props = defineProps<{
    list: { id: number; name: string }[]
    list2: number[]
}>()
```

#### 关于pinia

从上面的了解可以看出来，vue的响应式数据和vue组件之间并没有太大的强关联，在vue组件中创建一个响应式数据，仅仅代表该响应式数据是在组件中创建的，且伴随着组件销毁时被垃圾回收掉，但是其响应式和依赖收集同组件之间没有任何关联。

那么换个角度，把创建响应式数据的代码放到组件之外，组件仅仅是引用它，那么自然而然该数据就会成为一个全局响应式数据，这不就是和我们的全局状态是同一个东西了吗？所以我们是否就可以理解pinia本质上就是利用了vue3中的响应式系统，然后对其进行了一些语法功能上的封装，使其符合一个全局状态管理库的约定设计，即pinia按照「全局状态管理」的场景做了标准化、工程化的封装，让开发者能更规范地管理全局响应式状态。

#### setup函数设计

其实setup函数的设计和选项式API的流程差不多，只不过存在如下差异：

- setup函数本身接受一个props和setup的context上下文
- 如果在setup返回一个对象时，则其代表state，而如果返回一个render函数，则用来代替选项式API的render，这时候就不需要返回state了，因为render在setup里面，自然可以访问到各种响应式数据

setup函数返回值的编译，参考下面的代码：

```vue
<script setup>
import { ref, watchEffect, reactive, isReactive, h, defineProps } from 'vue'
const name = ref('zhou')

// 使用原始数据修改为p1添加属性时，此时副作用重新执行

let update = () => {
    console.log('点击')
}

function handle() {
  name.value = 'li'
  update = () => {
    console.log('点击-newUpdate')
  }
}

</script>

<template>
  <div @click="update">
    {{name}}
  </div>
</template>

```

这里的代码在编译后，有一个很巧妙的思路，由于setup返回里面的所有变量都放在一个对象中，那么在render函数中，绑定了一个可变的变量会怎么样呢？例如上面的update可以被修改，而render中好像看起来是访问范围update这个变量的，再不济，也是访问setupContext.update属性的，那如何保证render中获取的是最新的呢？其实很简单，它在编译时，如果发现一个变量时let声明的，即可变的，那么他会使用get和set作为这个属性，即编译成下面这个：

```js
// setup返回的对象中，update属性时一个get和set访问器。
const __returned__ = { props, name, get update() { return update }, set update(v) { update = v }, handle, ref, watchEffect, reactive, isReactive, h }
Object.defineProperty(__returned__, '__isScriptSetup', { enumerable: false, value: true })
return __returned__
```

#### 渲染函数上下文

组件的渲染函数上下文支持让render函数通过this.xxx或者renderContext.xx来访问组件定义的data或者setup返回的数据。本质上这个renderContext就是一个代理对象，在get和set中判断这个key是否存在于组件的state或者setup返回的对象中，如果存在，则返回相应的值。

```ts
/**
 * 封装组件上下文
 * 使用代理封装
 */
const renderContext = new Proxy(instance, {
    get(target, key) {
        if (key === '$slots') {
            return slots
        }
        if (state && (key in target.state)) {
            return Reflect.get(state, key)
        }
        else if (setupState && (key in setupState)) {
            return Reflect.get(setupState, key)
        }
        else if (key in target.props) {
            return Reflect.get(target.props, key)
        }
        return Reflect.get(target, key)
    },
    set(target, key, value) {
        if (key === 'props' || key === 'attrs' || key === 'state' || key === '$slots') {
            console.warn(`不能直接修改 ${String(key)} 属性`)
            return true
        }
        if (state && (key in target.state)) {
            Reflect.set(target.state, key, value)
        }
        else if (setupState && (key in setupState)) {
            Reflect.set(setupState, key, value)
        }
        else if (key in target.props) {
            console.warn(`不能直接修改props ${String(key)} 属性`)
        }
        else {
            Reflect.set(target, key, value)
        }
        return true
    },
})
```

#### 组件事件和emit

组件事件和为组件定义一个函数式的props没有太大的区别，只不过使用emit触发组件事件，则需要再组件内部再定义emit对象，其本质上和在组件内部定义props没有太大区别，目的就是让vue在内部能够区分props哪些是props属性，哪些是事件。但是，事件由于具有约定性，编译器会对事件在模板中的箭头函数、表达式进行缓存和包装，且不会在事件的函数变化时触发子组件的更新。因为事件系统约定它是被动触发的，每次触发时，都会获取组件上最新的事件函数来调用，所以不需要触发组件本的渲染函数的重新渲染

但是v-for循环中的箭头函数事件，仍然不会被缓存，因为v-for中，函数具有循环上下文，如果复用同一个箭头函数，那么该监听函数引用的循环上下文是之前的：例如`[1,2,3]`数组变成`[9,2,3]`数组，如果key使用的index，那么复用组件实例，仅更新组件内容时，如果其函数不更新，那么之前的旧函数获取到的数组值就是1，而不是9了。而且，函数缓存利用全局的cache数组，循环中，列表的长度不固定，且如果长度变化了，那么缓存位置需要重新更新，这成本和实现太麻烦。且事件本身不会触发子组件渲染，仅进行更新组件存储的事件的属性，性能也不低。

如果在for循环中绑定事件，如果不需要从传参，则将其抽离为一个函数，而如果需要传参，那也没有太好办法。每次循环都会创建一个新的函数，但是不会触发组件的重新渲染。

组件的事件也会被编译为下面的虚拟dom内容：

```js
const vNode = {
  type: MyComponent,
  props: {
    onClick: () => void
  }
}
```

- 作为虚拟dom来说，一个事件和普通的dom的事件没有任何差别。不过貌似在一些组件库中，还是使用emit配置类定义和触发事件。如果你不用emit来定义事件，则可以使用直接在props中定义一个on开头的属性也是可以的。然后直接在组件中调用这个属性函数即可。此时在使用组件时就不要使用`@`来监听事件了，而是使用`v-bind:onClick`来设置组件属性。不过还是那个问题，vue在编译时对`@`监听的事件做了缓存优化，不会导致组件产生不必要的更新。
  - 其实不管是`@click`还是`v-bind:onClick`这种，其实都会被编译为虚拟dom的onClick。都会被正常的emit触发，这是vue做了兼容处理
- vue3中，组件获取的props对象，是没有通过defineEmits定义的那些属性的，感觉他们是分开的，props对象仅处理使用defineProps定义的那些。但是emit函数有做处理（从源码得知），即你在组件中使用defineProps定义一个onUpdate属性，那么你可以直接通过props.onUpdate或者emit('update')来触发这个事件函数执行。emit函数它在执行时，是直接从组件对应的`虚拟dom的props原始属性`中去匹配，然后进行执行。
  - 这里的分开是指在组件中获取的props对象，但是vue内部实现时是分开还是聚合在一起的则不确定，如果分开，则类似`<component is=xx />`这种内置组件的props透传处理起来就比较麻烦了。

`注意：emit触发事件时，是同步执行该事件处理函数的`

一个值得讨论的问题：`如果你在watchEffect中使用emit来触发事件，那么由于在emit的过程中会执行事件处理函数，如果这个事件处理函数的执行本身也访问了其他响应式依赖，那么该watchEffect也会将这些响应数据作为依赖收集进去`，参考这个[github issue](https://github.com/vuejs/core/issues/6669)。

怎么说呢，尤大在评论回复：watchEffect根据定义，它会在同步操作期间跟踪数据。如果你想要安全、非跟踪性的副作用，请使用watch回调函数——它明确地将“跟踪”和“副作用”分开——几乎就是为这种情况设计的。

watchEffect的定义确实如此，他会在该函数执行的同步期间收集到所有响应式数据并将其作为依赖。而`emit本身是同步的且同步触发组件的事件处理函数`，这个概念本身需要理解才明白其行为。

`props对象和组件上的事件对象，不是同一个（即组件需要emit触发的事件不再props上），而且，你貌似拿不到`

#### 插槽的原理和实现

在使用一个组件时，为组件添加子元素，那么这些子元素，会被编译为一个插槽函数，且带有具名的对象，假设渲染该组件的虚拟dom为：

```js
const vNode = {
  type: MyComponent,
  children: {
    header: () => vNode,
    default: () => vNode,
  }
}
```

上面的children和普通的dom中的子节点不太一样，因为它是一个对象，带有key-value，这是为了方便子组件在渲染时，方便获取对应名称的插槽内容，他也可以是一个数组，然后基于name，这样子组件在匹配具名插槽内容时就需要遍历，感觉没有必要。

此时组件MyComponent的渲染函数则会变成：

```js
// MyComponent render
function render() {
  return {
    type: 'div',
    children: [
      {
        type: 'header',
        children: [this.$slots.header]
      },
      {
        type: 'div',
        children: [this.$slots.default]
      },
    ],
  },
}
```

可以看到，渲染插槽内容的过程，就是调用插槽函数并渲染由其返回的内容的过程。这与 React 中 render props 的概念非常相似：即父组件提供一个渲染内容的函数，子组件在合适的位置调用该函数获取内容并插入到这个为止。

而`<slot>` 标签其实就是被编译为插槽函数的调用，通过执行对应的插槽函数，得到外部向槽位填充的内容（即虚拟 DOM），最后将该内容渲染到槽位中。

如下例子：

```js
/**
 * 渲染组件的slots插槽
 */
const Item: Component = {
    name: 'Item',
    props: {
        id: Number,
        data: Object,
    },
    setup(props, ctx) {

        const context = computed(() => {
            return `id：${props.id}，name：${props.data.name}，age：${props.data.age}`
        })

        return function render() {
            return {
                type: 'div',
                props: {
                    'data-id': props.id,
                },
                children: [
                    {
                        type: 'div',
                        children: '--------------------我是Item组件--------------------',
                    },
                    // 渲染插槽内容（单个节点）
                    ctx.slots.default ? ctx.slots.default() : {
                        type: 'div',
                        children: '我是Item组件的默认插槽内容',
                    },
                    // 具名插槽（多个节点）通常整个的插槽渲染封装一个函数，这样可以区分是否存在多个节点。
                    {
                        type: Fragment,
                        // 为插槽传参
                        children: ctx.slots.context ? ctx.slots.context({ context: context.value }) : [],
                    }
                ],
            }
        }
    },

}

// 父组件的渲染内容
const vNode = {
    type: Item,
    props: {
        id: this.id,
        data: this.data,
    },
    // children存储插槽内容，它是一个对象，而不是普通的数组
    children: {
        'default': () => {
            return {
                type: 'h3',
                children: '我是h3标题，插入默认插槽内容',
            }
        },
        'context': (slotProps: any) => {
            // 这里函数会保留父组件的作用域，所以子组件重新渲染时，这里能够访问到父组件的最新数据
            return [
                {
                    type: 'div',
                    children: `插入具名context插槽，下面是内容-id(${this.id})`,
                },
                {
                    type: 'p',
                    children: slotProps.context,
                }
            ]
        },
    },
}
```

注意：

- 由于插槽被编译为函数调用，所以如果组件插槽的元素中使用的响应式数据收集的依赖仅仅针对组件本身，而不会收集到组件所在的父组件（假设父组件没有直接使用该响应式数据）
- 但是目前看来仅针对普通组件，或者不同组件有不同的编译实现，例如vue3中的keepAlive的插槽，它是内部组件，并非走普通插槽逻辑那样编译为函数调用，还有其他组件是否有特殊处理并不确定。
- 注意：上面仅用作无编译下的演示，在实际中，还需要配合编译器

#### 注册生命周期

生命周期本质上是在合适的运行时机，调用用户注册的生命周期回调函数。

不过在vue3中，有一部分组合式 API 是用来注册生命周期钩子函数的，例如 onMounted、onUpdated 等，setup由于是类似函数式组件，故需要一个机制为非对象式组件来注册一些内容并绑定到该组件的实例，例如生命周期、watchEffect（随组件自动卸载）、getCurrentInstance等

- 类似响应式依赖收集，将当前的effect放到一个activeEffect栈中，然后所有响应式数据就知道收集哪个Effect函数作为副作用函数了。生命周期也是类似。
- 我们添加一个当前组件创建时的组件实例栈，类似componentStack，这样，一些需要绑定到组件实例的API，例如生命周期API，就可以将其和当前组件实例绑定了。
- 注意，组合式API，setup里面没有onBeforeCreate（只有选项式API才有，我就说嘛），它是从挂载开始的。
- vue文档：注意，组合式 API 中的 setup() 钩子会在所有选项式 API 钩子之前调用，beforeCreate() 也不例外。这里是指setup的执行时机在所有选项式API钩子之前。

### 异步组件

实现参考：renderer-6.ts文件

从根本上来说，异步组件的实现不需要任何框架层面的支持，用户完全可以自行实现（注意，这里说的是框架，因为本质上是将组件的加载变成一个异步的，即使用时才加载）

因为组件本质上也是一个es模块（sfc），或者说就是一个对象。通常使用import()动态导入一个组件，获取到组件的对象，然后配合vue的内置动态组件：`<component is=xxx />`实现异步组件。

虽然用户可以自行实现组件的异步加载和渲染，但整体实现还是比较复杂的，因为一个完善的异步组件的实现，所涉及的内容要比上面的例子复杂得多。通常在异步加载组件时，我们还要考虑以下几个方面。

- 如果组件加载失败或加载超时，是否要渲染 Error 组件？
- 组件在加载时，是否要展示占位的内容？例如渲染一个 Loading 组件。
- 组件加载的速度可能很快，也可能很慢，是否要设置一个延迟展示 Loading 组件的时间？如果组件在 200ms 内没有加载成功才展示 Loading 组件，这样可以避免由组件加载过快所导致的闪烁。
- 组件加载失败后，是否需要重试？

为了替用户更好地解决上述问题，我们需要在框架层面为异步组件提供更好的封装支持，与之对应的能力如下。

- 允许用户指定加载出错时要渲染的组件。
- 允许用户指定 Loading 组件，以及展示该组件的延迟时间。
- 允许用户设置加载组件的超时时长。
- 组件加载失败时，为用户提供重试的能力。

异步组件本质上是通过封装手段来实现友好的用户接口，从而降低用户层面的使用复杂度。

```ts
/**
 * 实现异步组件defineAsyncComponent方法
 */
function defineAsyncComponent(options: {
    load: (testFlag?: boolean) => Promise<Component>
    loadingComponent?: Component
    errorComponent?: Component
    // 延迟多少时间后显示加载中组件
    delay?: number
    // 超时时间，超过该时间则显示错误组件
    timeout?: number
    // 重试次数，默认为0，表示不重试
    retryCount?: number
} | (() => Promise<Component>)): Component {


    const opt = typeof options === 'function' ? { load: options } : options

    const { load, delay, loadingComponent, errorComponent, timeout, retryCount = 0 } = opt

    function retryTools(fn: () => Promise<Component>, retryCount: number): Promise<Component> {
        return new Promise((resolve, reject) => {

            fn().then((comp) => {
                resolve(comp)
            }).catch(error => {
                if (retryCount === 0) {
                    reject(error)
                }
                else {
                    resolve(retryTools(fn, retryCount - 1))
                    // 等同于下面这个
                    // retryTools(fn, retry - 1).then(resolve, reject)
                }
            })


        })
    }

    return {
        name: 'AsyncComponentWrapper',
        setup(props, {slots}) {

            const componentRef = shallowRef<Component | null>(null)
            const loadedRef = ref(false)
            // 加载异常相关
            const isErrorRef = ref(false)
            const errorInfoRef = ref<any>(null)
            // 延迟相关，是否延迟显示loading组件
            const isDelayRef = ref(!!delay)

            // 定时器相关
            let timeoutTimer: any
            let delayTimer: any

            function loadComponent(testFlag?: boolean) {
                retryTools(() => {
                    return load(testFlag)
                }, retryCount).then((component: Component) => {
                    /**
                     * 如果超时或者已经出错了，则不再设置组件
                     */
                    if (!isErrorRef.value) {
                        componentRef.value = component
                        loadedRef.value = true
                    }

                }).catch((error) => {
                    // 错误处理
                    if (!isErrorRef.value) {
                        isErrorRef.value = true
                        errorInfoRef.value = error
                    }

                }).finally(() => {
                    clearTimeout(timeoutTimer)
                    clearTimeout(delayTimer)
                    isDelayRef.value = false
                })

                /**
                 * 处理超时逻辑
                 */
                if (timeout !== undefined) {
                    timeoutTimer = setTimeout(() => {
                        isErrorRef.value = true
                        errorInfoRef.value = 'loading timeout'
                    }, timeout)
                }


            }
            loadComponent(true)

            /**
             * 占位空节点
             */
            const EmptyComponent = {
                type: Fragment,
                children: [],
            }

            /**
             * 处理延迟逻辑
             */
            if (delay !== undefined && delay !== 0) {
                delayTimer = setTimeout(() => {
                    isDelayRef.value = false
                }, delay)
            }

            /**
             * 重试加载组件
             */
            function retry(testFlag?: boolean) {
                isErrorRef.value = false
                loadComponent(testFlag)
            }

            onUnMounted(() => {
                clearTimeout(timeoutTimer)
                clearTimeout(delayTimer)
            })

            return function render() {
                if (isErrorRef.value) {
                    return errorComponent ? {
                        type: errorComponent,
                        props: {
                            error: errorInfoRef.value,
                            retry,
                        },
                    } : EmptyComponent
                }

                if (!loadedRef.value) {
                    if (isDelayRef.value) {
                        return EmptyComponent
                    }
                    return loadingComponent ? {
                        type: loadingComponent,
                    } : EmptyComponent
                }

                return {
                    type: componentRef.value!,
                    props: {
                        ...props,
                    },
                    children: slots,
                }
            }
        }
    }

}
```

### 内建组件和模块

我们将讨论 Vue.js 中几个非常重要的内建组件和模块，例如 KeepAlive 组件、Teleport 组件、Transition 组件等，它们都需要渲染器级别的底层支持。

#### KeepAlive 组件的实现原理

实现参考：renderer-7.ts文件

KeepAlive 一词借鉴于 HTTP 协议。在 HTTP 协议中，KeepAlive 又称 HTTP 持久连接（HTTP persistent connection），其作用是允许多个请求或响应共用一个 TCP 连接。在没有 KeepAlive 的情况下，一个 HTTP 连接会在每次请求/响应结束后关闭，当下一次请求发生时，会建立一个新的 HTTP 连接。频繁地销毁、创建 HTTP 连接会带来额外的性能开销，KeepAlive 就是为了解决这个问题而生的。

HTTP 中的 KeepAlive 可以避免连接频繁地销毁/创建，与 HTTP 中的 KeepAlive 类似，Vue.js 内建的 KeepAlive 组件可以避免一个组件被频繁地销毁/重建。例如在一组tab中，将其tabItem给包装到KeepAlive组件中。

KeepAlive 的本质是缓存管理，再加上特殊的挂载/卸载逻辑

- KeepAlive应该有上下文的概念，即如果父组件中的内容中，有一个KeepAlive组件，那么在父组件存活期间，其KeepAlive是生效的，如果父组件被卸载了，那么其KeepAlive的缓存应该失效才对。
- KeepAlive 组件的实现需要渲染器层面的支持。这是因为被 KeepAlive 的组件在卸载时，我们不能真的将其卸载，否则就无法维持组件的当前状态了。
- 正确的做法是，将被 KeepAlive 的组件从原容器搬运到另外一个隐藏的容器中，实现“假卸载”。当被搬运到隐藏容器中的组件需要再次被“挂载”时，我们也不能执行真正的挂载逻辑，而应该把该组件从隐藏容器中再搬运到原容器。这个过程对应到组件的生命周期，其实就是 activated 和 deactivated
- activated 和 deactivated会对于整个KeepAlive树生效（递归对子树全部缓存），也就是说一个组件只要在KeepAlive组件数的下面，那么它就是被缓存的，那么当组件被插入到 DOM或者卸载DOM 时调用。
  - 此时，该组件不会触发onUnmounted钩子，因为没有被真正卸载

KeepAlive 组件与渲染器的结合非常深。

- 首先，KeepAlive 组件本身并不会渲染额外的内容，它的渲染函数最终只返回需要被 KeepAlive 的组件，我们把这个需要被 KeepAlive 的组件称为“内部组件”
- KeepAlive 组件会对“内部组件”进行操作，主要是在“内部组件”的 vnode 对象上添加一些标记属性，以便渲染器能够据此执行特定的逻辑。
- KeepAlive仅支持一个子节点（可以使用component来实现动态组件）本质其实是和v-if-else这种是一样的，你也可以直接用if-else这种，只要保证同一时间只有一个子节点即可。
- 对于dom节点，keepAlive不会进行缓存。
- 注意：在vue3中对于keepAlive中的子节点，其编译器不会将其编译为类似defalut那种slot插槽，可能是实现不同吧。
  - 目前自己的实现中，还是得用函数插槽，不然由于还没有编译器来对keepAlive组件做特殊处理，如果是让响应式数据收集到了父组件，那由于keepAlive的props没有更新，导致无法重新执行keepAlive组件的副作用函数，那就无法更新了。
- keepAlive可以多同一个组件的多次使用进行缓存，只不过需要添加key来区分。例如`<KeepAlive><CompA v-if="current === CompA" key="1" /><CompA v-else key="2" /></KeepAlive>`。此时keepAlive就会根据组件名+key来进行缓存。
- 在默认情况下，KeepAlive 组件会对所有“内部组件”进行缓存。但有时候用户期望只缓存特定组件。为了使用户能够自定义缓存规则，我们需要让 KeepAlive 组件支持两个 props，分别是 include 和 exclude。
- 缓存管理，基于max属性实现LRU（一种缓存淘汰算法），即根据缓存命中的时间来确定要删除哪些。最近访问的数据会放到列表的最前面。缓存满时，从列表最后面开始移除

keepAlive的实现：

- 定义一个组件，在setup中使用一个map来缓存需要缓存的组件VNode，创建一个空的div节点用来存储不活跃的组件对应的dom节点
- 组件卸载时清空节点和卸载缓存的组件
- 返回的渲染函数中，获取子节点。如果是dom则直接返回不进行缓存，如果是组件，则判断是否已经存在缓存中，如果已经存在则复用缓存中的VNode的el和组件实例，避免重复创建实例和dom。
- 在keepAlive存活期间，其组件树中的同一个组件所对应的dom节点只有一份，可以一直复用，通常保证根节点不变即可（但是需要注意合理更新）
- 在组件非激活期间，暂停其副作用函数的执行
- 使用一些字段标记keepAlive组件本身，以及扩展keepAlive组件实例和虚拟dom的一些字段，用来在渲染器进行渲染、卸载时的判断。这就是为什么和渲染器结合非常深。
  - keepAlive组件实例扩展两个_activate、_deActivate方法，用来在子组件需要卸载和激活时移动相关的子组件的dom到真实页面中。keepAlive下的子组件的卸载不是真的删除dom节点，而是将其移动到一个缓存节点中，后续激活时再移动到真实页面中，避免dom的创建。
  - keepAlive组件本身使用一个__isKeepAlive标识来表示是一个keepAlive组件，例如在组件实例上注入keepAliveCtx上下文
  - keepAliveCtx上下文：在创建keepAlive组件时注入，传递一些方法给keepAlive组件本身，用来移动或者创建dom
  - VNode中的新增：
    - keptAlive：标识组件是否已经被缓存，如果已经缓存，则不会走组件挂载
    - shouldKeepAlive：标识组件是否应该被keepAlive缓存，如果为true，则不会真正卸载
    - keepAliveInstance：keepAlive的组件实例，方便渲染器调用keepAlive组件实例中的_activate、_deActivate方法
- 基于组件名称匹配include和exclude

#### Teleport 组件的实现原理

实现参考：renderer-8.ts文件

- 通常情况下，在将虚拟 DOM 渲染为真实 DOM 时，最终渲染出来的真实 DOM 的层级结构与虚拟 DOM 的层级结构一致。但是列入弹框、蒙层这种通常会渲染到body下，以保证能够覆盖其他所有元素。
- 在 Vue.js 2 中我们只能通过原生 DOM API 来手动搬运 DOM 元素实现需求。这么做的缺点在于，手动操作 DOM 元素会使得元素的渲染与 Vue.js 的渲染机制脱节，并导致各种可预见或不可预见的问题。
- 于是 Vue.js 3 内建了 Teleport 组件。该组件可以将指定内容渲染到特定容器中，而不受 DOM 层级的限制。
- 通过为 Teleport 组件指定渲染目标 body，即 to 属性的值，该组件就会直接把它的插槽内容渲染到 body 下，而不会按照模板的 DOM 层级来渲染，于是就实现了跨 DOM 层级的渲染。

与 KeepAlive 组件一样，Teleport 组件也需要渲染器的底层支持。首先我们要将 Teleport 组件的渲染逻辑从渲染器中分离出来（因为他的渲染和普通的dom、组件渲染有很大差别），这么做有两点好处：

- 可以避免渲染器逻辑代码“膨胀”；
- 当用户没有使用 Teleport 组件时，由于 Teleport 的渲染逻辑被分离，因此可以利用 TreeShaking 机制在最终的 bundle 中删除 Teleport 相关的代码，使得最终构建包的体积变小。

设计说明：

- Teleport 只改变了渲染的 DOM 结构，它不会影响组件间的逻辑关系。也就是说，如果 Teleport 包含了一个组件，那么该组件始终和这个使用了 Teleport 的组件保持逻辑上的父子关系。
- 使用disabled属性来禁用Teleport的特性，让其变为一个普通的组件
- 一个可重用的 Modal 组件可能同时存在多个实例。对于此类场景，多个 Teleport 组件可以将其内容挂载在同一个目标元素上，而顺序就是简单的顺次追加，后挂载的将排在目标元素下更后面的位置上，但都在目标元素中。
- Teleport 是否拥有组件的性质是由框架本身决定的。通常，一个组件的子节点会被编译为插槽内容，不过对于 Teleport 组件来说，直接将其子节点编译为一个数组即可。
  - Teleport组件是否需要setup和render渲染方法？也就是它是否需要走组件那一套流程？感觉是不需要的，也不需要组件实例。
  - 思考Teleport的作用是什么？它将其children子节点渲染到一个指定的位置，它本身没有任何内容，仅仅是渲染操作，感觉和Fragment非常类似，仅仅拥有一个VNode虚拟节点进行占位，它可以直接处理子节点，并修改其子节点patch时的容器container即可。
- 当to变更时，则会直接更新子节点后，按照子节点顺序移动即可（直接添加到目标的最后）

```ts
/**
 * 内置组件Teleport的实现
 */

const Teleport: Component = {
    name: 'Teleport',
    __isTeleport: true,
    props: {
        to: String,
        disabled: Boolean,
    },

    /**
     * 自行处理Teleport的挂载和卸载之类的逻辑
     */
    process(oldVNode, newVNode, container, anchor, internals) {
        const  { props, children } = newVNode
        const { patch, elementAPI, patchChildren, move } = internals!
        const to = elementAPI.querySelector(props!.to)
        if (!oldVNode) {
            // 挂载
            if (Array.isArray(children)) {
                children.forEach((v: VNode) => {
                    patch(null, v, props!.disabled ? container : to, anchor)
                })
            }
        }
        else {
            const oldVNodeTo = elementAPI.querySelector(oldVNode.props!.to)
            // 更新，第三个参数可以根据disabled来确定是否是to还是container
            patchChildren(oldVNode, newVNode, to)
            if (oldVNodeTo !== to || props!.disabled !== oldVNode.props!.disabled) {
                // 说明需要移动，从新的to变成旧的to，或者从禁用变成启用，或者同时变更了
                if (Array.isArray(children)) {
                    children.forEach((v: VNode) => {
                        move(v, props!.disabled ? container : to)
                    })
                }
            }

        }
    },
}

```

#### Transition 组件的实现原理

实现参考：renderer-9.ts文件

实际上，Transition 组件的实现比想象中简单得多，它的核心原理是：

- 当 DOM 元素被挂载时，将动效附加到该 DOM 元素上；
- 当 DOM 元素被卸载时，不要立即卸载 DOM 元素，而是等到附加到该 DOM 元素上的动效执行完成后再卸载它。

当然，规则上主要遵循上述两个要素，但具体实现时要考虑的边界情况还有很多。不过，我们只要理解它的核心原理即可，其实对于动画，我们可以参考css动画的定义。

- 一个动画首先就是需要定义开始、结束和中间态，中间态由浏览器帮我们自行完成，那么我们只需要确定dom元素在最开始和最终的状态即可。
- 以dom加入到页面元素为例，要完成一个dom的进场动画，参考css动画的from、to语法，我们使用enter-from、enter-to和enter-active来为dom的挂载定义其动画周期。
  - enter-from：用来确定dom元素的最初始状态，例如：`.enter-from { transform: translateX(200px); }`
  - enter-to：用来描述dom元素最终的状态，通常它也是dom在页面中应该处于的实际位置，例如：`.enter-from { transform: translateX(0); }`
    - 他的作用就是让dom产生不同的状态，以便浏览器可以实现DOM 元素在两种状态间的切换
  - enter-active：用来表示dom元素正处于执行进场动画的过程中。通常我们会为其添加相应的动画css，例如：`.enter-active { transition: transform 1s ease-in-out; }`
- 其实你参考一个普通的css过渡动画的写法，在一开始加载页面时，我们需要确定dom元素处于不可见或者透明的初始状态，然后再为其添加相关的transition或者animation的css样式，使其触发动画的执行。
- 我们将上面的过渡流程结合到我们的渲染器中，那么大致的流程就是：
  - 创建dom
    - 应用dom的enter-from和enter-active
  - 挂载dom
  - 应用dom的enter-to并移除enter-from让dom产生两种不同的状态
    - 这会触发浏览器的过渡动画
  - 过渡完成后，需要移除enter-to和enter-active（通过监听dom的transitionend等过渡或者动画事件）

注意：

- 在实现时，创建dom和挂载dom会在同一个事件周期完成，这没有什么问题，不过为了保证dom在浏览器中产生两个不同的状态，我们`在应用enter-to时需要确保dom已经被真实渲染到页面中后，再为其添加enter-to`，不然浏览器会直接在当前帧将enter-to（如果enter-to的样式写在enter-from的后面的话）中的样式作为最终样式绘制到页面上，自然也不会出现两种dom状态，从而无法触发状态不同dom状态的过渡了。
  - 为了解决这个问题，我们需要在下一帧执行状态切换，使用requestAnimationFrame
  - 不过这里有一个bug：使用 requestAnimationFrame 函数注册回调会在当前帧执行，除非其他代码已经调用了一次 requestAnimationFrame 函数。需要通过嵌套一层 requestAnimationFrame 函数的调用即可解决上述问题。(即双rAF)

实际上，我们可以对上述为 DOM 元素添加进场过渡的过程进行抽象：

- 创建dom
- 阶段：动效开始之前：beforeEnter，应用enter-from和enter-active
- 挂载dom
- 阶段：动效开始：enter（在页面刷新的下一帧），应用enter-to和移除enter-from
- 阶段：动效结束，移除 enter-to 和 enter-active 类

dom离开和进入的阶段是一样的，只不过其变成了：

- 阶段：离开之前：beforeLeave，应用leave-from和leave-active
- 阶段：动效开始：enter（在页面刷新的下一帧），应用leave-to和移除leave-from
- 阶段：动效结束，移除 leave-to 和 leave-active 类
- 卸载dom

需要注意的点：

- 离开动效通常在dom卸载时应用，而这时候我们不能马上从页面中卸载dom，而是应该在动效结束时再卸载dom，这需要渲染器支持。
- Transition 组件的实现原理与 原生 DOM 的过渡原理一样。只不过，Transition 组件是基于虚拟 DOM 实现的。我们在为原生 DOM 元素创建进场动效和离场动效时能注意到，整个过渡过程可以抽象为几个阶段，这些阶段可以抽象为特定的回调函数
- Transition组件仅接收一个子节点（你可以自行封装专门用于处理动画的组件）
- 优化Transition组件的mode模式（因为存在动画队列，新增了非常多的判断条件处理不同的边界情况）
