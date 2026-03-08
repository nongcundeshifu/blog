---
title: vue.js设计与实现-响应式
date: 2026-03-04 21:21:22
tags: 
  - vue
categories:
  - vue
---

仍然是最近几个月闲来无事，把js和css又重新过了一遍之后，又把目光又放到了vue3上面，之前几年都是用的react比较多，vue2只在大学毕业那一两年用过，vue3还是最近的工作中才有机会去使用的，同样，为了较为体系的去了解vue3，除了官方文档之外，还找了本《Vue.js设计与实现》来作为参考，这本书不厚，干货确实不少，对于vue了解不深的人来说，会有不小的收获。同时在学习的过程中，也会动手去自行实现，最终跑出了一个感觉还不错的vue框架示例。

<!-- more -->

## 前言

上篇文章更多的是概念上的东西，比较零散，从本篇开始我们就通过实际的具体实现来更加深入的了解vue本身。这次我们先从vue的响应式系统开始吧。

相关代码参考：[github：vue-design](https://github.com/nongcundeshifu/vue-design)

## 响应式系统

响应式系统的设计，其复杂点也在于需要针对不同的数据结构（例如Map和Set）和不同的行为（例如for in迭代）来处理不同场景下的符合预期的依赖收集和响应式数据触发，自动完成响应式依赖和正确的触发副作用函数，没有那么简单。想要基于 Proxy 实现一个相对完善的响应系统，也免不了去了解 ECMAScript 规范。

### 响应式数据与副作用函数

- 副作用函数指的是会产生副作用的函数（例如，effect函数中修改全局变量、或者修改`document.body.innerText = 'hello vue3'`这种
  - 当 effect 函数执行时，它会设置 body 的文本内容，但除了 effect 函数之外的任何函数都可以读取或设置 body 的文本内容。也就是说，`effect 函数的执行会直接或间接影响其他函数的执行`，这时我们说 effect 函数产生了副作用。
- 理解了什么是副作用函数，再来说说什么是响应式数据。假设在一个副作用函数中读取了某个对象的属性（例如obj.text）。
  - 此时：`当这个对象的属性变化时，我们期望能够重新执行副作用函数`。
  - 即`依赖了这个数据去执行一些副作用的函数，在这个数据更新时，我们期望这个函数能够以最新的数据去重新执行，以便这个副作用函数能够以最新的数据去重新处理它的副作用。`

### 响应式系统的基本实现

接着上文思考，如何才能让 obj 变成响应式数据呢？通过观察我们能发现两点线索：

- 当副作用函数 effect 执行时，会触发字段 obj.text 的读取操作；
- 当修改 obj.text 的值时，会触发字段 obj.text 的设置操作。

如果我们能拦截一个对象的读取和设置操作，事情就变得简单了，`当读取字段 obj.text 时，我们可以把副作用函数 effect 存储到一个“桶”里，当设置 obj.text 时，再把副作用函数 effect 从“桶”里取出并执行即可`

代码示例

```js

type EffectFn = (...data: any[]) => any

const bucket = new Set<EffectFn>()

const obj = {
    id: 1,
}

// 临时保存当前正在执行的副作用函数本身，方便让响应式对象和副作用函数进行关联
let currentEffect: EffectFn | null = null

const objProxy = new Proxy(obj, {
    get(target, key) {
        if (currentEffect && !bucket.has(currentEffect)) {
            bucket.add(currentEffect)
        }
        return target[key as keyof typeof target];
    },
    set(target, key, value) {
        target[key as keyof typeof target] = value
        bucket.forEach(fn => fn())
        return true
    }
})

// 通过currentEffect变量，来告诉响应式数据，当前执行的副作用函数是谁
function watchEffect(fn: (...data: any[]) => any) {
    currentEffect = fn
    fn()
    currentEffect = null
}

function effect() {
    const id = objProxy.id
    const idStr = 'effect idStr is: ' + objProxy.id // 多次执行
    console.log('effect id', id)
    console.log(idStr)
}

watchEffect(effect)

// 执行后输出
// effect id 1
// effect idStr is: 1
// objProxy.id = 13  // 更新后
// effect id 13
// effect idStr is: 13
```

上面是一个非常简单的响应式“系统”，当然真正的响应式系统，远没有那么简单，下面我们就去拆解其响应式系统中核心部分的细节内容。

## 依赖收集的实现

实现细节：

- [x] 参考示例responsive-1：最基础的功能，即当副作用函数 effect 执行时，会触发字段 obj.text 的读取操作，并和该对象建立联系。当修改 obj.text 的值时，会重新触发执行和其建立联系的副作用函数
  - [x] 并且，其代理对象的get方法中，并非是某个硬编码的副作用函数，而是支持任意的副作用函数：watchEffect和currentEffect的作用
- [x] 参考示例responsive-2：在示例responsive-1中，obj.text更新时会正确触发effect函数，但是如果我更新了obj.name时，effect仍然会执行，这并不符合预期，因为effect中并没有关联obj.name属性，需要优化
  - [x] 再次优化桶，使其存储的是任意数量代理对象的副作用函数的桶
    - [x] 使用weakMap来建立原始对象和其响应式副作用函数之间的关联，并且简化了副作用函数的内存管理
  - [x] 优化代理对象的创建，支持动态创建和包装原始对象的代理对象
  - [x] 代码优化：将响应式对象的依赖收集部分（get中的），封装到track函数中，track 是为了表达追踪的含义，将触发副作用函数的重新执行，放到trigger 函数中。
- [x] 参考示例responsive-3：副作用函数中分支切换时的依赖收集和清理，例如副作用函数中有`const value = obj.ok ? obj.name : 'default'`语句，如果 obj.ok为true时，该副作用函数会和obj.ok和obj.name进行关联，但是如果obj.ok切换为false时，我们不应该再将obj.name和副作用函数进行关联（三元表达式短路操作，此时obj.name不会被读取），应该需要清理。虽然不影响结果，但是会产生不必要的更新
- [x] 参考示例responsive-4：嵌套的副作用函数实现，在上诉实现中，我们使用了一个变量来存储当前的副作用函数以便响应式数据关联，而副作用函数可能嵌套。（注意，vue3的watchEffect嵌套会导致副作用函数被多次被收集）
  - [x] 即使副作用函数嵌套，例如effect1嵌套了一个函数fn，里面使用了响应式数据，那么fn本身是否应该视为一个单独的副作用函数进行处理？我觉得直接以最外层的副作用函数作为响应式数据的关联对象即可（默认支持）。因为除非用户显式指定，否则你无法把副作用函数里面创建的另外一个函数也识别为副作用函数。
  - [x] 如果显式嵌套了（例如使用watchEffect）那么需要实现和兼容（好像有点麻烦，自己实现起来貌似总是差点意思）。最终除了使用一个副作用函数栈来代替变量副作用函数变量外，还在副作用函数中使用next来缓存第一次watchEffect时的副作用函数，以便能正常清理deps依赖
    - [x] 然而这里仍然存在问题就是如果一个副作用函数中两个同级的watchEffect调用，那么只有一个next那么是无法缓存另外一个的，就导致第二个同级的watchEffect会一直调用第一个同级的watchEffect副作用函数。这在组件中的嵌套渲染函数副作用中是致命问题。
  - [x] 而vue3中，组件的更新也是基于effect，不过他可以基于生命周期，来卸载watchEffect再重建？也许是个思路，因为vue的watchEffect明确在非组件中需要自己卸载的。
  - [x] 可以参考responsive-object-6.ts实现的嵌套副作用函数，可以手动进行停止。这种嵌套的问题是，每次更新都会重新调用watchEffect创建一个新的副作用函数，所以依赖无法清理导致被多次收集，所以这种情况只能用变量缓存或者基于onCleanup来停止掉旧的watchEffect。
- [x] 参考示例responsive-5：避免无限循环：在副作用函数中，修改响应式数据，例如`obj.id++`这种操作。在trigger中触发副作用函数执行更新时，判断需要触发的副作用函数是否已经存在当前正在执行的副作用函数栈中，如果已经存在，则在trigger中跳过执行。（vue3如果是两个watchEffect互相修改，则会出现栈溢出）
- [x] 参考示例responsive-6：可调度执行，可调度性是响应系统非常重要的特性。首先我们需要明确什么是可调度性。所谓可调度，指的是当 trigger 动作触发副作用函数重新执行时，有能力决定副作用函数执行的时机、次数以及方式。基于一个任务队列，当需要执行时，将副作用函数添加到任务队列中，然后通过微任务的方式执行队列中的副作用函数
- [x] 参考示例responsive-7：计算属性 computed 与 lazy懒执行
  - [x] 懒执行effect副作用函数，现在的watchEffect是立即执行其传递的函数的，如果想要实现一个方法，使其在需要的时候再执行其函数呢？例如计算属性，这时候就需要考虑实现懒执行的副作用函数了。
- [x] 参考示例responsive-7：watch 的实现，所谓 watch，其本质就是观测一个响应式数据，当数据发生变化时通知并执行相应的回调函数。它和watchEffect的区别本质就是前者会基于显示指定的响应式数据来触发回调，而后者会自行收集回调函数中使用到的响应式数据来触发更新。
  - [x] 立即执行的watch回调和once实现
- [x] 参考示例responsive-8：过期的副作用，使用一个清除副作用的函数作为第三个参数传递给watch或者watchEffect，内部在执行新一轮副作用函数时先调用一次该清除函数。

注意：

- vue3的watchEffect嵌套，貌似会出现内部的watchEffect被重复收集，即多次执行，每次外层更新，内部就会被多收集一次依赖，所以vue3中，不要嵌套watchEffect等函数（或者自行清理）
- vue3的多个watchEffect中，如果互相修改响应式数据，那么会导致其无限循环，且出现警告。

依赖收集的最基本的实现基于于对响应式数据“读取”和“设置”操作的拦截，从而`在副作用函数与响应式数据之间建立联系`。当“读取”操作发生时，我们将当前执行的副作用函数存储到“桶”中，从而实现依赖的收集；当“设置”操作发生时，再将副作用函数从“桶”里取出并执行，从而实现副作用函数的重新执行。这就是响应系统的根本实现原理。

整个流程思路是不变的，那么我们就按照这个流程思路，一步步细化其中的细节。

### 依赖收集-bucket

我们使用一个bucket来存放副作用函数和响应式数据之间的关联关系，具体类似于：

```ts
/**
 * 存储响应式对象和副作用函数之间的关联关系
 * bucket结构：其中target是原始对象，key是其属性名，value是副作用函数集合（Set结构）
 * 简化的类型：bucket = WeakMap<target, Map<key, Set<EffectFn>>>
 * WeakMap {
 *   target1: Map {
 *     key1: Set [ effectFn1, effectFn2 ]
 *     key2: Set [ effectFn1, effectFn2 ]
 *   }
 *   target2: Map {
 *     key1: Set [ effectFn1, effectFn2 ]
 *     key2: Set [ effectFn1, effectFn2 ]
 *   }
 * }
 */
const bucket = new WeakMap<Record<string | symbol, any>, Map<string | symbol, Set<EffectFn>>>()
```

- 这里使用WeakMap来存放原始对象和其收集到的副作用函数，目的是为了在响应式数据被垃圾回收时，其副作用函数的依赖能够自动清理
- 其代表的含义为：某个对象的某个属性的副作用函数集合
- 其中key不仅仅可以是原始对象本身的属性key，也可以是由我们自行定义的一些特殊key，用于特殊场景下的依赖收集和依赖触发
  - 例如：用一个描述符为ITERATE_KEY的自定义symbol来代表在进行迭代操作时的依赖收集，由于迭代操作其含义是访问所有元素，而并非某个特定的key，所以这种场景下为对象的所有key都进行副作用函数的依赖收集本身也并不高效，且在对象新增删除属性时，难以再次识别并触发该副作用函数的重新执行。故为其特殊抽离一个迭代的symbol key来应对这种情况。

### track和trigger函数

在我们包装的代理对象中，我们会在get钩子中去对副作用函数进行依赖收集，在set中会触发副作用函数的重新执行，将其逻辑抽离出来，则分别对应于track和trigger函数。

track：依赖收集。`function track<T extends Record<string | symbol, any>>(target: T, key: string | symbol) => void`

- 在需要进行依赖收集的地方调用，例如代理的get方法中，在执行时，会基于当前正在运行的副作用函数，将其添加到bucket中去和响应式数据进行关联。
- 而当副作用函数执行时，其会被添加到一个activeEffectStack副作用函数栈中去，并在副作用函数执行完成后从栈中移除
- 而track则会基于模块全局的activeEffectStack副作用函数栈来获取当前正在执行的副作用函数

trigger： 触发响应式对象的副作用函数执行

```ts
type TriggerType = 'SET' | 'ADD' | 'DELETE'
// 触发响应式对象的副作用函数执行
function trigger<T extends Record<string | symbol, any>>(target: T, key: string | symbol, type: TriggerType, value?: any) {
  // code
}
```

- trigger的本质也是从bucket中取出和target、key相关联的副作用函数集合，并依次执行，它通常在代理对象的set的钩子中执行
- trigger中需要注意的是，其参数包含一个TriggerType，目的是为了区分该次触发更新是由于属性的新增、删除或者修改，它可以用来区分是否要触发ITERATE_KEY这种特殊的迭代相关副作用函数的重新执行。
  - 举个例子，我在副作用函数effectFn1中使用for...in遍历一个响应式对象obj时，并仅仅打印其key的值（其中没有再使用`obj[key]`来访问其属性值）很明显，这个副作用函数我仅仅是想打印这个响应式数据对象的key，那么理论上，当这个响应式对象的key新增或者删除时，我期望这个副作用函数会重新执行，而当响应式对象的某个属性值变化时，我不期望它重新执行。
  - 要实现上面这个需求，就需要配合上文提到的ITERATE_KEY这个特殊的依赖收集key了，我们会在代理对象的ownKeys钩子中进行依赖收集（其对应for...in等迭代操作所触发的代理对象钩子），且使用的正是ITERATE_KEY：`track(target, ITERATE_KEY);`。然后当该响应式对象在进行delete删除属性，或者set一个新属性时，其trigger方法会重新触发ITERATE_KEY相关的副作用函数的执行。从而实现了我们的需求。
- trigger内部还会对数组或者map等特殊数据进行特殊处理：例如当用户直接通过设置一个数组的响应式数据的length来减少数组的长度时，那么也等同于触发了那些索引大于当前数组length的元素的删除操作，也需要将他们的副作用函数重新执行。
  - 副作用函数effectFn1引用了arrProxy[10]，然后我直接设置arrProxy.length = 0，那么此时effectFn1需要被重新执行。

### 副作用函数的分支依赖和依赖清理

副作用函数存在分支切换时的依赖收集和清理，例如副作用函数中有`const value = obj.ok ? obj.name : 'default'`语句，如果 obj.ok为true时，该副作用函数会和obj.ok和obj.name进行关联，但是如果obj.ok更新为false时，在我们重新执行副作用函数时，我们不应该再将obj.name和副作用函数进行关联（三元表达式短路操作，此时obj.name不会被读取），应该需要清理。

那么该如何实现依赖的清理呢？如果你尝试使用依赖收集之间的对比，例如判断副作用函数重新执行时，所依赖的响应式数据和之前的差异，并依次来移除响应式对象和副作用函数的关联，那么实现起来会非常麻烦。所以这里用了一个更为简单粗暴的方式，即每次重新执行副作用函数之前，会将该副作用函数相关的响应式数据依赖全部都清除掉，然后在重新执行的过程中按照最新的依赖关系重建。

实现：

- 在副作用函数中，添加一个deps属性（`Set<Set<EffectFn>>`），它是一个Set集合，其中的值是bucket（`bucket = WeakMap<target, Map<key, Set<EffectFn>>>`）结构中的Set集合。它在track方法中进行添加。
- 在每次执行副作用函数之前，都会执行一个cleanup函数，其本质就是用来清空副作用函数关联的响应式数据。
- 这种方式极大的简化了响应式数据和副作用函数的依赖关系更新问题。

```ts
/**
 * 清除副作用函数中，所有关联的响应式数据对其的依赖
 */
function cleanup(fn: EffectFn) {

    // 清除副作用函数关联的响应式数据
    const deps: Set<Set<EffectFn>> = Reflect.get(fn, 'deps')
    deps.forEach(set => {
        set.delete(fn)
    })
    deps.clear()

}
```

### 嵌套的副作用函数

嵌套的副作用函数主要有两种情况，其一是直接嵌套两个watchEffect这种：

```ts
/**
 * 嵌套的响应式，并在更新其中某个属性时，仅触发对应的副作用函数执行
 */
watchEffect(function effectFn1() { // effectFn1
    console.log('objProxy id is', objProxy.id)
    // 嵌套的副作用函数
    watchEffect(function effectFn2() { // effectFn2
        console.log('objProxy name is', objProxy.name)
    })
})
```

其二则是vue3的组件渲染时，如果组件本身渲染了其他组件，其本质上也是一个嵌套的副作用函数执行。但是这种形式的嵌套是没有问题的，而第一种情况则会存在问题。

```ts
/**
 * vue3的组件渲染时的大致的嵌套副作用函数执行方式
 */
let effectFn2
watchEffect(function effectFn1() {
    console.log('objProxy id is', objProxy.id)
    if (effectFn2) {
      // 这里的effectFn2，等同于下面的那个watchEffect返回的副作用函数，而响应式更新时触发的副作用函数更新，就是返回的这个函数
      effectFn2()
    }
    else {
      effectFn2 = watchEffect(function effectFn2() {
          console.log('objProxy name is', objProxy.name)
      })
    }
})
```

- 首先为了支持嵌套副作用函数的执行，即一个副作用函数中会触发另外一个新的副作用函数的执行，我们使用了一个activeEffectStack副作用函数栈来存放多个当前正在执行的副作用函数，而副作用函数的执行要求同步，故在track时只需要取出栈顶的副作用函数进行依赖收集的关联即可。
- 在第一种情况的嵌套中，会存在什么问题？假设当effectFn1这个副作用函数被重新触发了，那么其内嵌套的watchEffect被重新执行，此时objProxy.name会又收集一个effectFn2函数，此时objProxy.name的收集的副作用函数中，就会有两个名为effectFn2的函数了
  - 但是上文我们提到，每次副作用函数重新执行时，会执行一个cleanup的清理函数，为何objProxy.name会收集两次effectFn2？
  - 这就是问题所在，这种嵌套的watchEffect，在其重新执行时，内部的watchEffect等同于创建了一个全新的副作用函数，既然是全新的副作用函数，自然cleanup没有任何作用了。也就是objProxy.name收集到的两次effectFn2不是同一个函数对象。
- 而在vue3中的组件渲染时，明明也是嵌套的副作用函数执行，为什么就不会有问题呢？
  - 这是因为vue3组件渲染中的副作用函数嵌套执行和第一种情况存在本质不同，当组件A渲染render函数时，其会放在一个watchEffect方法中，而此时，如果组件A嵌套了一个组件B：
    - 如果组件B未创建，则会走创建组件流程，此时走到setup函数中，会执行setup中可能存在的watchEffect，那么此时情况类似第一种情况的嵌套，但是，由于setup仅仅在组件生命周期中执行一次，所以即使后续组件更新，其setup中的watchEffect也不会被重新执行，故也不会存在问题。
    - 如果是组件B需要更新，那么此时的情况就和上述中的代码情形类似了，组件实例会保留在创建时，调用watchEffect返回的那个副作用函数，并且在组件更新时，直接调用该副作用函数，而不是再次调用watchEffect重新创建一次。

如何解决？正常情况下， 不要去嵌套watchEffect，因为这通常没有任何必要，如果真的嵌套了，你可以这样写（虽然感觉也不是很有必要）：

```ts
/**
 * 嵌套的响应式，并在更新其中某个属性时，仅触发对应的副作用函数执行
 */
let nestedEffectHandle1: any
function effectFn() {
    console.log('objProxy id is', objProxy.id)
    // 先停止内部的watchEffect函数监听
    nestedEffectHandle1?.stop()

    // 嵌套的副作用函数，重新创建
    nestedEffectHandle1 = watchEffect(() => {
        console.log('objProxy name is', objProxy.name)
    })
}
```

### 死循环问题

在使用watchEffect时，有如下代码：

```ts
/**
 * 在副作用函数中修改响应式数据时，处理无限递归调用
 */
function effect() {
    console.log('objProxy id is', objProxy.id)
    objProxy.id++
}

watchEffect(effect)
```

上面的代码问题在于，这个副作用函数在执行时会将objProxy.id作为依赖进行收集，同时在执行时会修改objProxy.id从而又触发该副作用函数的执行，如果没有任何处理，那么会出现死循环或者栈溢出（看触发副作用函数的执行时机是异步还是同步的）。

解决方法很简单，在触发trigger函数中，判断当前的activeEffectStack中是否存在即将要触发更新的副作用函数，如果存在则跳过。但是，在vue3如果是两个watchEffect互相修改，则会出现栈溢出，例如：

```ts
const objProxy = reactive({
    id: 1,
    age: 32,
})

/**
 * 两个副作用函数，互相修改对方依赖的数据
 */
function effect() {
    console.log('objProxy id is', objProxy.id)
    objProxy.age++
}

function effect2() {
    console.log('objProxy age is', objProxy.age)
    objProxy.id++
}

/**
 * 这里在vue3中会出现最大递归错误
 */
watchEffect(effect)
watchEffect(effect2)
```

> 死循环问题是可以被处理的，基于任务队列Set集合的特性或者activeEffectStack栈，但是即使处理掉了死循环，也会让代码的执行顺序难以被简单的预测和理解，故不如直接通过异常来告知用户。

### 可调度执行

可调度性是响应系统非常重要的特性。首先我们需要明确什么是可调度性。所谓可调度，指的是当 trigger 动作触发副作用函数重新执行时，有能力决定副作用函数执行的时机、次数以及方式。

- 之前我们在trigger 动作触发副作用函数重新执行时，是直接找到所有需要更新的副作用函数，然后依次同步执行，不过我们完全可以将其执行时机交由用户来决定，其实现方式则是在watchEffect中通过一个scheduler参数来控制，其值是一个函数，它接收watchEffect创建的副作用函数作为参数，并决定是否执行该副作用函数。
- 那么我们在trigger函数中，就可以判断其副作用函数是否存在scheduler，如果存在则调用scheduler，将副作用函数的执行控制权交由其scheduler，否则还是同步执行。
- 如此，我们就可以基于一个任务队列，当需要执行时副作用函数时，将副作用函数添加到任务队列中，然后通过微任务的方式执行队列中的副作用函数。

```ts
/**
 * 副作用函数执行的任务队列
 * 在执行副作用函数的过程中时，如果中间新增了副作用函数，那么执行是否要等到下一轮微任务再执行，则需要看实现需求。
 * 实现思路：基于一个任务队列，当需要执行时，将副作用函数添加到任务队列中，然后通过微任务的方式执行队列中的副作用函数
 */
const queueTask = {
    queue: new Set<EffectFn>(),

    isFlushing: false,

    add(effectFn: EffectFn) {
        this.queue.add(effectFn)
        if (this.isFlushing) {
            return
        }
        this.isFlushing = true
        Promise.resolve().then(() => {
            this.flush()
        }).finally(() => {
            this.isFlushing = false
        })
    },
    /**
     * 这里可以保护异步死循环，在执行时添加同样的副作用任务不会生效
     */
    flush() {
        this.queue.forEach(effectFn => effectFn())
        this.queue.clear()
    },

}

watchEffect(() => {
  // code
}, {
  scheduler(effectFn) {
    queueTask.add(effectFn)
  }
})
```

> 这在后续的计算属性和watch实现中都有应用

### 计算属性computed和lazy执行

懒执行watchEffect的副作用函数，现在的watchEffect是立即执行其传递的函数的，如果想要实现一个方法，使其在需要的时候再执行其函数呢？例如计算属性，这时候就需要考虑实现懒执行的副作用函数了。

- 默认对于watchEffect(effectFn)来说，effectFn是同步立即执行的，而我们知道，`计算属性const res = computed(fn)`中的fn是在res被访问时才会计算并执行，那么如果要实现计算属性，那么就需要阻止watchEffect来同步立即执行effectFn函数，我们通过添加lazy参数来控制这一行为。
- 同时在副作用函数由于响应式数据更新被重新执行时，我们使用新增的scheduler调度器参数来控制这一行为。

```ts
/**
 * 计算属性
 */
export function computed<T extends any>(getter: (oldValue: any) => T) {
    const origin: {
        value: T,
    } = {} as any
    Object.defineProperty(origin, '__is_computed', {
        enumerable: false,
        value: true,
    })
    let value: any
    let oldValue: any
    // 是否需要重新计算标识
    let dirty = true

    const { effectFn } = watchEffect(() => {
        return getter(oldValue)
    }, {
        lazy: true,
        scheduler() {
            dirty = true
            // 这里触发的也是origin的副作用函数执行
            trigger(origin, 'value', 'SET');
        }
    })

    return new Proxy(origin, {
        get(target, key) {
            if (key === 'value') {
                // 访问value时，如果需要计算新的值，则执行副作用函数
                if (dirty) {
                    oldValue = value
                    /**
                     * 等于嵌套effect，但是由于effectFn都是同一个，所以没有问题，这里进行的是getter里面的依赖收集
                     */
                    value = effectFn?.()
                    dirty = false
                }
                /**
                 * 这里收集的是origin的value
                 */
                track(target, key);
                return value
            }
            return undefined
        },
    })
}

const res = computed(() => {
  return 'name is ' + objProxy.name
})
```

- 在computed中，我们仍然是利用watchEffect来实现其依赖收集，不过我们会在内部创建一个带有value的对象，computed返回的是对其的代理对象
- 这里的重点在于其内部的watchEffect调用，它利用lazy控制watchEffect函数在第一次时不会执行传递给它的副作用函数，且通过scheduler，在其更新时它仅仅设置dirty标识，表示其计算属性需要重新被执行。并且使用trigger触发origin这个响应式对象的副作用函数依赖重新执行。
- 然后在origin的代理对象中，对其添加get钩子，在其中会判断dirty来决定是否需要重新执行副作用函数，同时使用track进行依赖收集。
- 至此我们就完成了可以懒执行的计算属性，同时我们也可以提供代理对象的set钩子，使其可以作为一个可写的计算属性。

> 这里的行为不一定和vue3的计算属性的行为一致，不过其大致思路是如此，可以根据需求自行控制其计算属性的行为，例如可以在scheduler中直接计算出最新的computed值，然后对比新值和旧值来决定是否触发trigger

### watch实现

所谓 watch，其本质就是观测一个响应式数据，当数据发生变化时通知并执行相应的回调函数。它和watchEffect的区别本质就是前者会基于显示指定的响应式数据来触发回调（不会收集回调中的响应式数据作为依赖），而后者会自行收集回调函数中使用到的响应式数据来触发更新。

- 同时支持immediate来立即触发回调的执行
- 支持once来设置仅执行一次回调

```ts
/**
 * watch函数，观察一个响应式数据，当其变化时执行回调
 */
export function watch(source: any, fn: EffectFn, options?: {
    flush?: FlushType,
    immediate?: boolean
    once?: boolean,
}) {

    let oldValue: any = undefined
    let newValue: any = undefined

    let getter = () => {
        return traverse(source)
    }

    if (typeof source === 'function') {
        getter = source as any
    }

    options = options || {}
    Reflect.set(options, 'lazy', true)

    /**
     * 这里需要单独的一个清理函数，因为effect函数中设置的副作用和需要执行的副作用函数不是同一个（一个是getter，一个是fn）
     */
    let onCleanupCallback: (() => void) | null = null
    function onCleanup(cleanFn: () => void) {
        onCleanupCallback = cleanFn
    }

    // 控制是否执行过，用来判断once选项
    let isRun = false

    /**
     * 同步执行的副作用函数调度器
     */
    function syncScheduler() {
        if (options?.once && isRun) {
            return
        }
        isRun = true
        newValue = effectFn?.()
        // 清理上一次回调的副作用代码
        onCleanupCallback?.()
        fn(oldValue, newValue, onCleanup)
        oldValue = newValue
    }

    /**
     * 响应式系统重副作用函数的调度器函数，用来调度副作用函数的执行
     */
    function queueScheduler() {
        queueTask.add(syncScheduler)
    }

    if (options.flush === 'sync') {
        Reflect.set(options, 'scheduler', syncScheduler)
    }
    else {
        Reflect.set(options, 'scheduler', queueScheduler)
    }

    // effect函数类似watchEffect函数，不过更加底层
    const { effectFn, stop } = effect(getter, options)

    if (options.immediate) {
        syncScheduler()
    }
    else {
        oldValue = effectFn();
    }

    return {
        stop,
    }

}
```

- watch的核心在于，我们通过一个traverse方法来递归访问watch方法的第一个参数source对象，并将其作为副作用函数（如果用户提供的是一个非函数的话，本质上是收集source作为响应式依赖），这样当该source响应式数据更新时，就会触发副作用函数的执行
- 然后再利用scheduler调度器参数来控制副作用函数的执行，即调用watch方法的第二个参数：回调函数

### 过期的副作用清理

所谓的过期的副作用，其实按照官网的例子，就是在watch中执行可能出现一些存在副作用的代码，例如fetch

```js
watch(id, (newId) => {
  fetch(`/api/${newId}`).then(() => {
    // 回调逻辑
  })
})
```

- 由于其watch中执行了一个异步fetch方法，其处理时机是不确定的，如果在其处理的过程中又再次触发了watch的更新，那么之前的fetch可能不再需要了，那如何清理呢？
- 通常我们会想到的就是取消上一个fetch请求或者丢弃掉上一个fetch请求的逻辑处理，而为了支持这一个功能，watch或者watchEffect方法的回调函数支持第三个参数onCleanup，来帮助我们做到这一点

```ts
watch(() => objProxy.id, async (oldValue, newValue, onCleanup) => {

    let expired = false
    onCleanup(() => {
        expired = true
        console.log('清理操作，取消本次异步任务')
    })

    await new Promise(resolve => setTimeout(resolve, newValue))

    if (expired) {
        return
    }
    objProxy.time = newValue
})
```

onCleanup的实现其实也很简单，在上一节的watch实现中，内部有一个onCleanupCallback变量，其存放的就是每次watch回调函数在执行时，其通过onCleanup方法传递的清理函数，在每次重新执行watch回调之前，都会调用一次onCleanupCallback这个方法来清理上一次watch回调函数产生的一些副作用代码。

## 非原始值的响应式方案

实现细节，处理不同对象访问形式的依赖收集，参考：responsive-object-1文件

- [x] 使用Proxy代理对象（非原始值）的响应式
- [x] 使用get收集属性读取时的响应式依赖
- [x] 使用ownKeys收集对对象进行迭代时的响应式依赖（例如for...in、Reflect.ownKeys()等）
  - [x] 由于对象迭代时的proxy钩子ownKeys没有key参数，所以使用一个symbol符号代替key来表示进行迭代时的副作用集合。
- [x] 使用set触发副作用函数的更新
  - [x] 判断是添加新的属性还是修改已有属性的值，用来触发迭代的副作用函数更新
- [x] 使用deleteProperty触发对象属性删除时迭代的副作用函数更新

浅响应与深响应，参考：responsive-object-2文件

- [x] 支持嵌套对象的响应式对象的创建，递归reactive即可
- [x] 在响应式对象中添加一个symbol符号用来判断是否是响应式对象
- [x] 用一个reactiveMap的weakMap来收集已经创建的响应式对象（原始对象为key，值为响应式对象），避免重复创建。
- [x] 实现浅响应式对象（默认本身就是浅响应式，只要不递归即可）

只读和浅只读，参考：responsive-object-3文件

- [x] 实现readonly()，我们希望一些数据是只读的，当用户尝试修改只读数据时，会收到一条警告信息。这样就实现了对数据的保护
- [x] 如果一个数据是只读的，那就意味着任何方式都无法修改它。因此，没有必要为只读数据建立响应联系。

代理数组，参考：responsive-object-4文件

- [x] 当我们通过索引读取或设置数组元素的值时：`arr[0] = 3;const value = arr[1]`，代理对象的 get/set 拦截函数也会执行，因此我们不需要做任何额外的工作，就能够让数组索引的读取和设置操作是响应式的了。
  - [x] 处理数组的length变更时的依赖触发（包括显示更新和隐式更新）
  - [x] for...in 循环遍历使用length作为key来收集依赖
  - [x] 处理数组查找的方法：includes、indexOf 和 lastIndexOf。支持原始值和响应式对象的查找
  - [x] 隐式修改数组长度的原型方法：push/pop/shift/unshift。解决方式：重写这些方法，在这些方法执行期间，不进行任何响应式依赖

代理map和set，参考：responsive-object-5文件

- [x] 由于map和set的特殊性质，则其需要自定义其所有属性和方法，才可以完成响应式依赖的收集和触发，例如size属性、forEach、keys、values方法等等。其创建的代理对象，只有一个get钩子。
- [x] 注意深层响应式的处理，需要将传递给用户的数据也变为响应式的。
- [x] 处理keys，其本身行为仅仅关心键，而非值，故其需要特殊处理
- [x] 处理迭代器和可迭代对象方法

实际上，实现响应式数据要比想象中难很多，并不是单纯地拦截 get/set 操作即可。举例来说，如何拦截 for...in 循环？track 函数如何追踪拦截到的 for...in 循环？类似的问题还有很多。除此之外，我们还应该考虑如何对数组进行代理。Vue.js 3 还支持集合类型等等

- Vue.js 3 的响应式数据是基于 Proxy 实现的，那么我们就有必要了解 Proxy 以及与之相关联的 Reflect。
- 什么是 Proxy 呢？简单地说，使用 Proxy 可以创建一个代理对象。`它能够实现对其他对象的代理，这里的关键词是其他对象`，也就是说，Proxy 只能代理对象，无法代理非对象值，例如字符串、布尔值等
- Proxy 只能够拦截对一个对象的基本操作（定义了13种）而像obj.fn()这种叫做复合操作，其本质上是两个对象中基本操作的结合：obj的读取，fn的apply调用
  - 理解 Proxy 只能够代理对象的基本语义很重要，后续我们讲解如何实现对数组或 Map、Set 等数据类型的代理时，都利用了 Proxy 的这个特点。
- 我们知道，vue2的响应式系统是基于defineProperty来定义属性描述符的get和set来实现的。那么为何vue3采用了proxy呢？其实大致可以从性能和功能两方面去考量。
  - 性能上，defineProperty 需要在一开始初始化时深度递归所有属性，而proxy可以做到初始化时只代理根对象，只有当读取嵌套对象属性（如 obj.a.b）时，才会在 get 拦截器中动态代理这个嵌套对象，做到懒代理，减少初始化时的性能开销
  - 功能上
    - defineProperty属性描述符仅能支持对象属性的get和set操作来实现依赖的收集，而proxy除了支持代理对象的get和set，还支持你对一个对象的所有操作的代理，例如for in遍历对象、删除对象、in操作符等
    - 原生数组、set、map等代理的支持
    - 支持对象属性的动态新增和减少时的依赖收集和副作用函数触发，无需用户额外关注

### Reflect中的方法的作用和Proxy

在代理中Reflect中的get和set方法还能接收第三个参数，即指定接收者 receiver，这在一些情况下，能解决直接`target[key]`的写法中的一些问题。参考如下代码

```js
/**
 * 创建响应式代理对象
 */
function createProxy<T extends Record<string | symbol, any>>(obj: T) {
    return new Proxy<T>(obj, {
        get(target, key, receiver) {
            track(target, key);
            // 使用其第三个参数，在原始对象的属性拥有get钩子时，将get钩子的this设置为代理对象。而如果直接使用target[key]则无法做到。
            return Reflect.get(target, key, receiver)
        },
        set(target, key, value) {
            // 用来处理麻烦的ts类型编写
            Reflect.set(target, key, value)
            trigger(target, key);
            return true
        }
    })
}
const obj = {
    id: 1,
    get idValue() {
      return this.id,
    }
}

const objProxy = createProxy(obj)

// 这里期望依赖收集期望是：它读取了idValue属性，那么在idValue更新时会触发副作用函数，但是idValue没有set钩子，不过idValue的get钩子中，其实是依赖的id属性，那么我期望在id更新时，能够触发该副作用函数
watchEffect(() => {
  // 读取响应式对象的数据
  objProxy.idValue
})
```

上面的代码中，使用Reflect.get方法的第三个参数，可以指定原始对象的get方法中的this指向代理对象objProxy，这样，在依赖收集时，由于读取了代理对象的id方法，就可以触发代理对象的依赖收集，从而实现我们想要的效果。

### 代理Object

响应系统应该拦截一切读取操作，以便当数据变化时能够正确地触发响应。下面列出了对一个普通对象的所有可能的读取操作：

- 访问属性：obj.foo：直接使用get拦截
- 判断对象或原型上是否存在给定的 key：key in obj：根据规范，in 操作符的运算结果是通过调用一个叫作 HasProperty 的抽象方法得到的。所以，使用has 拦截函数来实现
- 使用 for...in 循环遍历对象：`for (const key in obj){}`
  - for...in 的核心是「获取属性名列表」→ 触发 ownKeys，故代理中使用ownKeys钩子。注意：由于如果只是for...in循环，那么默认是为了获取对象的key，只要key没有更新（新增、删除）那么理论上不需要触发副作用函数
  - 注意，不使用getOwnPropertyDescriptor，它是「获取单个属性的详细信息」→ 不触发遍历逻辑，也许内部会使用它
  - for...in 的核心是「获取属性名列表」，而 getOwnPropertyDescriptor 是「获取单个属性的描述符」 只有当遍历到某个属性，且需要检查该属性是否「可枚举（enumerable）」时，才会间接调用 getOwnPropertyDescriptor，但这是遍历过程中的「辅助操作」，而非「触发遍历的核心操作」

其他操作：

- 删除一个存在的属性，使用代理的deleteProperty钩子
  - 注意，delete操作符和Reflect.deleteProperty会触发代理的get钩子，其原因在于，虽然delete 不直接触发 get，但是在其过程中的 “前置检查” 会触发get。
  - 需要优化delete操作时的get获取

### 代理数组

- 当我们通过索引读取或设置数组元素的值时：`arr[0] = 3;const value = arr[1]`，代理对象的 get/set 拦截函数也会执行，因此我们不需要做任何额外的工作，就能够让数组索引的读取和设置操作是响应式的了。
- 但对数组的操作与对普通对象的操作仍然存在不同，下面总结了所有对数组元素或属性的“读取”操作
  - 通过索引访问数组元素值：arr[0]（和对象一致，无需特殊处理）
  - 访问数组的长度：arr.length。（也和对象一致，无需处理）
    - 但是需要注意数组本身的长度变化时，length会变化
  - 把数组作为对象，使用 for...in 循环遍历（和对象一致，无需处理）
  - 使用 for...of 迭代遍历数组。
    - 迭代获取到的时数组的value，所以其本质上是会触发每个key的get钩子，所以其value变化时，会重新触发for...of的副作用
  - 数组的原型方法，如 concat/join/every/some/find/findIndex/includes 等，以及其他所有不改变原数组的原型方法。
    - 这属于数组访问的情况（即这些迭代本身会访问每个数组元素，从而为每个元素添加依赖）
    - 不过会把方法也作为key进行收集，且其方法内部可能会调用一些奇奇怪怪的方法，从而也会被收集起来
    - 感觉数组的依赖收集，只修要收集数字字符串和length即可
- 除了通过数组索引修改数组元素值这种基本操作之外，数组本身还有很多会修改原数组的原型方法。调用这些方法也属于对数组的操作，有些方法的操作语义是“读取”，而有些方法的操作语义是“设置”。因此，当这些操作发生时，也应该正确地建立响应联系或触发响应。
- 规范中明确说明，如果设置的索引值大于数组当前的长度，那么要更新数组的 length 属性。`所以当通过索引设置元素值时，可能会隐式地修改 length 的属性值。因此在触发响应时，也应该触发与 length 属性相关联的副作用函数重新执行`
  - 因为貌似length的设置是内部对原数组对象的设置，因为其代理set钩子中的设置操作最终作用于原数组对象，不会再走一遍代理
- 无论是使用 for...of 循环，还是调用 values 等方法，它们都会读取数组的 Symbol.iterator 属性。该属性是一个 symbol 值，为了避免发生意外的错误，以及性能上的考虑，我们不应该在副作用函数与 Symbol.iterator 这类 symbol 值之间建立响应联系
- includes查找数组时的响应式处理：实现数组查找的原型方法时，支持对原型对象和代理对象的判断，即类似：`objProxy.includes(objProxy[0])`或者`objProxy.includes(obj[0])`
  - vue3给出的方案是在代理中，`返回自定义的includes函数方法`。其他的一些方法可能也是按照这个去实现的。
  - 除了 includes 方法之外，还需要做类似处理的数组方法有 indexOf 和 lastIndexOf，因为它们都属于根据给定的值返回查找结果的方法
- 隐式修改数组长度的原型方法：push/pop/shift/unshift
  - 这些方法的实现中，`会读取length方法，同时会修改length方法，这是问题的所在`，如果多个独立effect副作用函数中都有push方法，则可能会触发无限循环，导致栈溢出，符合修改的同时读取，且是不同的副作用修改和读取。
  - 解决方式：重写这些方法，在这些方法执行期间，不进行任何响应式依赖，从而只应用其更新操作，因为这些方法本身也被视为修改，只不过内部会进行读取。

### 代理Set和Map

- 使用 Proxy 代理集合类型的数据不同于代理普通对象，因为集合类型数据的操作与普通对象存在很大的不同。
- 例如Set的严格校验：给原生 Set 实例套上 Proxy 后调用 values() 抛出 incompatible receiver 错误，核心原因是：原生 Set 方法（如 values()）会严格校验调用时的 this 是否为「原始 Set 实例」，而 Proxy 代理对象会破坏这个校验逻辑。
  - ES6 原生的 Set.prototype.values() 等方法，内部会执行一个关键校验：检查 this 的「内部槽位（Internal Slot）」是否为 Set 实例特有的 `[[SetData]]`
  - 内部槽位（如 `[[SetData]]、[[MapData]]`）是 JavaScript 引擎为原生对象（Set/Map/ArrayBuffer 等）维护的私有数据结构，无法被 Proxy 代理，也无法通过普通方式访问 / 修改；
  - 当你调用 proxySet.values() 时，values() 方法的 this 指向的是 Proxy 代理对象，而非原始 Set 实例；
  - 引擎检查 Proxy 对象的内部槽位时，发现没有 `[[SetData]]`，就判定「receiver 不兼容」，抛出错误。
- ES6 原生内置对象（Set/Map/Date/RegExp 等）的方法，和普通对象的方法有本质区别
  - 普通对象方法：仅依赖 this 的属性 / 方法，Proxy 可以正常代理；
  - 内置对象方法：依赖「内部槽位」校验，而内部槽位是「不可代理、不可伪造」的 ——Proxy 代理对象无法拥有这些槽位，因此无法通过校验。
  - 这是 ECMAScript 规范的明确规定：Set.prototype 上的方法属于「固有方法（Intrinsic Method）」，其 this 必须是「Set 实例」（拥有 [[SetData]] 槽位），否则抛出 TypeError。
  - 内部槽位（如 `[[SetData]]、[[MapData]]`）是 JavaScript 引擎为原生对象（Set/Map/ArrayBuffer 等）维护的私有数据结构，无法被 Proxy 代理，也无法通过普通方式访问 / 修改
- Set中除了values方法外，size属性也是一个getter钩子属性，其本身也会校验`[[SetData]]`插槽

Set 和 Map 类型的数据有特定的属性和方法用来操作自身。这一点与普通对象不同。正是因为这些差异的存在，我们不能像代理普通对象那样代理 Set 和 Map 类型的数据。`但整体思路不变，即当读取操作发生时，应该调用 track 函数建立响应联系；当设置操作发生时，应该调用 trigger 函数触发响应`

实现

- 针对`[[SetData]]`插槽判断，在使用Reflect.get时，最后一个参数要使用target而非receiver
- 针对size，其本身不能修改，故对其进行访问时利用迭代的key（ITERATE_KEY）来进行依赖的追踪
- 针对set或者map的原型函数，实现自定义的方法，来进行依赖追踪和副作用函数的触发。类似对部分数组原型方法的修改，在代理中进行包装。
- add、delete、clear方法，需要触发副作用函数的更新
- 其他方法，例如has、forEach、values等迭代方法，则在包装函数中，对副作用函数进行依赖收集（同样使用ITERATE_KEY作为key）
  - 遍历操作只与键值对的数量有关，因此任何会修改 Map 对象键值对数量的操作都应该触发副作用函数重新执行
  - 但是需要注意，在调用原始forEach等迭代方法时，传递给函数回调的，需要是一个代理对象，而非原始对象
- map的处理
  - 对于map来说由于其和对象类似，则get方法需要基于获取的key来存储副作用函数，并通过set判断是更新还是添加一个新的kye来触发副作用函数
  - map其实和对象类似，如果是需要进行深层次的代理，那么map内部在数据返回时，也应该需要对其进行代理，在get方法中判断其是否是可代理对象，如果是的话，则递归调用返回一个响应式数据
- map或者set无法使用for in进行迭代，无法迭代出任何东西

### 避免原始数据污染

参考如下代码：

```js
const obj = new Map<any, any>([['name', 'obj']])

const objProxy = reactive(obj)

const p1 = reactive(new Map([['name', 'p1']]))

objProxy.set('p1', p1)

/**
 * 通过原始值的get时，访问到了一个响应式对象，此时副作用函数进行了响应式依赖收集，收集到的时p1
 */
watchEffect(() => {
    console.log('p1 size', obj.get('p1').size)
})

// 使用原始数据修改为p1添加键值对，自己实现时此时副作用重新执行，也就是问题所在。但是在vue3中副作用不会执行
setTimeout(() => {
    obj.get('p1').set('pId', 100)
}, 5000)
```

怎么说呢，上面的代码，其实在我看来，是符合预期的，因为objProxy是代理的一个map对象，你为一个map对象设置值的时候，你很难说，我是真的想要给他的设置为一个响应式对象，还是期望给他的值设置为一个原始对象，只能说，二选一。就好比，如果obj是一个普通对象，那么我要给他的值设置成一个响应式对象时，其本身的行为也很难说你期望的是设置为一个响应式对象，还是要设置为一个原始值。只能说，二选一时统一行为就好。

对于对象来说，vue3下面这种情况写法不同，其更新也不同

```js
// 其原始值本身的p1属性，就是一个响应式对象
const obj = {
  name: 'obj',
  p1: reactive({
    name: 'p1',
  })
}

const objProxy = reactive(obj)

/**
 * 通过原始值访问到了一个响应式对象，此时副作用函数进行了响应式依赖收集
 */
watchEffect(() => {

    console.log('p1 name', obj.p1.pId)

})

// 使用原始数据修改为p1添加属性时，此时副作用重新执行

function handle() {
  obj.p1.pId = 100
}


// ------------分割，下面的这种写法，不会更新--------------

const obj = {
  name: 'obj',
}

const objProxy = reactive(obj)

// 通过代理对象，去设置一个新的属性，其值为响应式对象
objProxy.p1 = reactive({
  name: 'p1',
})

/**
 * 通过原始值访问到了一个响应式对象，此时副作用函数不会对其进行响应式依赖收集
 */
watchEffect(() => {

    console.log('p1 name', obj.p1.pId) // 此时obj.p1是一个不普通对象，非响应式对象

})

// 使用原始数据修改为p1添加属性时，不会重新执行副作用

function handle() {
  obj.p1.pId = 100
}

```

注意，如果是下面这么写，在vue3中也可以更新

```js


const p1 = reactive(new Map([['name', 'p1']]))

const obj = new Map<any, any>([['name', 'obj'], ['p1', p1]])

const objProxy = reactive(obj)

/**
 * 通过原始值的get时，访问到了一个响应式对象，此时副作用函数进行了响应式依赖收集，收集到的时p1
 */
watchEffect(() => {
    console.log('p1 size', obj.get('p1').size)
})

// 使用原始数据修改为p1添加键值对，此时副作用在vue3中也会重新执行
setTimeout(() => {
    obj.get('p1').set('pId', 100)
}, 5000)
```

重点在于，vue3中所谓的原始数据obj是什么样子的，如果在通过reactive创建时，传递的obj对象中，其本身就存在一个属性是响应式数据属性，那么你通过原始obj对象去访问这个数据时，依然会进行依赖收集，这是合理的，但是如果你是通过使用响应式对象去添加一个值为响应式对象的属性，`即使obj存在该属性，且为一个响应式对象属性，那么为obj添加的属性值仍然是原始数据，而非响应式对象`，此时就无法进行依赖收集。

在 set 方法内，我们把 value 原样设置到了原始数据 target 上。如果 value 是响应式数据，就意味着设置到原始对象上的也是响应式数据，我们把响应式数据设置到原始数据上的行为称为数据污染。

解决方法（或者说设计选择）：`在使用响应数据添加值时，需要将其原始值添加上去，而非响应式代理对象。`除了 set 方法需要避免污染原始数据之外，Set 类型的 add 方法、普通对象的写值操作，还有为数组添加元素的方法等，都需要做类似的处理。

注意vue3中map的set方法的key的行为：map的set方法，其key在使用一个原始对象或者响应式数据对象时，其行为会基于原始对象本事那个key是原始对象或者响应式对象而改变。参考下面代码

```js
const p1Obj = {
    name: 'p1-1',
  }
const p1 = reactive(p1Obj)
const obj = new Map([['name', 'obj'], [p1, 'p1 value']]) // 这里p1Obj作为key，或者使用p1作为key
const objProxy = reactive(obj)

objProxy.set(p1Obj, 'p1 value update') // p1Obj作为key设置
objProxy.set(p1, 'p1 value update') // p1为key设置
```

你可以修改上面的代码来验证，从结果看，如果在设置一个值时，原始对象的key如果存在，则直接修改，即使它是一个响应式数据对象，如果不存在，则将其变为一个原始值作为key然后进行设置。注意，上面vue会给出一个警告：Reactive Map contains both the raw and reactive versions of the same object as keys, which can lead to inconsistencies. Avoid differentiating between the raw and reactive versions of an object and only use the reactive version if possible，即：响应式映射同时包含同一对象的原始版本和响应式版本作为键，这可能导致数据不一致。应避免区分对象的原始版本和响应式版本，并尽可能仅使用响应式版本。

### 合理地触发响应的副作用函数

- 当值没有更新时，不需要触发副作用函数
- 处理从原型上继承属性的情况
  - 注意，由于obj.hasOwnProperty调用不会触发has钩子，所以，无法对其进行响应式依赖收集

正常情况原型对象上有get钩子属性时的代码参考：

```js
const obj = {
    title: 'obj',
}

const parent = {
    id: 1,
    get name() {
        console.log('this is', this) // 此时其this为obj对象
        console.log('this === obj', this === obj) // true
        return 'name-' + this.id;
    }
}

Object.setPrototypeOf(obj, parent)

console.log('obj name is', obj.name)
// // 等价于如下代码
console.log('parent is', Reflect.get(obj, 'name', obj))
```

在代理中访问原型的get钩子属性

```js
const obj = {
    title: 'obj',
}

const parent = {
    id: 1,
    get name() {
        console.log('this is', this) // 其值为代理对象，即objProxy对象
        console.log('this === obj', this === obj) // false
        return 'name-' + this.id;
    }
}

Object.setPrototypeOf(obj, parent)

const objProxy = new Proxy(obj, {
    get(target, key, receiver) {
        // 该钩子会被触发2次，一次是获取name属性，且原型parent中会调用this.id，所以会触发objProxy.id，获取id属性，符合普通对象访问时的预期，this被指向了实际的属性访问者
        return Reflect.get(target, key, receiver);
    }
})

console.log('objProxy name is', objProxy.name)
```

从原型上继承属性的情况示例：

```js
// 最终obj对象它的原型是protoProxy这个代理对象，注意，不是protoProxy的原型
const obj: any = {
    id: 1,
    name: 'zhou',
    age: 32,
}

const proto = {
    title: 'proto',
}

const objProxy = reactive(obj)
const protoProxy = reactive(proto)

/**
 * 将objProxy的原型指向protoProxy
 * 该设置默认会将objProxy代理的原始对象obj的原型给设置为protoProxy对象
 */
Reflect.setPrototypeOf(objProxy, protoProxy)

watchEffect(() => {
    console.log('objProxy title is', objProxy.title)
})

// 会导致触发两次副作用函数（虽然去重了，但是其本质是都触发了objProxy和protoProxy对象的set）
objProxy.title = 'li'
```

- 上面的示例中，该依赖被收集了两次，objProxy和protoProxy都会收集该依赖，原因在于如果对象自身不存在该属性，那么会获取对象的原型，并调用原型的 `[[Get]]` 方法得到最终结果。此时依赖分别被收集了两次
- 同时，在为objProxy.title设置一个原本不存在的属性时，会触发其set钩子，并且在钩子中我们使用Reflect.set(target, key, newVal, receiver) 来完成默认的设置行为，即引擎会调用 obj 对象部署的 `[[Set]]` 内部方法
- 而其中`[[Set]]`方法的流程中：如果设置的属性不存在于对象上，那么会取得其原型，并调用原型的 `[[Set]]`方法，也就是 protoProxy 的 `[[Set]]`内部方法。在这里就是protoProxy代理的set钩子
- 而protoProxy中的set钩子，其同样调用了Reflect.set(target, key, newVal, receiver)来设置属性，此时又会触发副作用函数执行，但是由于receiver在这里的值是objProxy，所以，最终会在obj上添加title属性。
  - 这是receiver机制本身，为了确保赋值上下文的正确绑定，否则你的代理可能会出现和普通赋值不一样的行为，即：obj.title = xxx，其`本身的规范行为会为obj对象本身添加属性`，而Reflect.set(target, key, newVal, receiver)中receiver就是为了确保能够和这个行为一致。

解决方法：

- 上面的示例中，先确认，是否应该为protoProxy收集该副作用函数。上面中，修改protoProxy的title时，会更新副作用函数，这是否符合预期呢？如果objProxy不再使用了，但是该依赖仍然会被protoProxy给收集起来，这是否符合预期？
  - 如果不符合，则考虑从去除该依赖的角度来解决
  - 如果符合，则用其他方案，防止重复触发
- 防止重复触发
  - 利用set集合本身，在触发更新时可以自动去重，不过这依赖于异步副作用函数队列的执行
  - 在set中进行逻辑判断，在该场景下，由于触发了原型的set，且传递了receiver来保证上下文的一致，此时target和receiver所代理的原始对象不是同一个，以此可以进行区分。（receiver是原始对象的代理对象，在通常情况下，它代理的原始对象和target是同一个）

## 原始值的响应式代理方案(ref)

实现细节，参考：responsive-object-6文件

- [x] 实现ref
- [x] 区分ref和reactive
- [x] 处理响应式丢失问题
- [x] 自动脱ref
- [x] 实现toRef和toRefs
- [x] 实现proxyRefs函数，它包装一个对象，访问对象中的属性时，如果其值是ref，则返回value，如果不是则直接返回原始值。

JavaScript 中的 Proxy 无法提供对原始值的代理，因此想要将原始值变成响应式数据，就必须对其做一层包裹，也就是ref

- 对于这个问题，我们能够想到的唯一办法是，使用一个非原始值去“包裹”原始值，例如使用一个对象包裹原始值，但这样做会导致两个问题：
  - 用户为了创建一个响应式的原始值，不得不顺带创建一个包裹对象；
  - 包裹对象由用户定义，而这意味着不规范。用户可以随意命名，例如 wrapper.value、wrapper.val 都是可以的。
- 为了解决这两个问题，我们可以封装一个函数，将包裹对象的创建工作都封装到该函数中。也就是ref，同时其包装对象中存在一个value，这也就是为什么ref对象需要使用value来获取值
- 注意：ref是将传递的那个数据包装为一个响应式ref对象，其内部本身不是一个ref，而是一个reactive对象。或者你可以理解为：ref(obj)时，其内部创建一个wrapper对象，其对象的value属性值是obj，然后再将一整个wrapper对象使用reactive()方法创建响应式。所以，你可以将ref视为reactive对象，只不过其属性只有一个value，且多了一个用于标识是否是ref的__v_isRef标识（通常它仅仅作用于自动解包），除此之外，它和普通的reactive在功能上没有任何区别。
- 如果对象本身是一个ref，则返回原有ref，不会嵌套创建ref

```ts
const REF_FLAG = '__v_isRef'
/**
 * 创建一个ref
 */
function createRef<T extends any>(value: T, options?: {
    shallow?: boolean,
}): Ref<T> {

    /**
     * 如果已经是ref了，则直接返回
     */
    if (isRef(value)) {
        return value
    }

    const wrapper = {
        value: value,
    }

    /**
     * 定义一个不可枚举且不可写的属性，标识这是一个ref对象
     */
    Object.defineProperty(wrapper, REF_FLAG, {
        value: true,
    })

    return createReactive(wrapper, {
        shallow: options?.shallow,
    })
}
```

### 如何区分响应式数据是一个ref还是一个reactive对象

我们有必要区分一个响应式数据到底是不是 ref，因为这涉及下文讲解的自动脱 ref 能力。（ref解包）

我们使用 Object.defineProperty 为包裹对象 wrapper 定义了一个不可枚举且不可写的属性 __v_isRef，它的值为 true，代表这个对象是一个 ref，而非普通对象。这样我们就可以通过检查 __v_isRef 属性来判断一个数据是否是 ref 了。

### 解决响应丢失问题

ref 除了能够用于原始值的响应式方案之外，还能用来解决响应丢失问题。这也是反复提到的，编写vue的响应式代码时很容易搞混传值和传递响应式数据。例如结构或者展开运算符（...）时，reactive导致的响应式丢失，因为其本质上，它变成了一个值。

```js
const obj = {
    name: 'obj',
    id: 1,
}

const objProxy = reactive(obj)

const newObj = {
    ...objProxy,
}

// 一个hooks函数接收一个id，当id变化时，期望重新执行
function useIdName(id: any) {
    watchEffect(() => {
        console.log('useIdName id', id)
    })
}
// 传递的是一个值，不是一个响应式数据。且，我期望当objProxy.id变化时，能够重新执行useIdName中的watchEffect，但是，这里的id是一个值传递，不是响应式数据，所以无法触发更新
useIdName(objProxy.id);

/**
 * ref代理
 */
watchEffect(() => {

    console.log('newObj.id', newObj.id) // 不具有响应式依赖

})
```

上面的问题，在于newObj通过扩展运算符，其变成了一个普通对象，不具有响应式，且由于其属性都是原始值，故也无法进行响应式。另外一个问题则是useIdName方法仅需要传递一个id，不关心额外数据，那有没有办法当objProxy.id变化时，能够重新执行useIdName里面的副作用函数呢？

貌似用ref可以做到这一点，我们改成如下代码试试看：

```js
const obj = {
    name: 'obj',
    id: 1,
}

const objProxy = reactive(obj)

const newObj = {
    name: ref(objProxy.name),
    id: ref(objProxy.id),
}

// 一个hooks函数接收一个id，当id变化时，期望重新执行
function useIdName(id: Ref<number>) {
    watchEffect(() => {
        console.log('useIdName id', id.value)
    })
}
// 用ref包裹住id
useIdName(ref(objProxy.id));

/**
 * ref代理
 */
watchEffect(() => {
    console.log('newObj.id', newObj.id.value) // 很可惜，当objProxy.id更新时，其仍然不会被重新执行
})
```

凭感觉，貌似我将原始数据转换为一个ref，那么我应该就拥有了响应式了，很可惜，虽然这两个watchEffect都被响应式数据依赖收集了，但是，它们收集的响应式对象和objProxy不是同一个，而是它们自己创建的那个ref对象，即ref(objProxy.id)和ref(objProxy.name)。所以，当objProxy.id更新时，他们仍然不会触发依赖更新。此时，我们就需要使用到toRef和toRefs了。

- toRef(reactive, key)
  - 可以将值、refs 或 getters 规范化为 refs
  - 也可以基于响应式对象上的一个属性，创建一个对应的 ref。这样创建的 ref 与`其源属性`保持同步：改变源属性的值将更新 ref 的值，反之亦然。
  - `其本质的实现是返回一个带有value属性的wrapper对象（也是一个ref对象），其value属性时一个getter属性，它内部访问了reactive对象的key`
  - 这样，访问wrapper.value属性时，其间接的访问了`reactive[key]`，从而做到了将依赖和reactive响应式对象的属性进行关联。它仅仅是一个“访问代理”
  - 对返回的wrapper进行设置时，其本质上也是对源reactive响应式对象的属性进行设置
  - 注意：它返回的wrapper对象，其本身不是一个响应式对象，所以它本身不会进行依赖收集，这一点很重要，而是一个和源响应式对象reactive关联的一个普通对象，只不过其长的比较像（目前来说是这样的）
    - 注意toRef本身的定义，返回的ref和源属性同步，其同步的是那一个属性的值，重点是绑定属性
  - 而对他进行设置，本质上也是对源响应式对象进行设置
  - 一些边界需要考虑：如果源响应式对象本身是一个ref了，那么则直接返回源ref
- toRefs(reactive)，内部本质是封装toRef。


参考如下代码：

```js

/**
 * toRef
 */
/**
 * toRef:可以将值、refs 或 getters 规范化为 refs
 * 也可以基于响应式对象上的一个属性，创建一个对应的 ref。这样创建的 ref 与其源属性保持同步：改变源属性的值将更新 ref 的值，反之亦然。
 */
function toRef<T extends any>(obj: any, key: string): Ref<T> {
    const wrapper = {
        get value() {
            return obj[key]
        },
        /**
         * 可写，则实现双向更新
         */
        set value(value: T) {
            obj[key] = value
        },
    }
    Object.defineProperty(wrapper, '__v_isRef', {
        value: true,
    })
    return wrapper
}

/**
 * toRefs
 */
function toRefs<T>(obj: T): Record<keyof T, Ref<any>> {
    const ret: any = {}
    for (const key in obj) {
        ret[key] = toRef(obj, key)
    }

    Object.defineProperty(ret, '__v_isRef', {
        value: true,
    })
    return ret
}

const obj = {
    name: 'obj',
    id: 1,
}

const objProxy = reactive(obj)

const newObj = toRefs(objProxy)

// 一个hooks函数接收一个id，当id变化时，期望重新执行
function useIdName(id: Ref<number>) {
    watchEffect(() => {
        console.log('useIdName id', id.value)
    })
}
// 用toRef包裹
useIdName(toRef(objProxy, 'id'));

/**
 * ref代理
 */
watchEffect(() => {
    console.log('newObj.id', newObj.id.value) // objProxy更新时，其也可以正常更新了
})
```

### 自动脱ref

toRefs 函数的确解决了响应丢失问题，但同时也带来了新的问题。由于 toRefs 会把响应式数据（这里主要指reactive对象）的第一层属性值转换为 ref，因此必须通过 value 属性访问值。这其实增加了用户的心智负担。

所谓自动脱 ref，指的是属性的访问行为，即如果读取的属性是一个 ref，则直接将该 ref 对应的 value 属性值返回。其主要解决的是在某种情况下，读取一个ref时，自动返回ref.value的值。这里的某种情况我们可以找到一些场景，例如：

- 组件模板中使用ref时（因为其有编译阶段，可以为我们做到）
- 作为 reactive 对象的属性​
  - 一个 ref 会在作为响应式对象的属性被访问或修改时自动解包。换句话说，它的行为就像一个普通的属性
  - 只有当嵌套在一个深层响应式对象内时，才会发生 ref 解包

组件模板中使用的ref自动解包：

- 组件在使用`顶层ref变量`时，会进行解包。
- 参考proxyRefs函数，它包装一个对象为proxy，且实现了get钩子，当访问对象中的属性时，如果是ref，则返回value，如果不是则直接返回原始值。(同时实现了set钩子)
  - 在vue3中，整个setup函数返回的对象会将其使用这个proxyRefs方法进行包装，从而实现模板中的ref解包能力
  - 注意，他并不会递归调用，而是只代理一个对象的第一层属性
  - 这也解释了为什么只有顶层的ref会进行解包（当然，一般情况，我们也仅仅只会将ref作为顶层变量）

注意，只有你手动创建的原始对象中包含了ref，那么在转换为响应式对象时，才可能出现一个响应式对象中的属性是一个ref的情况。即：

```js
const count = ref(0)
// 这种情况下的写法，才会出现响应式对象属性是一个ref
const state = reactive({
  count
})

console.log(state.count) // 0

state.count = 1
console.log(count.value) // 1
```

注意，访问数组和原生集合（Map和Set）中的元素时，不会产生解包，原文：与 reactive 对象不同的是，当 ref 作为响应式数组或原生集合类型 (如 Map) 中的元素被访问时，它不会被解包：

```js
const books = reactive([ref('Vue 3 Guide')])
// 这里需要 .value
console.log(books[0].value)
```

原因：

- 数组在进行迭代时，如果对ref进行解包，会让副作用函数将该ref作为依赖进行收集，这可能会产生额外的非预期的依赖收集，且和普通非ref值数组的依赖收集产生不一致的行为。
- 普通数组的响应式对象，在迭代时，默认仅关注数组本身的变化（length、长度等），而如果是一个ref的响应式数组，如果解包，那么此时依赖收集除了数组本身变化之外，还会额外包含所有ref对象，从而产生一来收集不一致和额外性能开销。
- 经过测试，在赋值时也不会解包（需要`books[0].value = xxx`），如果解包了，那么你无法确认，用户是想要修改这个数组元素，还是修改ref的值。因为数组的索引具有动态性，它和属性不一样，我们很容易期望数组修改为不一样的值，对象对于这种的修改需求会弱一点，且也是为了一致性做考量

### 总结ref

因为 JavaScript 的 Proxy 无法提供对原始值的代理，所以我们需要使用一层对象作为包裹，间接实现原始值的响应式方案。由于“包裹对象”本质上与普通对象没有任何区别，因此为了区分 ref 与普通响应式对象，我们还为“包裹对象”定义了一个值为 true 的属性，即 __v_isRef，用它作为 ref 的标识

## 总结

一个完善的响应式系统其本身需要考虑和处理的情况非常多，从依赖收集，到常用的响应式API（watch、计算属性等），再到语法层面的数据访问（for in、in操作符、数组、Set和Map），最后还有原始数据的响应式代理方案和ref解包，其响应式系统从非常多的角度去考虑了可能出现的依赖收集情况，来帮助用户减轻依赖收集的心智负担。