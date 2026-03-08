---
title: vue.js设计与实现-编译器
date: 2026-03-06 17:37:11
tags: 
  - vue
categories:
  - vue
---

仍然是最近几个月闲来无事，把js和css又重新过了一遍之后，又把目光又放到了vue3上面，之前几年都是用的react比较多，vue2只在大学毕业那一两年用过，vue3还是最近的工作中才有机会去使用的，同样，为了较为体系的去了解vue3，除了官方文档之外，还找了本《Vue.js设计与实现》来作为参考，这本书不厚，干货确实不少，对于vue了解不深的人来说，会有不小的收获。同时在学习的过程中，也会动手去自行实现，最终跑出了一个感觉还不错的vue框架示例。

<!-- more -->

## 前言

上篇文章我们了解了vue3的渲染器和组件，这篇文章我们接着来看看其vue的编译器。

相关代码参考：[github：vue-design](https://github.com/nongcundeshifu/vue-design)

## 编译器

编译技术是一门庞大的学科，想要做到完善的说明很难，而且我也是第一次接触，很多地方也不是非常了解。但不同用途的编译器或编译技术的难度可能相差很大，对知识的掌握要求也会相差很多

如果你要实现诸如 C、JavaScript 这类通用用途语言（general purpose language），那么就需要掌握较多编译技术知识。例如，理解上下文无关文法，使用巴科斯范式（BNF），扩展巴科斯范式（EBNF）书写语法规则，完成语法推导，理解和消除左递归，递归下降算法，甚至类型系统方面的知识等

作为前端工程师，我们应用编译技术的场景通常是：表格、报表中的自定义公式计算器，设计一种领域特定语言（DSL）等。其中，实现公式计算器甚至只涉及编译前端技术，而领域特定语言根据其具体使用场景和目标平台的不同，难度会有所不同。`Vue.js 的模板和 JSX 都属于领域特定语言`，它们的实现难度属于中、低级别，只要掌握基本的编译技术理论即可实现这些功能

## 模板DSL的编译器

编译器其实只是一段程序，它用来将“一种语言 A”翻译成“另外一种语言 B”。其中，语言 A 通常叫作源代码（source code），语言 B 通常叫作目标代码（object code 或 target code）。编译器将源代码翻译为目标代码的过程叫作编译（compile）。完整的编译过程通常包含词法分析、语法分析、语义分析、中间代码生成、优化、目标代码生成等步骤

整个编译过程分为编译前端和编译后端。

- 编译前端包含词法分析、语法分析和语义分析，它通常与目标平台无关，仅负责分析源代码。
- 编译后端则通常与目标平台有关，编译后端涉及中间代码生成和优化以及目标代码生成。但是，编译后端并不一定会包含中间代码生成和优化这两个环节，这取决于具体的场景和实现。
- 中间代码生成和优化这两个环节有时也叫“中端”。

Vue.js 模板编译器的目标代码其实就是渲染函数render和h函数（这里说的仅仅是template中的）

- 详细而言，Vue.js 模板编译器会首先对模板进行词法分析和语法分析，得到模板 AST。
  - AST 是 abstract syntax tree 的首字母缩写，即抽象语法树。所谓模板 AST，其实就是用来描述模板的抽象语法树
- 接着，将模板 AST 转换（transform）成 JavaScript AST。
- 最后，根据 JavaScript AST 生成 JavaScript 代码，即渲染函数代码

## parser解析器-生成模板AST

> 注意：这一章仅介绍一下解析器中的一些概念和简单的粗略实现，下文中还会有一章去深入parser解析器并使用更合适的实现方式。

基础实现参考：compiler-1.ts

如下示例模板：

```js
const template = `
    <template>
      <div>
        <h1 v-if="ok">Vue Template</h1>
      </div>
    </template>
`

// 模板抽象语法树（模板AST）
const ast = {
    type: 'Root', //逻辑根节点
    children: [
        {
            type: 'Element', // 元素节点
            tag: 'div',
            children: [
                {
                    type: 'Element', // 元素节点
                    tag: 'h1',
                    // 属性，其中指令也放置在这里
                    props: [
                        {
                            type: 'Directive', // 指令节点
                            name: 'if', // 指令名称
                            exp: {
                                type: 'SimpleExpression', // 简单表达式节点
                                content: 'ok',
                            }
                        }
                    ],
                    children: [
                        {
                            type: 'Text', // 文本节点
                            content: 'Vue Template'
                        }
                    ]
                }
            ]
        }
    ]
}
```

可以看到，AST 其实就是一个具有层级结构的对象。模板 AST 具有与模板同构的嵌套结构。每一棵 AST 都有一个逻辑上的根节点，其类型为 Root。模板中真正的根节点则作为 Root 节点的 children 存在。观察上面的 AST，我们可以得出如下结论。

- 不同类型的节点是通过节点的 type 属性进行区分的。例如标签节点的 type 值为 'Element'。
- 标签节点的子节点存储在其 children 数组中。
- 标签节点的属性节点和指令节点会存储在 props 数组中。
- 不同类型的节点会使用不同的对象属性进行描述。例如指令节点拥有 name 属性，用来表达指令的名称，而表达式节点拥有 content 属性，用来描述表达式的内容。

### 词法分析和语法分析

我们可以通过`封装 parse 函数来完成对模板的词法分析和语法分析，得到模板 AST。`

这一步仅仅类似于将源码分词、然后再分析其模板语法，将其转换一个模板AST抽象语法树，同时`这里仅验证部分语法错误`，例如是否有正确的开闭合标签、指令语法是否正确，它并不验证不同语法之间是否正确，例如v-else之前是否有一个v-if，这是后续的语义分析中的验证。

解析器的核心工作是将源文本转换为抽象语法树（AST），分为两个不可分割的阶段：

- 词法分析（Lexical Analysis）：由有限自动机（FSM）/ 正则实现，将原始文本拆分为词法单元（Token）（`如num:123、op:+、id:name、punctuator:(`），无语法含义，仅做 “字符分组”；
- 语法分析（Syntactic Analysis）：由递归下降算法（主流）实现，基于BNF/EBNF 定义的语法规则，对 Token 流进行语法结构匹配，将无含义的 Token 组合为有层级、有含义的 AST，赋予 Token 语法关系（如 “1+2*3” 的 AST 要体现 “乘除优先于加减” 的语法规则）。
- 词法分析回答 “这是什么字符组合”，递归下降的语法分析回答 “这些字符组合符合什么语法规则，彼此是什么关系”。
- 递归下降算法处理的输入是Token 流（而非原始文本），输出是AST（而非 Token），这是理解它的前提。

> 在下文的`深入解析器-parser`章节会尝试使用递归下降算法来解析模板得到模板AST

有了模板 AST 后，我们就可以对其进行语义分析，并对模板 AST 进行转换了。

### 语义分析

举个例子：

- 检查 v-else 指令是否存在相符的 v-if 指令。
- 分析属性值是否是静态的，是否是常量等。
- 插槽是否会引用上层作用域的变量。

可以做一些优化分析、标记以及语义上的正确性。

### parser解析器-词法分析tokenize

解析器的入参是字符串模板，解析器会逐个读取字符串模板中的字符，并根据一定的规则将整个字符串切割为一个个 Token（词法单元）。这里的 Token 可以视作词法记号，后续我们将使用 Token 一词来代表词法记号进行讲解。

例如`<p>vue</p>`会拆分为3个token，开始p标签，vue字符串，结束p标签。如何拆分的？依据什么规则？这就不得不提到有限状态自动机。状态机就是不同状态下会处理不同的操作以及可以处理什么操作，有限说明状态的数量是有限的，而自动则表明该状态机是自动在不同的状态进行切换。

实际上，解析 HTML 并构造 Token 的过程是有规范可循的。在 WHATWG 发布的关于浏览器解析 HTML 的规范中，详细阐述了状态迁移

Vue.js 的模板作为一个 DSL，并非必须遵守该规范。但 Vue.js 的模板毕竟是类 HTML 的实现，因此，尽可能按照规范来做，不会有什么坏处。更重要的一点是，规范中已经定义了非常详细的状态迁移过程，这对于我们编写解析器非常有帮助。

例如上面的`<p>vue</p>`可以拆分为如下几个状态以及状态转移规则：

- 初始状态，代表未匹配任何其他已有的状态规则
  - `<`：匹配标签开始状态
  - 字符：匹配文本状态
- 标签开始状态：
  - 字符：匹配标签开始名称状态
  - `/`：匹配标签结束状态
- 标签开始名称状态：
  - `>`：标签开始名称的结束，转回到初始状态（添加token）
- 文本状态：
  - `<`：匹配标签开始名称状态，（添加token）
- 标签结束状态：
  - 字符：匹配标签结束名称状态
- 标签结束名称状态：
  - `>`：标签结束名称的结束，转回初始状态（添加token）

有了上面的状态和状态转移规则，则可以从初始状态开始，依次遍历整个字符串模板，并在遍历的字符中根据当前所处的状态和当前遍历到的字符，确定下一步操作，直到完整整个字符串模板的遍历，即完成了词法分析阶段token的解析。

```ts
/**
 * 对模板的标记化处理
 * 例如<p>vue</p>会被标记化为三个token：开始标签token、文本token、结束标签token
 */
function tokenize(source: string): Token[] {
    // 状态机的状态
    let currentState = State.initial

    // 缓存字符
    const chars = []

    // 存放生成的token
    const tokens: Token[] = []

    // 遍历模板字符串的每一个字符
    for (let i = 0; i < source.length; i++) {
        const char = source[i]!

        // 匹配状态机当前的状态
        switch (currentState) {
            case State.initial:
                if (char === '<') {
                    currentState = State.tagOpen
                } else if (isAlpha(char) || isDigit(char)) {
                    chars.push(char)
                    currentState = State.text
                }
                break
            case State.tagOpen:
                if (isAlpha(char) || isDigit(char)) {
                    chars.push(char)
                    currentState = State.tagOpenName
                } else if (char === '/') {
                    currentState = State.tagEnd
                }
                break
            case State.tagOpenName:
                if (isAlpha(char) || isDigit(char)) {
                    chars.push(char)
                } else if (char === '>') {
                    const tagName = chars.join('')
                    tokens.push({
                        type: 'tagOpen',
                        name: tagName,
                    })
                    chars.length = 0
                    currentState = State.initial
                }
                break
            case State.text:
                if (isAlpha(char) || isDigit(char)) {
                    chars.push(char)
                } else if (char === '<') {
                    const textContent = chars.join('')
                    tokens.push({
                        type: 'text',
                        content: textContent,
                    })
                    chars.length = 0
                    currentState = State.tagOpen
                }
                break
            case State.tagEnd:
                if (isAlpha(char) || isDigit(char)) {
                    chars.push(char)
                    currentState = State.tagEndName
                }
                break
            case State.tagEndName:
                if (isAlpha(char) || isDigit(char)) {
                    chars.push(char)
                } else if (char === '>') {
                    const tagName = chars.join('')
                    tokens.push({
                        type: 'tagEnd',
                        name: tagName,
                    })
                    chars.length = 0
                    currentState = State.initial
                }
                break
        }
    }
    return tokens
}
```

### parser解析器-构建AST

不同用途的编译器之间可能会存在非常大的差异。它们唯一的共同点是，都会将源代码转换成目标代码。但如果深入细节即可发现，不同编译器之间的实现思路甚至可能完全不同，其中就包括 AST 的构造方式

对于通用用途语言（GPL）来说，例如 JavaScript 这样的脚本语言，想要为其构造 AST，较常用的一种算法叫作递归下降算法，这里面需要解决 GPL 层面才会遇到的很多问题，例如最基本的运算符优先级问题。

> 对于vue这种模板语言来说，它并没有所谓的运算符优先级问题，DSL 与 GPL 的区别在于，GPL 是图灵完备的，所以通常我们可以用GPL来实现DSL（例如用js来实现vue的模板DSL）。

HTML 是一种标记语言，它的格式非常固定，标签元素之间天然嵌套，形成父子关系。因此，一棵用于描述 HTML 的 AST 将拥有与 HTML 标签非常相似的树型结构。AST 在结构上与模板是“同构”的，它们都具有树型结构。而为vue的模板构建AST，我们可以使用栈来进行构建。

- 我们需要使用程序根据模板解析后生成的 Token 构造出这样一棵 AST
- 根据 Token 列表构建 AST 的过程，其实就是对 Token 列表进行扫描的过程。从第一个 Token 开始，顺序地遍历整个 Token 列表，直到列表中的所有 Token 处理完毕。
- 而对于标签的父子关系，我们使用栈来进行维护。栈的先进后出特性，非常适合处理这种嵌套的逻辑处理（例如函数调用也是一个栈）
  - 这里考虑到的是token的生成方式，token在生成时是按照字符顺序依次处理并生成相应的token，对于一个嵌套结构来说，token的前半部分都是节点的开始标签，后半部分是节点的结束标签，而token在处理过程中，仅仅标识或者分析出其token是一个开始标签或者结束标签，而不会管它们是否嵌套或者闭合。这按照源码本身的结构进行解析，而源码本身又是一种嵌套结构，自然token的列表也可以视为一种嵌套结构，所以，基于栈来构建token对应的AST

```ts
/**
 * 构建模板AST树
 */
function templateAST(tokens: Token[]) {
    const ast: TemplateASTNode = {
        type: 'Root',
        children: [],
    }

    /**
     * 节点栈
     */
    const elementStack = [ast]

    for (let i = 0; i < tokens.length; i++) {
        const token = tokens[i]!
        switch (token.type) {
            case 'tagOpen':
                const elementNode: TemplateASTNode = {
                    type: 'Element',
                    tag: token.name,
                    children: [],
                }
                elementStack.push(elementNode)
                break
            case 'text':
                const textNode: TemplateASTNode = {
                    type: 'Text',
                    content: token.content,
                }
                const parentElement = elementStack[elementStack.length - 1]!
                parentElement.children!.push(textNode)
                break
            case 'tagEnd':
                const node = elementStack[elementStack.length - 1]!
                if (node.type === 'Element' && node.tag === token.name) {
                    elementStack.pop()
                    const parent = elementStack[elementStack.length - 1]!
                    parent.children!.push(node)
                }
                else {
                    // 匹配不到开始标签，说明模板有误
                    throw new Error(`标签不匹配: </${token.name}>`)
                }
                break
        }
    }
    return ast

}
```

通过遍历由tokenize解析出来的tokens，我们最终可以得到和本章开头示例模板中类似的模板AST数据结构。

## transform转换器 - 模板 AST 转换为 JavaScript AST

在语义分析的基础上，我们即可得到模板 AST。接着，我们还需要将模板 AST 转换为 JavaScript AST。因为 Vue.js 模板编译器的最终目标是生成渲染函数，而渲染函数本质上是 JavaScript 代码，所以我们需要将模板 AST 转换成用于描述渲染函数的 JavaScript AST。

我们可以封装 transform 函数来完成模板 AST 到 JavaScript AST 的转换工作

### AST 的转换与插件化架构

所谓 AST 的转换，指的是对 AST 进行一系列操作，将其转换为新的 AST 的过程。新的 AST 可以是原语言或原 DSL 的描述，也可以是其他语言或其他 DSL 的描述。例如，我们可以对模板 AST 进行操作，将其转换为 JavaScript AST。

我们通常使用深度优先来对AST进行遍历，然后再遍历的过程中，对相应的节点进行处理，不过这里的“处理”在我们节点类型越来越多时，这个里面的代码会变得越来越臃肿，可能难以维护，所以，我们可以封装这里的处理逻辑，以插件化的架构形式，将节点的操作封装到独立的转换函数中

- 将转换逻辑函数（数组）作为参数传递给转换函数transform，
  - 然后transform基于不同的节点类型，调用不同的转换逻辑
  - 或者所有节点都调用其转换逻辑函数，然后每个转换逻辑函数自己判断需要处理的节点类型，并进行相应的处理。
- 为此，我们可以构建一个转换函数的上下文context，它用来承载一些在整个转换逻辑过程中的一些信息，这对于转换逻辑函数的编写非常有用
  - 例如当前转换的节点是哪一个？当前转换的节点的父节点是谁？甚至当前节点是父节点的第几个子节点？等等。
  - 以此可以实现节点的删除和替换操作
- 在转换 AST 节点的过程中，往往需要根据其子节点的情况来决定如何对当前节点进行转换，这时候就需要设计一个能力，让转换函数能够处理节点一开始的逻辑和节点的子节点处理完毕时的逻辑。
  - 它类似于koa的洋葱模型，本质上转换函数的处理逻辑可以分为两部分，当遍历到该节点，调用转换函数对其进行处理（处理未递归子节点的部分），然后递归处理其子节点，当子节点全部处理完成后，再调用转换函数来处理已递归子节点的那部分的逻辑。
  - 实现方式，可以将转换函数的逻辑拆分为两部分，一部分是转换函数本身的逻辑，另外一个部分则是转换函数返回的一个新函数，这个新函数会在该次递归完子节点时进行调用。
  - 顺序问题：第一部分的转换函数按照转换函数数组本身的顺序调用，而第二部分则按照其倒序的顺序调用。使其符合洋葱模型的调用顺序（例如日志打印，其在进入时放在最外层在最开始调用，那么它也应该作为最外层在返回时作为最后调用）
    - 这在下文的"将模板 AST 转为 JavaScript AST"中就有用到，其为节点生成js AST时，需要子节点的js AST也生成完毕，这是比较符合直觉的实现。
  - 而且，我们在得到模板AST后，可以对模板AST中的一些内容进行预处理，例如解析Element节点的属性，对其进行统一化处理，例如v-xx指令、v-bind动态绑定、@xxx事件等。利用该插件化架构也可以很好的对其合理的拆分逻辑和统一维护

### 将模板 AST 转为 JavaScript AST

为什么要将模板 AST 转换为 JavaScript AST 呢？原因我们已经多次提到：我们需要将模板编译为渲染函数。而渲染函数是由 JavaScript 代码来描述的，因此，我们需要将模板 AST 转换为用于描述渲染函数的 JavaScript AST（注：这里用的是自定义的JavaScript AST格式，较为简陋）

例如：`div><p>Vue</p><p>Template</p></div>`，它转换的渲染函数如下：

```js
function render() {
  return h('div', [
    h('p', 'Vue'),
    h('p', 'Template')
  ])
}
```

与模板 AST 是模板的描述一样，`JavaScript AST 是 JavaScript 代码的描述`。所以，本质上我们需要设计一些数据结构来描述渲染函数的js代码

首先，我们观察上面这段渲染函数的代码。它是一个函数声明，为了简化问题，这里我们不考虑箭头函数、生成器函数、async 函数等情况。除了函数声明外，还有语句，例如return返回语句，还有可能会存在：表达式、字符串、数组字面量、对象字面量、函数调用等。

那么，根据以上的渲染函数调用信息，我们就可以设计一个基本的数据结构来描述相关js语法（这本质上，其实就是js AST的结构，倒也不用设计，去抄一下就好）当然，也可以自己设计器js AST的数据结构，并且配合后续自己创建generate生成器来生成代码。

大致定义了如下用于描述js AST的数据结构：

```ts
/**
 * 标识符
 */
type IdentifierNode = {
    type: 'Identifier',
    name: string,
}

/**
 * 返回语句
 */
type ReturnStatement = {
    type: 'ReturnStatement',
    return: any
}

/**
 * 函数声明的AST（不是调用）
 */
type FunctionDeclNode = {
    type: 'FunctionDecl'
    id: IdentifierNode
    // 应该是一个参数名称
    params: IdentifierNode[]
    body: any[]
    arrowFunction?: boolean
}


/**
 * 函数调用
 */
type CallExpression = {
    type: 'FunctionCall'
    callee: IdentifierNode
    arguments: any[]
}

/**
 * 三元表达式
 */
type ConditionalExpression = {
    type: 'ConditionalExpression'
    expression: any
    left: any
    right: any
}

/**
 * 文本字面量
 */
type StringLiteral = {
    type: 'StringLiteral'
    value: string
}

/**
 * 数组字面量
 */
type ArrayLiteral = {
    type: 'ArrayLiteral'
    elements: any[]
}

/**
 * 对象字面量
 */
type ObjectLiteral = {
    type: 'ObjectLiteral',
    properties: Record<string, any>
}

/**
 * 表达式
 */
type Expression = {
    type: 'Expression',
    value: string
}

/**
 * js AST
 */
type JsAST = FunctionDeclNode | ArrayLiteral | CallExpression | StringLiteral | IdentifierNode | ReturnStatement | Expression | ObjectLiteral | ConditionalExpression

```

在定义好了js AST结构后，此时我们就可以利用transform转换器的插件化架构，定义不同的转换函数来处理不同模板AST节点到js AST节点的转换逻辑了。例如：

- transformTextNodeToJsAST：转换模板Text节点的AST为js AST节点
- transformElementNodeToJsAST：转换模板标签节点的AST为js AST节点

而其中的转换规则，大致如下：

- 如果是注释节点，则会转换为一个调用_createCommentVNode函数的函数调用表达式js AST，对应`_createCommentVNode('xxx')`
- 如果是文本节点，则会转换为一个调用_createTextVNode函数的函数调用表达式js AST，对应`_createTextVNode('xxx')`
- 如果是标签节点，则最终会转换为一个调用h函数的函数调用表达式js AST，对应`h('xxx')`
  - 其函数调用表达式的参数会根据该标签节点的子节点、props属性来定义，可能是一个对象字面量、数组或者字符串等
- 如果是根节点，会变成一个函数声明的js AST，对应：`function render() { xxx }`

最终，经过转换后，会得到一整颗JS AST对象

```js
// 模板示例
const template = `<div>
    <h3 size="16 + 1" @click="clickHandle">thisH3 {{ (1 + 2).toString() }}</h3>
    <!-- this is comment -->
    <p>Vue</p>   
</div>`

// 转换后的js AST
const jsAST = {
    "type": "FunctionDecl",
    "id": {
        "type": "Identifier",
        "name": "render"
    },
    "params": [
        {
            "type": "Identifier",
            "name": "_ctx"
        },
        {
            "type": "Identifier",
            "name": "_cache"
        },
        {
            "type": "Identifier",
            "name": "$props"
        },
        {
            "type": "Identifier",
            "name": "$setup"
        }
    ],
    "body": [
        {
            "type": "ReturnStatement",
            "return": {
                "type": "FunctionCall",
                "callee": {
                    "type": "Identifier",
                    "name": "h"
                },
                "arguments": [
                    {
                        "type": "StringLiteral",
                        "value": "div"
                    },
                    {
                        "type": "ArrayLiteral",
                        "elements": [
                            {
                                "type": "FunctionCall",
                                "callee": {
                                    "type": "Identifier",
                                    "name": "h"
                                },
                                "arguments": [
                                    {
                                        "type": "StringLiteral",
                                        "value": "h3"
                                    },
                                    {
                                        "type": "ObjectLiteral",
                                        "properties": {
                                            "size": {
                                                "type": "StringLiteral",
                                                "value": "16 + 1"
                                            },
                                            "onClick": {
                                                "type": "Expression",
                                                "value": "_cache[0] || (_cache[0] = $props.clickHandle)"
                                            }
                                        }
                                    },
                                    {
                                        "type": "Expression",
                                        "value": "\"thisH3 \" + _toDisplayString( (1 + 2).toString() )"
                                    }
                                ]
                            },
                            {
                                "type": "FunctionCall",
                                "callee": {
                                    "type": "Identifier",
                                    "name": "_createCommentVNode"
                                },
                                "arguments": [
                                    {
                                        "type": "StringLiteral",
                                        "value": " this is comment "
                                    }
                                ]
                            },
                            {
                                "type": "FunctionCall",
                                "callee": {
                                    "type": "Identifier",
                                    "name": "_createTextVNode"
                                },
                                "arguments": [
                                    {
                                        "type": "StringLiteral",
                                        "value": " "
                                    }
                                ]
                            },
                            {
                                "type": "FunctionCall",
                                "callee": {
                                    "type": "Identifier",
                                    "name": "h"
                                },
                                "arguments": [
                                    {
                                        "type": "StringLiteral",
                                        "value": "p"
                                    },
                                    {
                                        "type": "StringLiteral",
                                        "value": "Vue"
                                    }
                                ]
                            }
                        ]
                    }
                ]
            }
        }
    ]
}
```

## generate生成器 - 生成渲染函数

有了 JavaScript AST 后，我们就可以根据它生成渲染函数字符串了，这一步可以通过封装 generate 函数来完成代码生成

`代码生成本质上是字符串拼接的艺术`

- 大致流程就是通过判断其js AST的类型，然后调用不同的代码生成函数来生成字符串并将其拼接起来，同时，如果遇到数组、函数参数之类的需要递归调用。
- 如果我们希望最终生成的代码具有较强的可读性，因此我们应该考虑生成代码的格式，例如缩进和换行等。这就需要我们扩展 context 对象，为其增加用来完成换行和缩进的工具函数（同时保持其代码生成的一些信息）

例如函数定义的代码生成处理函数：

```ts
// 生成类型为FunctionDecl函数定义的字符串
function generateFunctionDecl(jsAST: FunctionDeclNode, context: generateContext) {
  // 从上下文中获取相关的字符串拼接方法（其中push是添加字符，newLine换行，indent控制缩进，默认是2格，deIndent控制取消缩进）
  const { push, newLine, indent, deIndent } = context

  // 函数名称
  const funcName = jsAST.id.name
  push(`function ${funcName} (`)
  // 参数生成
  genNodeList(jsAST.params, context)
  push(') {')
  indent()
  // 函数体
  jsAST.body.forEach(statement => {
      newLine()
      // 递归处理函数中的语句
      genNode(statement, context)
  })
  deIndent()
  if (jsAST.body.length) {
      newLine()
  }
  push('}')

}
```

上一节中的模板示例，最终转换为js代码字符串为：

```js
// 模板示例
const template = `<div>
    <h3 size="16 + 1" @click="clickHandle">thisH3 {{ (1 + 2).toString() }}</h3>
    <!-- this is comment -->
    <p>Vue</p>   
</div>`

// 转换后的渲染函数字符串
function render (_ctx, _cache, $props, $setup) {
  return h('div', [
    h('h3', {
      size: '16 + 1',
      onClick: _cache[0] || (_cache[0] = $props.clickHandle)
    }, "thisH3 " + _toDisplayString( (1 + 2).toString() )),
    _createCommentVNode(' this is comment '),
    _createTextVNode(' '),
    h('p', 'Vue')
  ]);
}
```

## 深入解析器-parser

实现参考：compiler-2.ts

我们初步讨论了解析器（parser）的工作原理，知道了解析器本质上是一个状态机。但我们也曾提到，正则表达式其实也是一个状态机。因此在编写 parser 的时候，利用正则表达式能够让我们少写不少代码（这里不是将状态机替换成正则，本质上解析过程还是基于状态机，不过在匹配标签之类的内容时，用正则可以简化我们的代码）

### 文本模式对解析器的影响

一个完善的 HTML 解析器远比想象的要复杂。我们知道，浏览器会对 HTML 文本进行解析，那么它是如何做的呢？其实关于 HTML 文本的解析，是有规范可循的，即 WHATWG 关于 HTML 的解析规范，其中定义了完整的错误处理和状态机的状态迁移流程，还提及了一些特殊的状态，例如 DATA、CDATA、RCDATA、RAWTEXT 等

`文本模式指的是解析器在工作时所进入的一些特殊状态，在不同的特殊状态下，解析器对文本的解析行为会有所不同`。具体来说，当解析器遇到一些特殊标签时，会切换模式，从而影响其对文本的解析行为：

- DATA：解析器的初始模式则是 DATA 模式。
  - 在默认的 DATA 模式下，解析器在遇到字符 `<` 时，会切换到标签开始状态（tag open state）。
  - 换句话说，`在该模式下，解析器能够解析标签元素。`
  - 当解析器遇到字符 `&` 时，会切换到字符引用状态（character reference state），也称 HTML 字符实体状态。
    - HTML 实体是一段以连字号（&）开头、以分号（;）结尾的文本（字符串）。HTML 实体常用于显示保留字符（这些字符会被解析为 HTML 代码）和不可见的字符（如“不换行空格”）。例如：`&lt;表示为 <`，`&gt;表示为 >`
- RCDATA：`<title> 标签、<textarea> 标签，当解析器遇到这两个标签时，会切换到 RCDATA 模式；`
  - 在该模式下，当解析器遇到字符 < 时，不会再切换到标签开始状态，而会切换到 RCDATA less-than sign state 状态。
  - 在 RCDATA less-than sign state 状态下，如果解析器遇到字符 /，则直接切换到 RCDATA 的结束标签状态，即 RCDATA end tag open state；
  - 否则会将当前字符 < 作为普通字符处理，然后继续处理后面的字符。
  - 由此可知，`在 RCDATA 状态下，解析器不能识别标签元素。`
  - `不过在 RCDATA 模式下，解析器仍然支持 HTML 实体`
- RAWTEXT：`<style>、<xmp>、<iframe>、<noembed>、<noframes>、<noscript> 等标签，当解析器遇到这些标签时，会切换到 RAWTEXT 模式；`
  - 解析器在 RAWTEXT 模式下的工作方式与在 RCDATA 模式下类似。唯一不同的是，`在 RAWTEXT 模式下，解析器将不再支持 HTML 实体。`
- CDATA：`当解析器遇到 <![CDATA[ 字符串时，会进入 CDATA 模式。`
  - CDATA 模式在 RAWTEXT 模式的基础上更进一步。`在 CDATA 模式下，解析器将把任何字符都作为普通字符处理，直到遇到 CDATA 的结束标志为止。`
  - MDN：CDATA 片段不应该在 HTML 中被使用，基本上在html里面见不到，它只在 XML 中有效。

对于 Vue.js 的模板 DSL 来说，模板中不允许出现 `<script>` 标签，因此 Vue.js 模板解析器在遇到 `<script>` 标签时也会切换到 RAWTEXT 模式。

### 递归下降算法构造模板 AST

之前我们首先对模板内容进行标记化得到一系列 Token，然后根据这些 Token 构建模板 AST。实际上，创建 Token 与构造模板 AST 的过程可以同时进行，`因为模板和模板 AST 具有同构的特性。`，同构是指模板结构和模板AST都是类似的嵌套结构。你可以想象成一个同一个配置文件用xml去配置和改成用json去配置时，他们本质上都可以互相转换来表达同一个意思。

之前我们在生成token的同时，是基于一个数组来存储解析到的token，其实我们在解析token的同时可以直接基于其token的类型和状态（例如标签开始或者结束）来利用栈直接处理其模板AST结构。例如解析到一个标签开始的token，我们直接可以基于该token创建一个标签AST，然后压入栈中并继续解析，然后再解析到一个其匹配的结束标签时，完成该标签AST的完整解析，将其移除栈。并且在其过程中解析到的子元素可以继续入栈，出栈。其本质和我们遍历token去解析是类似的流程。

所以这里和之前的实现差异：

- 这里词法分析和语法分析同时进行，基于核心的parseChildren函数
- 词法分析仍然基于有限状态自动机或者正则（正则本质也是一个状态机）
- 而语法分析阶段（转换为模板AST）则使用递归下降算法
- 同时，新增了文本模式，在不同模式下，决定能否处理某种状态
  - 例如RCDATA模式下，即使遇到`<`开头也不会进行标签解析。而是作为普通文本

我们使用parseChildren来解析我们的模板文本

- parseChildren 函数接收两个参数。
  - 第一个参数：上下文对象 context。
  - 第二个参数：由父代节点构成的栈，用于维护节点间的父子级关系。
- parseChildren 函数本质上也是一个状态机，该状态机有多少种状态取决于子节点的类型数量。在模板中，元素的子节点可以是以下几种。
  - 标签节点，例如 `<div>`。
  - 文本插值节点，例如 `{{ val }}`。
    - 注意，在vue模板中，如果文本插值总的括号之间带有空格，则不会被识别，例如：`{ {item}}`则会直接被作为字符串
  - 普通文本节点，例如：text。
  - 注释节点，例如 `<!---->`。
  - CDATA 节点，例如 `<![CDATA[ xxx ]]>`(基本上不会有)
  - 在标准的 HTML 中，节点的类型将会更多，例如 DOCTYPE 节点等。为了降低复杂度，我们仅考虑上述类型的节点。
- parseChildren 解析函数是整个状态机的核心，状态迁移操作都在该函数内完成。
  - `在具有嵌套的结构的标签模板中，每次解析标签都可能递归调用该parseChildren去解析子节点（此时父节点处于构建中的状态），从而会产生一个新的状态机，这就是“递归下降”中“递归”二字的含义。`
  - 而上级 parseChildren 函数的调用用于构造上级模板 AST 节点，被递归调用的下级 parseChildren 函数则用于构造下级模板 AST 节点。最终，会构造出一棵树型结构的模板 AST，这就是“递归下降”中“下降”二字的含义。类似于从一棵树的根节点开始构建，向下依次构建到树的叶子节点，并从叶子节点开始依次完成节点的构建，最终完成根节点的构建。
  - parseChildren内部等同于一个状态机，不停地处理字符，和tokenize的区别在于，他会递归调用自己并处理子节点为模板AST节点，而tokenize则是直接遍历模板字符并进行状态转换。其递归下降的核心在于：递归并构造
- 在解析模板时，我们不能忽略空白字符。这些空白字符包括：换行符（\n）、回车符（\r）、空格（' '）、制表符（\t）以及换页符（\f）。
- 状态机何时停止？以：`<div><p>Vue</p></div>`为例的话
  - 最开始调用parseChildren，启动一个状态机1，解析到div标签，此时调用parseElement处理，内部会递归调用parseChildren，此时开启状态机2，遇到p标签后再次调用parseElement，同样开启一个状态机3
  - 然后`在状态机3中解析文本，此时会遇到结束p标签，此时状态机3结束`
  - 然后再状态机2中也会遇到结束标签div，此时状态机2结束
  - 最后待解析字符串为空，第一个parseChildren结束
- 也就是说，状态机会在遇到结束标签（且匹配父栈的标签名时）以及待解析字符串为空时结束

注意：也许其模板规则比你想的要严格，例如空格在标签中不能随意加，例如这几种都是错误的模板格式：`</ h3>`、`< h3>`

```ts

/**
 * 文本模式
 */
const TextMode = {
    Data: 1,
    RCDATA: 2,
    RAWTEXT: 3,
    // CDATA: 4,
} as const

type TextMode = typeof TextMode[keyof typeof TextMode]

/**
 * 解析字符串模板，生成模板AST树
 * ancestors表示由父代节点构成的栈，用于维护节点间的父子级关系，这和解析tokens时的那个栈类似
 */
function parseChildren(context: ParseContext, ancestors: any[]): TemplateASTNode[] {
    const nodes: TemplateASTNode[] = []

    const { mode, advanceSpaces } = context
    // 用来防止无限循环的调试逻辑
    const whileCount = whileUnLimited()
    /**
     * 只要不结束就一直解析
     * 内部等同于一个状态机，不停地处理字符，和tokenize的区别在于，他会递归调用自己并处理子节点为模板AST节点
     */
    while (whileCount.count && !isEnd(context, ancestors)) {
        let node: TemplateASTNode | undefined
        /**
         * 只有在Data或RCDATA模式下，才能解析插值
         */
        if (mode === TextMode.Data || mode === TextMode.RCDATA) {
            /**
             * 只有在Data模式下才能解析标签和注释节点之类的
             */
            if (mode === TextMode.Data && context.source[0] === '<') {
                // 处理注释节点
                if (context.source[1] === '!' && context.source.startsWith('<!--')) {
                    node = parseComment(context)
                }
                else if (context.source[1] === '/') {
                    // 这里直接抛出异常了，因为理论上来说，如果正常的话遇到结束标签，应该是在isEnd函数中就已经处理掉了
                    throw new Error('无效的结束标签')
                }
                else if (/[a-z]/i.test(context.source[1] || '')) {
                    // 处理标签节点，内部会递归调用parseChildren来解析该标签的子节点
                    node = parseElement(context, ancestors)
                }
            }
            else if (context.source.startsWith('{{')) {
                // 处理插值节点
                node = parseInterpolation(context)
            }
        }

        if (!node) {
            // node不存在，则直接全部作为文本节点处理，外层isEnd作为循环结束判断
            node = parseText(context, ancestors)
        }
        nodes.push(node)
    }
    return nodes
}
```

### 解析标签节点

- 解析标签节点的核心是parseElement函数
- 首先，它会调用一个parseTag函数来解析标签名、解析属性
  - 同时它会判断其是否是闭合标签（例如`<img xxx />`这种）还是一个组件（大写字母开头按照约定会作为一个组件，会影响后续生成的渲染函数代码）
- 在解析完标签名、标签属性后，如果发现是一些特殊标签，则需要切换状态机的文本模式，例如textarea需要切换到TextMode.RCDATA模式，此时后续除非遇到textarea的结束标签字符，否则其余字符都会被视为字符串
- 同时，他会递归调用parseChildren来解析出其子节点。

### 解析属性

- 属性仅在解析标签节点的逻辑中进行调用（parseTag函数中调用）
- 属性只有`=`两边可以添加空白内容
- 属性的解析就是在属性名-等号-属性值这三个部分不断解析的过程。
  - 属性值的解析，可以使用这种表达式进行解析：`/^"([^"]*)"/i.exec(context.source) || /^'([^"]*)'/i.exec(context.source)`或者，解析第一个引号，然后找到剩余字符中下一个同样引号的位置，其中的内容，即为属性值
- 我们可以不在解析属性时，处理v-show、@等特殊指令和事件绑定的逻辑，而是先解析属性ast，然后交由语义分析阶段，或者在将模板AST转换为js AST阶段对这些特殊属性进行处理。
  - 例如：v-bind:xxx 解析出来的属性名就是v-bind:xxx，v-show解析出来的属性名就是v-show，然后在后续中我们可以基于属性名再进一步处理。

```ts
/**
 * 解析属性
 * 只有在开始标签才会进行属性解析
 * 这里仅解析属性，不处理指令、动态绑定等逻辑，交给后续阶段处理
 */
function parseAttributes(context: ParseContext) {
    const { advanceSpaces, advanceBy } = context

    const props: any[] = []
    const whileCount = whileUnLimited()

    /**
     * 不以 > 开头
     * 属性只有=两边可以添加空白内容
     * 优化其代码逻辑，属性解析可以看做不断消费这三种情况：[属性名] [? =] [? 属性值] 你可以将v-show、:xxx 也作为一个属性名来看待
     */
    while (whileCount.count && (context.source[0] !== '>' && !context.source.startsWith('/>')) ) {
        advanceSpaces()
        // 处理属性解析
        // 解析属性名称，同时兼容:data、@click这种写法。
        const match = /^[a-z@:][^\t\r\n\f/>= ]*/i.exec(context.source)
        if (!match) {
            throw new Error('动态属性解析失败，无法匹配属性名称')
        }
        const name = match[0]
        advanceBy(name.length)
        advanceSpaces()
        // 解析等号
        if (context.source[0] !== '=') {
            // 布尔属性
            props.push({
                type: 'Attribute',
                name,
                value: true,
            })
            continue
        }
        advanceBy(1)
        advanceSpaces()
        // 解析属性值
        const match2 = /^"([^"]*)"/i.exec(context.source) || /^'([^"]*)'/i.exec(context.source)
        if (!match2) {
            throw new Error('动态属性解析失败，无法匹配属性值')
        }
        advanceBy(match2[0].length)
        advanceSpaces()
        props.push({
            type: 'Attribute',
            name,
            value: match2[1],
        })
    }

    return props
}
```

### 解析字符实体

- HTML 实体总是以字符 & 开头，以字符 ; 结尾。
- HTML 实体有两类，一类叫作命名字符引用（named character reference），也叫命名实体（named entity），顾名思义，这类实体具有特定的名称，例如上文中的 `&lt;`。WHATWG 规范中给出了全部的命名字符引用，有 2000 多个，可以通过命名字符引用表查询
- 还有一类字符引用没有特定的名称，只能用数字表示，这类实体叫作数字字符引用（numeric character reference）。与命名字符引用不同，数字字符引用以字符串 `&#` 开头，比命名字符引用的开头部分多出了字符 #，例如 `&#60;`。实际上，`&#60;` 对应的字符也是 <，换句话说，`&#60; 与 &lt;` 是等价的。数字字符引用既可以用十进制来表示，也可以使用十六进制来表示。例如，十进制数字 60 对应的十六进制值为 3c，因此实体 `&#60;` 也可以表示为 `&#x3c;`
- 理解了 HTML 实体后，我们再来讨论为什么 Vue.js 模板的解析器要对文本节点中的 HTML 实体进行解码。为了理解这个问题，我们需要先明白一个大前提：`在 Vue.js 模板中，文本节点所包含的 HTML 实体不会被浏览器解析。这是因为模板中的文本节点最终将通过如 el.textContent 等文本操作方法设置到页面，而通过 el.textContent 设置的文本内容是不会经过 HTML 实体解码的`
- html实体涉及的状态机会比较麻烦，主要是由于历史原因，其字符实体需要支持不添加分号和添加分号的情况，以及属性中字符引用问题。这里暂时了解一下即可吧。

### 处理文本插值

- `文本插值仅仅在html内容中使用`，属性中指令、动态绑定本身就是表达式，不需要用文本插值
- 文本插值中的表达式字符串，也会对其进行html实体解码的
- 文本插值的解析比较简单，`在parseChildren中识别到 {{ 字符时即进入文本插值的解析，并且在遇到 }} 时结束插值解析`，且中间遇到的所有内容都会被视为一个表达式语句（进行特殊标记，后续表达式的处理仍然会在模板AST转js AST时进行）。

## 编译优化

编译优化指的是编译器将模板编译为渲染函数的过程中，尽可能多地提取关键信息，并以此指导生成最优代码的过程。

具体来说，Vue.js 3 的编译器会充分分析模板，提取关键信息并将其附着到对应的虚拟节点上（同样会利用编译时来优化生成的render代码）。在运行时阶段，渲染器通过这些关键信息执行“快捷路径”，从而提升性能。

编译优化的策略与具体实现是由框架的设计思路所决定的，不同的框架具有不同的设计思路，因此编译优化的策略也不尽相同。但优化的方向基本一致，`即尽可能地区分动态内容和静态内容，并针对不同的内容采用不同的优化策略。`

### 动态节点收集与补丁标志

- 三种传统diff算法，无论哪一种 Diff 算法，当它在比对新旧两棵虚拟 DOM 树的时候，总是要按照虚拟 DOM 的层级结构“一层一层”地遍历。
- 虽然diff算法可以找出最少更新内容，高效的更新的dom，但是无论如何，传统diff算法都需要对dom树进行一层一层的比较，最终得出需要更新的内容，而对于一个很小的元素修改时，可能在其过程中，产生非常多的无意义的比对操作。
- 实际上模板的结构非常稳定。通过编译手段，我们可以分析出很多关键信息，例如哪些节点是静态的，哪些节点是动态的。结合这些关键信息，编译器可以直接生成原生 DOM 操作的代码，这样甚至能够抛掉虚拟 DOM，从而避免虚拟 DOM 带来的性能开销。
- 传统 Diff 算法无法利用编译时提取到的任何关键信息，这导致渲染器在运行时不可能去做相关的优化。而 Vue.js 3 的编译器会将编译时得到的关键信息“附着”在它生成的虚拟 DOM 上，这些信息会通过虚拟 DOM 传递给渲染器。最终，渲染器会根据这些关键信息执行“快捷路径”，从而提升运行时的性能。
  - 正常来说，只要这些关键信息的运行时处理不会超过虚拟dom本身的性能开销，那么都可以算作是优化

### Block 与 PatchFlags

PatchFlags（更新标志）

- 来描述标签的虚拟节点拥有一个额外的属性，即 `patchFlag`，它的值是一个数字。`只要虚拟节点存在该属性，我们就认为它是一个动态节点。`这里的 patchFlag 属性就是所谓的补丁标志。
- 我们可以把补丁标志理解为一系列数字标记，并根据数字值的不同赋予它不同的含义，示例如下。
  - 数字 1：代表节点有动态的 textContent（例如上面模板中的 p 标签）。
  - 数字 2：代表元素有动态的 class 绑定。
  - 数字 3：代表元素有动态的 style 绑定。
  - 数字 4：其他……。
  - 这里利用了位运算来组合patchFlag

Block（动态收集）

- 和PatchFlags不同的是，PatchFlags是描述其自身需要更新的内容：直接文本textContent、class、style、动态属性等，Block是描述子节点的动态性
  - 如果一个节点具有PatchFlags，那么说明它就是动态节点，如果其作为子节点时（组件中的），就需要添加到其block中的dynamicChildren属性
- 同时，我们就可以在虚拟节点的创建阶段，把它的动态子节点提取出来，并将其存储到该虚拟节点的 dynamicChildren 数组内
- 而与普通虚拟节点相比，它多出了一个额外的 dynamicChildren 属性。我们把带有该属性的虚拟节点称为“块”，即 Block。所以，一个 Block 本质上也是一个虚拟 DOM 节点，只不过它比普通的虚拟节点多出来一个用来存储动态子节点的 dynamicChildren 属性。
- `一个 Block 不仅能够收集它的直接动态子节点，还能够收集所有动态子代节点。`如下示例，此时的header虚拟节点是一个block，其拥有dynamicChildren，且里面存放的是一个p标签虚拟节点，可能是因为这里如果不这么做，那么对于header节点来说，它如果不具有dynamicChildren，但是它的子节点具有dynamicChildren，那么在对比时，我就要先对比其header的子节点，然后发现子节点有dynamicChildren，那么再对比子节点的dynamicChildren，仍然不具备高效性，且如果不告知header拥有动态孙子节点，那么就会丢弃孙子节点的更新，仍然不符合预期。
  - 所以最好的方式，就是将子节点的dynamicChildren信息，直接放到header中去，然后div节点则不具有dynamicChildren。
  - 同时，这样的好处也在于可以将需要更新的动态虚拟节点进行结构化打平，减少需要递归的树层级，所以block对动态节点的比对操作是忽略 DOM 层级结构的。但是这也会带来一个问题：即 v-if、v-for 等结构化指令会影响 DOM 层级结构，使之不稳定。下文会提到。
- 有了 Block 这个概念之后，渲染器的更新操作将会以 Block 为维度。也就是说，当渲染器在更新一个 Block 时，会忽略虚拟节点的 children 数组，而是直接找到该虚拟节点的 dynamicChildren 数组，并只更新该数组中的动态节点。这样，在更新时就实现了跳过静态内容，只更新动态内容。
- 当我们在编写模板代码的时候，所有模板的根节点都会是一个 Block 节点
  - 实际上，除了模板中的根节点需要作为 Block 角色之外，带有结构化指令的节点，如任何带有 v-for、v-if/v-else-if/v-else 等指令的节点都需要作为 Block 节点

Block收集子孙节点的动态节点：

```html
<header>
  <div>
    <p>{{name}}</p>
  </div>
</header>
```

收集动态子节点

- 在编译器生成的渲染函数代码中，并不会直接包含用来描述虚拟节点的数据结构，而是包含着用来创建虚拟 DOM 节点的辅助函数
- 编译器在优化阶段提取的关键信息会影响最终生成的代码，具体体现在用于创建虚拟 DOM 节点的辅助函数上
- 基于编译器，在需要创建动态block的节点中使用类似`(openBlock, createBlock('div', [ xx ]))`来代替节点的创建，同时节点创建的顺序是从内向内执行的，利用栈结构来存储createVNode中创建的动态节点，那么在真正执行createBlock时，就可以收集到其子结构中的动态节点了。

### 结构化指令的节点作为block（嵌套block）

为什么何带有 v-for、v-if/v-else-if/v-else 等指令的节点都需要作为 Block 节点？参考如下示例：

```html
<div>
  <section v-if="show">
    <p>{{name}}</p>
  </section>
  <div v-else>
    <p>{{name}}</p>
  </div>
</div>
```

上面的示例中，如果只有根节点带有block，那么会产生如下的虚拟dom

```js
const vNode = {
  type: 'div',
  dynamicChildren: [
    // 这里由于之前提到的动态节点会跨层级收集
    {
      type: 'p',
      children: 'xxx',
    }
  ]
}
// new
const newVNode = {
  type: 'div',
  dynamicChildren: [
    // 这里由于之前提到的动态节点会跨层级收集
    {
      type: 'p',
      children: 'xxx',
    }
  ]
}
```

可以看到，当show更新时，其产生的虚拟dom会一样，导致无法进行更新。还有如下例子：

```html
<div>
  <section v-if="show">
    <p>{{name}}</p>
  </section>
  <section v-else>
    <div>
      <p>{{name}}</p>
    </div>
  </section>
</div>
```

上面的例子中，仍然是由于dynamicChildren会跨层级收集，如果不对齐进行处理会导致收集到的虚拟dom是一样的，从而无法进行更新。

那么根据上面的例子问题的根本原因在于：

- block需要在模板结构不变的情况下，其收集和更新才能保证正确。因为`dynamicChildren 数组中收集的动态节点是忽略虚拟 DOM 树层级的`。换句话说，结构化指令会导致更新前后模板的结构发生变化，即模板结构不稳定。
- 那么处理很简单，只需要在可能出现模板结构不稳定的结构化指令中将其包装为block块即可，这样将不同块中的动态节点收集到该块中
  - 同时，一个节点如果作为block，那么也需要添加到其上层的dynamicChildren属性中，以此就产生的嵌套的block（结构化指令也拥有动态性），那么在根节点的block进行对比时，会直接对不同的block块进行对比，这样就能够保证对于模板结构不稳定的节点，在更新时，其仍然拥有相对稳定的嵌套结构（保持其相对嵌套层级）
  - 并且，不同的block块其key是不同的，`例如在v-if、v-else的节点，主动为其添加key，以便在上层的block块对比时，由于不同的块拥有不同的key，就可以明确其整体的移除和创建。`
  - v-show不会破坏其结构化指令，所以不需要处理
- 对于for循环来说，其也是类似的，将整个循环包裹在一个虚拟的for循环节点中，类似Fragment，并将其视为一个块。，这样循环中的子节点列表会被收集到这个块中，从而保证其相对结构的稳定。
  - 但是注意，for循环的包裹虚拟节点本身无法保证其结构的稳定性的，因为其本身就是一个可能变更的结构指令，所以会对其进行特殊处理，即只要是for循环都会进行传统的diff。这里的block仅仅只是为了保证其循环所在的整体内容的稳定。
  - 不过循环中的每个子节点，都可以作为block来进行动态节点收集。

### 静态提升

静态的虚拟节点在更新时也会被重新创建一次。很显然，这是没有必要的，所以我们需要想办法避免由此带来的性能开销。而解决方案就是所谓的“静态提升”，即把纯静态的节点提升到渲染函数之外，避免每次执行render函数时都创建一个新的虚拟dom，即使它不需要更新。

- 需要强调的是，静态提升是以树为单位的。
- 除了根节点的 div 标签会作为 Block 角色而不可被提升之外，整个静态元素及其子代节点都会被提升。
  - 而如果在静态节点数中插入了一个动态节点，那么整颗节点树不会提升（以该动态节点向上查找的树）

除了静态虚拟节点的创建，还有静态属性，也可能会提升。

### 预处理字符串

基于静态提升，我们还可以进一步采用预字符串化的优化手段。预字符串化是基于静态提升的一种优化策略。静态提升的虚拟节点或虚拟节点树本身是静态的，那么，能否将其预字符串化呢

- 预字符串化能够将这些静态节点序列化为字符串，并生成一个 Static 类型的 VNode（如果组件中有一大片是静态节点，那么编译器会考虑将其作为一个Static 类型的 VNode，那么在其创建时，会直接通过innerHTML来插入到页面中。这样做有如下好处：
  - 大块的静态内容可以通过 innerHTML 进行设置，在性能上具有一定优势。
  - 减少创建虚拟节点产生的性能开销。
  - 减少内存占用。

### 缓存内联事件处理函数

提到优化，就不得不提对`内联事件`处理函数的缓存。缓存内联事件处理函数可以避免不必要的更新。

我们知道，vue的dom事件绑定支持下面两种，它们有什么区别？

- 事件内联事件处理器：事件被触发时执行的内联 JavaScript 语句 (与 onclick 类似)。
- 方法事件处理器：一个指向组件上定义的方法的属性名或是路径。

有一种说法是，内联事件处理器：每次组件更新（重新渲染）时，会生成新的匿名函数，触发虚拟 DOM 的 diff 算法检测到 “事件绑定函数变了”，进而更新 DOM 上的事件绑定。从props对比的逻辑流程中，这确实是有影响的，不过vue在编译阶段，会对其进行缓存优化：

- 对于内联语句或者内联函数，如果不为其创建缓存，那么每次其更新都是一个新的函数，那么在对比props时就会导致组件进行不必要的更新。
  - 所以，组件的render方法会接收一个_cache数组参数，并且在模板编译时，对事件绑定的函数进行缓存优化，类似于：`<div @click="() => xxx"></div>`会被编译为：`_createElementVNode("div", { onClick: _cache[0] || (_cache[0] = (...args) => xxx) })`
- 注意：仅针对事件，如果定义普通属性是一个函数，则没有该优化效果

### v-once手动缓存节点

Vue.js 3 不仅会缓存内联事件处理函数，配合 v-once 还可实现对虚拟 DOM 的缓存。Vue.js 2 也支持 v-once 指令，当编译器遇到 v-once 指令时，会利用我们上一节介绍的 cache 数组来缓存渲染函数的全部或者部分执行结果

```html
<div>
  <div v-once>{{ foo }}</div>
</div>
```

- 上面会变成类似：`cache[1] || (cache[1] = createVNode("div", null, ctx.foo, 1 /* TEXT */))`，后续更新导致渲染函数重新执行时，会优先读取缓存的内容，而不会重新创建虚拟节点。同时，由于虚拟节点被缓存，意味着更新前后的虚拟节点不会发生变化，因此也就不需要这些被缓存的虚拟节点参与 Diff 操作了（会暂停block节点的收集）
- v-once 指令通常用于不会发生改变的动态绑定中，例如绑定一个常量（或者一些基于配置生成的dom结构）

实际上，v-once 指令能够从两个方面提升性能。

- 避免组件更新时重新创建虚拟 DOM 带来的性能开销。因为虚拟 DOM 被缓存了，所以更新时无须重新创建。
- 避免无用的 Diff 开销。这是因为被 v-once 标记的虚拟 DOM 树不会被父级 Block 节点收集

## 服务端渲染（同构渲染）

Vue.js 可以用于构建客户端应用程序，组件的代码在浏览器中运行，并输出 DOM 元素。同时，Vue.js 还可以在 Node.js 环境中运行，它可以将同样的组件渲染为字符串并发送给浏览器。这实际上描述了 Vue.js 的两种渲染方式，即客户端渲染（client-side rendering，CSR），以及服务端渲染（server-side rendering，SSR）。

SSR 和 CSR 各有优缺点。

- SSR 对 SEO 更加友好，而 CSR 对 SEO 不太友好。
  - ssr需要执行js并请求数据后才能构建页面内容，而SEO爬虫通常不会执行js，从而使爬虫无法正确加载和解析页面内容，自然也无法对其页面内容进行SEO
  - 相对于csr来说，由于白屏问题存在，所以其页面加载速度（web性能指标）可能会影响SEO排名
- 由于 SSR 的内容到达时间更快，因此它不会产生白屏问题。相对地，CSR 会有白屏问题。
- 另外，由于 SSR 是在服务端完成页面渲染的，所以它需要消耗更多服务端资源。CSR 则能够减少对服务端资源的消耗。
- 对于用户体验，由于 CSR 不需要进行真正的“跳转”，用户会感觉更加“流畅”，所以 CSR 相比 SSR 具有更好的用户体验。

那么，我们能否融合 SSR 与 CSR 两者的优点于一身呢？答案是“可以的”，这就是接下来我们要讨论的同构渲染。

### 同构渲染

同构渲染的“同构”一词的含义是，同样一套代码既可以在服务端运行，也可以在客户端运行。例如，我们用 Vue.js 编写一个组件，该组件既可以在服务端运行，被渲染为 HTML 字符串；也可以在客户端运行，就像普通的 CSR 应用程序一样。

同构渲染分为首次渲染（即首次访问或刷新页面）以及非首次渲染。

- 实际上，同构渲染中的首次渲染与 SSR 的工作流程是一致的。也就是说，当首次访问或者刷新页面时，整个页面的内容是在服务端完成渲染的，浏览器最终得到的是渲染好的 HTML 页面。但是该页面是纯静态的，这意味着用户还不能与页面进行任何交互，因为整个应用程序的脚本还没有加载和执行。
- 除此之外，同构渲染所产生的 HTML 页面与 SSR 所产生的 HTML 页面有一点最大的不同，即前者会包含当前页面所需要的初始化数据。直白地说，服务器通过 API 请求的数据会被序列化为字符串，并拼接到静态的 HTML 字符串中，最后一并发送给浏览器。这么做实际上是为了后续的激活操作，后面会详细讲解。

假设浏览器已经接收到初次渲染的静态 HTML 页面，接下来浏览器会解析并渲染该页面。在解析过程中，浏览器会发现 HTML 代码中存在 `<link> 和 <script>` 标签，于是会从 CDN 或服务器获取相应的资源，这一步与 CSR 一致。当 JavaScript 资源加载完毕后，会进行激活操作，这里的激活就是我们在 Vue.js 中常说的 “hydration”（水合）。激活包含两部分工作内容。

- Vue.js 在当前页面已经渲染的 DOM 元素以及 Vue.js 组件所渲染的虚拟 DOM 之间建立联系。
- Vue.js 从 HTML 页面中提取由服务端序列化后发送过来的数据，用以初始化整个 Vue.js 应用程序。

激活完成后，整个应用程序已经完全被 Vue.js 接管为 CSR 应用程序了。后续操作都会按照 CSR 应用程序的流程来执行。当然，如果刷新页面，仍然会进行服务端渲染，然后再进行激活，如此往复。

同构渲染除了也需要部分服务端资源外，其他方面的表现都非常棒。由于同构渲染方案在首次渲染时和浏览器刷新时仍然需要服务端完成渲染工作，所以也需要部分服务端资源，但相比所有页面跳转都需要服务端完成渲染来说，同构渲染所占用的服务端资源相对少一些。

另外，对同构渲染最多的误解是，它能够提升可交互时间（TTI）。事实是同构渲染仍然需要像 CSR 那样等待 JavaScript 资源加载完成，并且客户端激活完成后，才能响应用户操作。因此，理论上同构渲染无法提升可交互时间。（只不过，相比于csr来说，由于数据和html一同返回，其部分页面的内容会相比csr来说更早出现在页面以及完成激活，所以这部分交互时间会更早，但是对于传统ssr来说，则并没有提升）

### 将虚拟 DOM 渲染为 HTML 字符串

为了将虚拟节点 ElementVNode 渲染为字符串，我们需要实现 renderElementVNode 函数。该函数接收用来描述普通标签的虚拟节点作为参数，并返回渲染后的 HTML 字符串。

- 这里同样利用了递归拼接字符串的过程
- 注意：这里的渲染成字符串，仅仅只是将虚拟dom转换为html的字符串，该html字符串返回给浏览器时，浏览器会生成相应的dom，而vue在水合时，仍然会基于组件渲染函数创建相应的虚拟dom，只不过vue在这个阶段会将虚拟dom和页面中实际的dom元素给关联起来。
- 注意：在处理属性值时，我们需要对其进行转义处理，这对于防御 XSS 攻击至关重要。HTML 转义指的是将特殊字符转换为对应的 HTML 实体。
- 在将其渲染为字符串时，要考虑以下内容。
  - 自闭合标签的处理。对于自闭合标签，无须为其渲染闭合标签部分，也无须处理其子节点。
  - 属性名称的合法性，以及属性值的转义。
  - 文本子节点的转义

### 将组件渲染为HTML字符串

- 我们需要实现 renderComponentVNode 函数，并用它把组件类型的虚拟节点渲染为 HTML 字符串
- 实际上，把组件渲染为 HTML 字符串与把普通标签节点渲染为 HTML 字符串并没有本质区别。我们知道，组件的渲染函数用来描述组件要渲染的内容，它的返回值是虚拟 DOM。所以，我们只需要执行组件的渲染函数取得对应的虚拟 DOM，再将该虚拟 DOM 渲染为 HTML 字符串，并作为 renderComponentVNode 函数的返回值即可
- 在进行服务端渲染时，组件的初始化流程与客户端渲染时组件的初始化流程基本一致，但有两个重要的区别。
  - `服务端渲染的是应用的当前快照，它不存在数据变更后重新渲染的情况。因此，所有数据在服务端都无须是响应式的。利用这一点，我们可以减少服务端渲染过程中创建响应式数据对象的开销。`
    - 所谓快照，指的是在当前数据状态下页面应该呈现的内容。
  - 服务端渲染只需要获取组件要渲染的 subTree 即可，无须调用渲染器完成真实 DOM 的创建。因此，在服务端渲染时，可以忽略“设置 render effect 完成渲染”这一步
    - 这意味着，组件的 beforeMount 以及 mounted 钩子不会被触发。而且，由于服务端渲染不存在数据变更后的重新渲染逻辑，所以 beforeUpdate 和 updated 钩子也不会在服务端执行。
  - 通常服务端渲染的组件创建流程不要和客户端的搞混了，因为其流程虽然类似，但是有许多差别，为了逻辑清晰还是不需要在同一个函数中处理，这样也利于TreeShaking

### 客户端激活的原理

对于同构渲染来说，组件的代码会在服务端和客户端分别执行一次。

- 在服务端，组件会被渲染为静态的 HTML 字符串，然后发送给浏览器，浏览器再把这段纯静态的 HTML 渲染出来。这意味着，此时页面中已经存在对应的 DOM 元素。
- 同时，该组件还会被打包到一个 JavaScript 文件中，并在客户端被下载到浏览器中解释并执行。

这时问题来了，当组件的代码在客户端执行时，会再次创建 DOM 元素吗？答案是“不会”。由于浏览器在渲染了由服务端发送过来的 HTML 字符串之后，页面中已经存在对应的 DOM 元素了，所以组件代码在客户端运行时，不需要再次创建相应的 DOM 元素。

- 但是，组件代码在客户端运行时，仍然需要做两件重要的事：
  - 在页面中的 DOM 元素与虚拟节点对象之间建立联系；
  - 为页面中的 DOM 元素添加事件绑定。
- 我们知道，一个虚拟节点被挂载之后，为了保证更新程序能正确运行，需要通过该虚拟节点的 vnode.el 属性存储对真实 DOM 对象的引用。而同构渲染也是一样，为了应用程序在后续更新过程中能够正确运行，我们需要在页面中已经存在的 DOM 对象与虚拟节点对象之间建立正确的联系。
- 另外，在服务端渲染的过程中，会忽略虚拟节点中与事件相关的 props。所以，当组件代码在客户端运行时，我们需要将这些事件正确地绑定到元素上。
- 其实，这两个步骤就体现了客户端激活的含义。

当组件进行纯客户端渲染时，我们通过渲染器的 renderer.render 函数来完成渲染，而对于同构应用，我们将使用独立的 renderer.hydrate 函数来完成激活。

- 具体实现之前，我们先确定一下页面中已经存在的真实 DOM 元素与虚拟 DOM 对象之间的关系。
- `真实 DOM 元素与虚拟 DOM 对象都是树型结构，并且节点之间存在一一对应的关系。因此，我们可以认为两棵树它们是“同构”的。`而激活的原理就是基于这一事实，递归地在真实 DOM 元素与虚拟 DOM 节点之间建立关系。
  - 注意，在虚拟 DOM 中并不存在与容器元素（或挂载点）对应的节点。因此，在激活的时候，应该从容器元素的第一个子节点开始
  - 还有就是对于一些没有真实元素的节点（例如Fragment这种），同样需要确定真实节点的位置，这里有个很巧妙的思路在于，在客户端中，Fragment或者for循环这种表达多个子节点的无真实dom的虚拟节点，其前后仍然具有一个没有内容的text节点作为定位锚点anchor，而由于没有内容的text节点无法用纯html来创建，只能使用js创建，所以，在ssr中，其前后会使用注释节点来作为Fragment或者for循环的锚点，这样就能在水合节点匹配虚拟节点和真实dom了。

![nuxt服务端渲染时的列表dom](https://image.ncdsf.com/images/20260307221250017.png)

### 编写同构代码

“同构”一词指的是一份代码既在服务端运行，又在客户端运行。因此，在编写组件代码时，应该额外注意因代码运行环境的不同所导致的差异。

- 当组件的代码在服务端运行时，由于不会对组件进行真正的挂载操作，即不会把虚拟 DOM 渲染为真实 DOM 元素，所以组件的 beforeMount 与 mounted 这两个钩子函数不会执行。`又因为服务端渲染的是应用的快照，所以不存在数据变化后的重新渲染，因此，组件的 beforeUpdate 与 updated 这两个钩子函数也不会执行。`另外，在服务端渲染时，也不会发生组件被卸载的情况，所以组件的 beforeUnmount 与 unmounted 这两个钩子函数也不会执行。
- 实际上，只有 beforeCreate 与 created 这两个钩子函数会在服务端执行，所以当你编写组件代码时需要额外注意。
- 遇到需要注意双端渲染的代码时，通常有两个解决方案
  - 方案一：将代码移动到 mounted 钩子中，例如定时器任务；
  - 方案二：使用环境变量包裹这段代码，让其不在服务端运行。
    - 例如：构建工具会分别为客户端和服务端输出两个独立的包。构建工具在为客户端打包资源的时候，会在资源中排除被 import.meta.env.SSR 包裹的代码。
- 编写同构代码的另一个关键点是使用跨平台的 API。
  - 例如，仅在浏览器环境中才存在的 window、document 等对象。然而，有时你不得不使用这些平台特有的 API。`这时你可以使用诸如 import.meta.env.SSR 这样的环境变量来做代码守卫`
  - 类似地，Node.js 中特有的 API 也无法在浏览器中运行。因此，为了减轻开发时的心智负担，我们可以选择跨平台的第三方库。例如，使用 Axios 作为网络请求库。
  - 通常情况下，我们自己编写的组件的代码是可控的，这时我们可以使用跨平台的 API 来保证代码“同构”。然而，第三方模块的代码非常不可控
    - 所以，通常我们会使用条件引入来控制我们的导入代码
- 避免交叉请求引起的状态污染：即不同的请求之间状态出现共享时，导致的状态不一致
  - 编写同构代码时，额外需要注意的是，避免交叉请求引起的状态污染。在服务端渲染时，我们会为每一个请求创建一个全新的应用实例（APP实例）。这是为了避免不同请求共用同一个应用实例所导致的状态污染。
  - 所以，在编写组件代码时，要额外注意组件中出现的全局变量
- clientOnly组件：假设 `<SsrIncompatibleComp />` 是一个不兼容 SSR 的第三方组件，我们没有办法修改它的源代码，这时应该怎么办呢？这时我们会想，既然这个组件不兼容 SSR，那么能否只在客户端渲染该组件呢？
  - 我们可以自行实现一个 `<ClientOnly>` 的组件，该组件可以让模板的一部分内容仅在客户端渲染
  - 其原理是利用了 onMounted 钩子只会在客户端执行的特性。我们创建了一个标记变量 show，初始值为 false，并且仅在客户端渲染时将其设置为 true。这意味着，在服务端渲染的时候，`<ClientOnly>` 组件的插槽内容不会被渲染。而在客户端渲染时，只有等到 mounted 钩子触发后才会将将标记变量设置为false，此时才会渲染 `<ClientOnly>` 组件的插槽内容。这样就实现了被 `<ClientOnly>` 组件包裹的内容仅会在客户端被渲染。
  - 另外，`<ClientOnly>` 组件并不会导致客户端激活失败。因为在客户端激活的时候，mounted 钩子还没有触发，所以服务端与客户端渲染的内容一致，即什么都不渲染
