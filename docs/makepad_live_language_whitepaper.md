---
title: Makepad Live 语言白皮书
author: Albert Yu <dev.yufan@gmail.com>
created_date: 2023-10-31 22:07
updated_date: 2023-11-03 02:37
source: https://github.com/makepad/makepad_docs/blob/cb542da378744f46400f8477395b1fae1036a727/live_language_whitepaper_06052022.pdf
---

# 1. 概述

本文旨在介绍 Live 语言。Live 语言是 Makepad 实时编程环境的基石，允许在运行时改变应用程序的样式。Live 语言不是一门完备的编程语言，它的作用如同 CSS 之于 HTML，即：描述应用程序的表现形式。不同地是，Makepad 使用 Rust 来描述应用程序的结构和行为。

第二节以解释 Live 语言背后的动机为始。第三节通过展示 Live 语言最重要的特性对其进行高层级的介绍。其余章节则采用半正式的写作风格，意在提供参考。第四节介绍 Live 代码块的词法结构。第五节介绍 Live 代码块的语法结构。第六节介绍 Live 代码块中表达式的不同类型。在第七节中，将介绍 Live 代码块所使用的内部表示法。第八节介绍如何将 Live 代码块转换为扁平化节点列表。第九节介绍展开过程（expansion process），在此过程中，继承自其他对象的对象会被展开成扁平对象。

# 2. 动机

本节将揭示 Live 语言背后的动机。

Makepad 的主要目标之一是提供实时编程环境。实时编程环境是一种可在运行时改变应用程序代码的编程环境，并且代码的更改会在程序运行时立即反映出来。

Makepad 应用程序使用 Rust 开发。Rust 是一种多范式、通用编程语言，专为性能和安全而设计。Rust 注重性能和安全性，因此非常适合对渲染（特别是 UI 渲染）有较高要求的应用程序，Makepad 正是为此类应用程序而设计的。

不幸的是，Rust 是一种编译型语言，这一点与 Makepad 提供实时编码环境的目标不相符：每当对代码进行修改时，用编译语言编写的代码都需要重新编译。这种重新编译很容易耗时数秒，很难有实时编码的体验。更糟糕的是，重新编译之后还需要重启应用程序才能反应代码中的变化。

一种显而易见的替代方案是使用脚本语言，比如 JavaScript。用脚本语言编写的代码可以在运行中进行解释或重新编译，从而让开发周期变得更短。遗憾的是，脚本语言缺乏编译语言的性能优势，这又与 Makepad 力求支撑的重渲染应用产生冲突。将脚本语言和 Rust 结合使用时，代码的归属又极难平衡，因为最佳方案总是因应用程序而异的。

为了解决这一困境，我们采取了使用混合方法的方案：Makepad 应用程序使用 Rust 编写，但负责应用程序外观（比如样式）的部分则使用一种 DSL 编写，称之为 Live 语言。Live 语言的主要作用类似于 CSS：它描述了应用程序的用户界面应该如何呈现。特别是与 CSS 相似，Live 语言可以方便地覆盖样式。

> 译注：CSS 的可覆盖样式基于带权重的选择符来实现，高权重的选择符所声明的样式会覆盖低权重的，哪怕它们指向的是相同的 HTML 元素。

相较于 Rust 代码，Live 代码是在运行时编译和执行的。另外，我们为 Makepad 构建的可视化设计应用程序（或称 visual designer——专有名词不翻译了）能感知代码中哪些部分是 Live 代码。当使用 visual designer 修改一段 Live 代码时，可视化设计器不会重新编译应用程序，而是将更新后的 Live 代码发送给运行中的应用程序。这样，应用程序就可以重新编译并执行新的 Live 代码，从而立刻反映出代码中的变化而无需重新编译 Rust 代码或者重启应用程序。

# 3. 高层次介绍

在上一节中，我们介绍了 Live 语言背后的动机。本节将从高层次介绍 Live 语言的使用方法。为此，我们将重点介绍其核心特性，以及这些特性的的使用方法。需要注意的是，本节的内容并非为了全面介绍 Live 语言，而是为了给大家一个总体感觉。如需全面了解 Live 语言，请阅读参考章节。

Live 代码的主要目的是用初始值负责渲染用户界面的 Rust 构造，并在运行时当 Live 代码改变时自动更新这些值。这将是 3.1 节的主题。

Live 代码的一个重要需求是提供可覆盖的样式机制。这是通过允许对象继承其他 Live 对象来实现的。3.2 节将解释这种继承的形式。Live 语言还支持第二种继承形式，即允许对象从 Rust 结构体继承。3.3 节将解释这种继承的形式。Live 代码的一个重要特点是允许嵌入子 DSL。3.4 节将对此做进一步说明。3.5 节将解释 Live 语言是如何支持动态组件的。

## 3.1 初始化和更新 Rust 结构体

本节将说明 Live 代码如何用于初始化和更新 Rust Struct。

思考如下代码片段：

```rust
live_register! {
    Button: {{Button}} {
        bg: {
            color: #FFF
        }
    }
}

#[derive(Live)]
struct Button {
    bg: DrawQuad
}
```

这段代码定义了一个 Rust 构造 `Button`，用于在屏幕上绘制一个按钮。为了简单起见，我们没有在示例中演示如何把按钮渲染出来。除了构造定义，代码还包括调用 `live_register` 宏。该宏用于定义一个 Live 代码块。Live 代码块看起来很像 JSON 文档，但还是有一些重要的区别。

与 JSON 类似，Live 代码块也可以包含原生数据类型、数组和对象。但与 JSON 不同地是，Live 代码块中的对象可以继承自 Rust 结构体。这就是 `Button: {{Button}}` 这行的意思：它定义了一个名为 `Button` 的顶层属性（顶层属性有一个隐式根对象），该属性是一个从 Rust 结构体 `Button` 继承而来的 Live 对象。

当 Live 对象继承自 Rust 结构体时，我们说这个对象为 Rust 结构体提供了一个在 Live 代码中的定义。换言之，这里所做的事情就是在 Live 代码块中为 Rust 结构体提供了一个定义。

注意在 Rust 结构体 `Button` 上使用的属性 `#[derive(Live)]`。该属性用于生成胶水代码，使 `Button` 可以与 Live 代码双向交互。除此之外，它还生成了一个构造函数，可在 Live 代码块中使用与之相对应的定义来实例化 `Button`，具体代码如下：

```rust
let button = Button::new_from_module(
    cx,
    &module_path!(),
    id!(Button)
).unwrap();
```

该函数先是创建了一个 Rust 结构体 `Button` 的实例，将其所有字段设为默认值，然后通过应用 Live 对象 `Button` 将这些字段初始化为合适的值，该对象被定义为当前模块的顶层属性（`module_path` 是一个 Rust 宏，可展开为代表当前模块路径的字符串）。

将 Live 对象应用到 Rust 结构体的默认行为如下。遍历 Live 对象的属性。对于每个字段，检查在 Rust 结构体中是否有同名字段。如果有，首先检查属性和字段类型是否兼容，如果兼容就会发生下列两种情况之一：

- 如果属性和字段都是原生值或数组类型，则 Live 对象中的属性值会覆盖 Rust 结构体中的字段值。
- 如果 Live 对象的属性是对象类型，且字段具有结构体类型，则此应用过程将递归执行。

因此，当调用 `Button::new_from_module` 将 Rust 结构体 `Button` 与 Live 对象 `Button` 进行实例化和初始化时，最终结果将如下所示：

```rust
Button {
    bg: DrawQuad {
        color: Vec4 { x: 1.0, y: 1.0, z: 1.0, w: 1.0 }
        // DrawQuad 的其余属性在此省略
    }
}
```

一切都还不错，不过为什么要以如此复杂的方式初始化 Rust 结构体呢？答案是我们为 Makepad 开发的 visual designer 可以在程序运行时识别对 Live 代码块所做的更改。当 visual designer 检测到这种变化时，它不会出发应用程序的重新编译和重启。反之，它会将新、旧 Live 代码块之间的变化列表发送给应用程序。

于是当运行中的应用程序从 visual designer 接收到这样的差异时，它就会做出响应，更新其 Live 代码块的版本，然后重新应用它来更新 Rust 结构体的值。接着，它会出发应用程序的重新渲染，是新的值立即生效。这得益于 Live 代码块是在运行时而非编译时加载的，因此应用程序可以在内存中保留每一个 Live 代码块的表示形式。Live 代码块的内部表示形式是扁平化的节点列表，这可以让运行时更新代码的速度极快。

## 3.2 从 Live 对象继承

上一节介绍了如何使用 Live 代码来初始化和更新 Rust 结构体。在本节中，将展示 Live 语言如何允许对象从其他 Live 对象继承，以及如何实现可覆盖的样式。

思考如下代码片段：

```rust
live_register! {
    Lable: {{Label}} {
        text: {
            color: #FFF,
        },
        name: "Hello, world!",
    }
}

#[derive(Live)]
struct Label {
    text: DrawText,
    name: String,
}
```

这段代码定义了一个 Rust 结构体 `Label`，用于在屏幕上绘制一个标签。这与之前的例子类似，只是现在的结构体包含了两个字段。由于 `DrawText` 组件并不存储要绘制的文本，因此必须在稍后将其作为参数传递。为了简单起见，实际绘制标签的代码仍然排除在示例之外。

如果调用 `Label::new_from_module` 来实例化和初始化 Rust 结构体 `Label` 和 Live 对象 `Label` 的话，可以得到：

```rust
Label {
    text: DrawText {
        color: Vec4 { x: 1.0, y: 1.0, z: 1.0, w: 1.0 }
        // DrawText 的其余属性在此省略
    },
    name: "Hello, world!",
}
```

到此为止都很顺利。假设现在要创建一种新的标签，它的颜色总是红色。姑且命名为 `RedLabel`。这可以通过扩展 Live 代码块来实现：

```rust
RedLabel: Label {
    text: {
        color: #F00,
    },
}
```

`RedLabel: Label` 一行定义了一个顶层属性 `RedLabel`，它是一个从 Live 对象 `Label` 继承而来的 Live 对象。如果现在调用 `Label::new_from_module` 实例化 Rust 结构体 `Label` 并初始化 Live 对象 `RedLabel` 的话，将会得到：

```rust
Label {
    text: DrawText {
        color: Vec4 { x: 1.0, y: 0.0, z: 0.0, w: 1.0 }
        // DrawText 的其余属性在此省略
    },
    name: "Hello, world!",
}
```

请留意字段 `color` 值来自 Live 对象 `RedLabel`，而字段 `name` 的值依然来自 `Label`。这是因为在加载 Live 代码块时，`RedLabel` 被扩展为：

```rust
RedLabel: {{Label}} {
    text: {
        color: #F00,
    },
    name: "Hello, world!",
}
```

也就是说，`RedLabel` 从 `Label` 继承了属性 `name` 的值，但用自身的值覆盖了属性 `color` 的值。正是通过这种机制，Live 实现了可覆盖的样式。

## 3.3 从 Rust 结构体继承

上一节展示了 Live 语言如何允许对象从其他 Live 对象继承。本节将展示 Live 语言允许的另一种继承形式，从 Rust 结构体继承。

首先将之前示例中的 Live 对象 `Label` 的定义修改为：

```rust
Label: {{Label}} {
    name: "Hello, world!",
}
```

如果现在调用 `Label::new_from_module` 实例化 Rust 结构体 `Label` 并初始化 Live 对象 `Label` 的话，将会得到：

```rust
Label {
    text: DrawText {
        color: Vec4 { x: 1.0, y: 0.0, z: 0.0, w: 1.0 }
        // DrawText 的其余属性在此省略
    },
    name: "Hello, world!",
}
```

注意，尽管没有为 `color` 字段提供定义，但它还是获得了 `#0F0` 的值。那么这个值从何而来呢？答案是：Makepad 为结构体 `DrawText` 提供了内置的 Live 定义，如下所示：

```rust
DrawText: {{DrawText}} {
    color: #0F0,
    // DrawText 的其余属性在此省略
}
```

加载 Live 代码块时，由于 Live 对象 `Label` 继承自 Rust 结构体 `Label`，且该结构体包含了结构体类型为 `DrawText` 的字段 `text`，再加上存在 `DrawText` 的 Live 定义，因此 Live 对象 `Label` 会展开成：

```rust
Label: {{Label}} {
    text: DrawText {
        color: #0F0
    },
    name: "Hello, world!",
}
```

也就是说，属性 `text` 的值来自于顶层属性 `DrawText`，它提供了 Rust 结构体 `DrawText` 的定义，而后者又是 Rust 结构体 `Label` 的字段 `text` 的类型，最终 Live 对象继承自结构体 `Label`。

## 3.4 嵌入子DSL

在本节中将展示 Live 语言如何在 Live 代码中嵌入其他的子 DSL。目前， 我们使用此特性在 Makepad 中嵌入了着色器程序（shader programs）作为应用程序样式的一部份。

请看如下 Live 代码片段：

```rust
A: {
    color = fn(self) -> vec4 {
        return #0F0;
    },
    pixel = fn(self) -> vec4 {
        return mix(#F00, self.color(), 0.5);
    }
}

B: A {
    color = fn(self) -> vec4 {
        return #00F;
    }
}
```

首先要注意的是属性 `color` 使用了分隔符 `=` 而不是 `:`，`=` 定义的是所谓的实例属性，而 `:` 定义的是字段属性。字段属性是 Rust 结构体中存在着对应字段的属性，而实例属性是 Rust 结构体中不存在对应字段的属性。之所以要区分这些不同类型的属性，是因为避免命名空间重叠：即可以在同一个对象上定义同名的字段属性和实例属性。

其次要注意的是以 `fn` 开头的代码块。它们表示函数表达式的开始。函数表达式的作用类似于 Lisp 中的 quote。函数表达式并没有被完全解析，而是以令牌列表（a list of tokens）的形式被嵌入，然后可以根据被应用的不同类型再对其做不同的解释。

以 Makepad 为例，当函数表达式应用于着色器结构体时，将其解释为用 Makepad 着色语言编写的函数。Makepad 着色语言是一种独立于 Live 语言的 DSL。它在很大程度上遵循着 GLSL 语法和语义，并允许将相同的着色器代码转译为 Metal、HLSL 或 GLSL。

在 Live 代码块中嵌入着色器代码作为函数，可以在运行时编辑着色器。由于 Makepad 中的所有 UI 最终都是通过着色器渲染的，因此这个理念极其强大。此外，利用继承功能，函数属性可以像其他值一样被覆盖，从而是衍生的 UI 组件可以触及着色器代码本身。在上例中，属性 `pixel` 是一个定义像素着色器的函数，它调用了函数 `color`。由于对象 B 继承自 A，因此属性 `pixel` 的值相同，但函数 `color` 值不同。实际上，通过重写函数 `color`，可以使函数 `pixel` 的行为参数化。

## 3.5 动态组件

前面的章节节中展示了如何利用 Live 代码为 Rust 结构体的字段提供定义并覆盖其值。然而在 UI 系统中，有很多情况下组件的具体类型在运行前是未知的。在本节中，将解释 Live 语言是如何帮助使用这类动态组件的。

请看下面这段代码：

```rust
#[derive(Live)]
struct Document {
    header: FrameComponentRef
    body: FrameComponentRef
}
```

这里定义了一个 Rust 结构体 `Document`，用于将文档呈现在屏幕上。与前面的示例不同的是，`header` 和 `body` 字段并没有具体类型，而是具有 `FrameComponentRef` 类型，这是一种新类型，它封装了一个特质对象，该对象实现了 `FrameComponent` 特质。

基于这个 Rust 结构体，现在可以用 Live 代码编写以下内容：

```rust
Document {
    header: Fill {
        color: #F00,
    },
    body: Button {}
}
```

`Fill` 和 `Button` 都是动态组件，这意味着它们已经注册了一个工厂函数来创建自己的实例，并曝露了一种应用 Live 值的通用方法（通过使用 `FrameComponent` 特质）。Live 语言并不与特定的组件系统绑定，而是支持通过名称注册任何组件系统。

# 4. 词法结构

本节将介绍 Live 代码块的词法结构。

Live 代码块是以 UTF-8 编码的 Unicode 码点序列。Live 代码块的词法结构将这些码点组合成具有特定含义的序列，也就是令牌（tokens）。令牌可以是注释、标识符、关键字、文字、标点符号或分隔符。接下来的部分，将介绍这些令牌的更多细节。由于 Live 代码的词法结构意在尽可能与 Rust 的词法结构相匹配，所以后续会指明与 Rust 的差异之处。

## 4.1 注释

注释用于以对人可读的文本来对代码进行标注解释，目的是为了让代码更容易阅读。

注释有两种形式：

- 行注释以字符序列 `//` 开始，直至行尾。
- 块注释以字符序列 `/*` 开始，直到字符序列 `*/` 结束。块注释可以嵌套。

与 Rust 不同，不支持文档注释（doc comments）。

## 4.2 标识符

标识符用于标识常量、属性和变量等实体。

标识符以字母或下划线开头，后面跟零或多个数字、字母或下划线。标识符不能是关键字。

与 Rust 不同，标识符不能包含非 ASCII 数字或字母。

## 4.3 关键字

关键字是具有特殊含义的标识符，因此不能（总是）用作命名。Live 代码中的所有关键字都是弱关键字，意思是它们只有在特定的上下文中才有特殊含义。

以下为关键字列表：

- `crate`
- `fn`
- `use`
- `vec2`
- `vec3`
- `vec4`

## 4.4 字面量

字面量是一个单独的令牌，用于直接表示一个值。下面列举了 Live 代码支持的各种字面量。

### 4.4.1 布尔字面量

布尔字面量要么是 `true`，要么为 `false`。

### 4.4.2 整数字面量

整数字面量有四种形式：

- _二进制整数字面量_ 以字符序列 `0b` 开头，后接一个或多个二进制数字或下划线，至少有一个数字。
- _八进制整数字面量_ 以字符序列 `0o` 开头，后接一个或多个八进制数字或下划线，至少有一个数字。
- _十六进制整数字面量_ 以字符序列 `0x` 开头，后接一个或多个十六进制数字或下划线，至少有一个数字。
- _十进制整数字面量_ 以一个十进制数字开头，后接一个或多个十进制数字或下划线，至少有一个数字。

与 Rust 不同，整数字面量后面不能有后缀。

### 4.4.3 浮点数字面量

浮点数字面量有两种形式：

- 一个十进制整数，后面是（英文）句号（译注：即小数点），后面可能接另一个十进制整数，也可能接一个指数
- 一个十进制整数，后接一个指数

如果浮点数字面量是十进制整数后面跟小数点，则后面不能跟下划线或标识符。

与 Rust 不同，浮点数字面量后面不能有后缀。

### 4.4.4 向量（Vec）字面量

向量字面量有三种形式：

- 一个 vec2 字面量以关键字 `vec2` 开头，后面是 2 个浮点数字面量序列，以逗号分隔，并用花括号分界。
- 一个 vec3 字面量以关键字 `vec3` 开头，后面是 3 个浮点数字面量序列，以逗号分隔，并用花括号分界。
- 一个 vec4 字面量以关键字 `vec4` 开头，后面是 4 个浮点数字面量序列，以逗号分隔，并用花括号分界。

### 4.4.5 颜色字面量

颜色字面量以一个哈希符号（#）开始，后接以下形式之一：

- 4 对十六进制数字，表示 RGBA 值。
- 3 对十六进制数字，表示 RGB 值。
- 1 对十六进制数字，表示亮度值（luminance value）。
- 4 个十六进制数字。这是 4 对十六进制数字的缩写形式，代表每一对的两个数字是相同的。
- 3 个十六进制数字。这是 3 对十六进制数字的缩写形式，代表每一对的两个数字是相同的。
- 1 个或 2 个十六进制数字。此种缩写形式代表每个颜色通道的值都是相同的，从而构成一个灰度值。

### 4.4.6 字符串

字符串是由零或多个 Unicode 字符组成的序列，以双引号分界。

特殊字符可以使用_转义符_编码。转义符以反斜线开头，后接以下形式之一：

- 十六进制转义符以字符 `x` 开头，后接两个十六进制数字。表示具有给定值的 ASCII 字符。
- Unicode 转义符以字符 `u` 开头，后接最多六个十六进制数字，并用花括号括起来。表示具有给定值的 Unicode 码点。
- 空白转义符是字符 `n`, `r`，或者 `t`，分别代表换行符（LF）、回车符（CR）、和横向制表符（HT）。
- 空值转义符是字符 `0`，它用于编码空值字符（NUL）。
- 反斜线转义符就是反斜线，它表示反斜线自身。

### 4.4.7 原始字符串

原始字符串以字符 `r` 开头，后面是一个或多个哈希符号和一个双引号。原始字符串的主体是由零或多个字符组成的序列，以第二个双引号和与开头双引号之前相同数量的哈希符号结束。

> 译注：形如 `r###"this is a raw string."###`

## 4.5 标点符号

下表列举了 DSK 中使用的各种标点符号及其使用方法：

| 符号 | 名称     | 使用方法          |
| :--: | :------- | :---------------- |
| `+`  | 加号     | 加法              |
| `-`  | 减号     | 减法、负数        |
| `*`  | 星号     | 乘法              |
| `/`  | 斜线     | 除法              |
| `::` | 双冒号   | 路径分隔符        |
| `:`  | 冒号     | 字段 名/值 分隔符 |
| `=`  | 等号     | 实例 名/值 分隔符 |
| `=?` | 等号问号 | 模版 名/值 分隔符 |
| `,`  | 逗号     | 多项分隔符        |
| `->` | 箭头     | 函数返回值类型    |

## 4.6 分界符

下表列举了 Live 代码中使用的各种分界符：

| 符号 | 名称   |
| :--: | :----- |
| `{}` | 花括号 |
| `[]` | 方括号 |
| `()` | 圆括号 |

# 5. 语法结构

本节将介绍 Live 代码块的语法结构。

Live 代码块的词法结构是一个令牌序列。Live 代码块的语法结构将这些令牌进一步组织成一个解析树（parse tree）。解析树的顶层总是由一系列项组成。项又可以包含表达式。本节的其余部分将会详细介绍项和表达式。

> 译注：此处的“项”对应原文 _items_，后续将不再翻译。因为很难为 _item_ 找到贴切的中文翻译。“项”是准确的，但是读感不太好；“项目”读起来舒适，但却太容易被误认为是 _project_。

## 5.1 Item

An item 是可以出现在 Live 代码块顶层的任意内容。An item 既可以是使用声明（use declaration），也可以是顶层属性定义（top-level property definition）。下面将介绍这些 items。

### 5.1.1 使用声明

> **Syntax**  
> *use_declaration*:  
> `use` *initial_path_segment* (`::` *additional_path_segment*) *final_path_segment*
> 
> *initial_path_segment*:  
> *identifier*  
> | `crate`
> 
> *additional_path_segment*:  
> *identifier*
> 
> *finale_path_segment*  
> *identifier*  
> | `*`

使用声明用于从定义在其他模块的 Live 代码块中导入顶层属性。

使用声明以关键字 `use` 开头，接着是一个 *initial_path_segment*，然后是一串零个或多个路径分隔符（`::`）和 *additional_path_segment*，最后是又一个路径分隔符和一个 *final_path_segment*。

*initial_path_segment* 要么是 *identifier*，要么是关键字 `crate`。

*additional_path_segment* 是 *identifier*。

*final_path_segment* 要么是 *identifier*，要么是通配符（`*`）。

> 译注：原文的语法定义显示在一个方框中，由于是 PDF，所以渲染效果要比 Markdown 好一些。为了尽可能帮助读者阅读和理解，语法定义中的文本在后面的描述中不做翻译而是和语法定义中的原文一一对应。随后的章节也会照此处理。

### 5.1.2 顶层属性定义

> **Syntax**  
> *top_level_property_definition*:  
> *identifier<sup>?</sup>* *identifier* `=` *object_expression*

顶层属性定义用于在隐式根对象上定义属性。

顶层属性定义以一个可选的 *identifier* 开头，接着是另一个 *identifier*，然后是一个实例名/值分隔符（`=`），最后是一个对象表达式。

## 5.2 表达式

表达式是可求值的令牌序列。

### 5.2.1 基础表达式

#### 5.2.1.1 字面量表达式

字面量表达式就是前文介绍过的字面量之一。它的求值结果就是字面量的值。

#### 5.2.1.2 数组表达式

> **Syntax**  
> *array_expression*:  
> `[`*primary_expression*(*, primary_expression*)*<sup>\*</sup>,<sup>?</sup>*`]`

数组表达式是由一个或多个 *primary_expression* 组成的序列，中间用逗号分隔，并用方括号分界。它的求值结果是一个数组类型的值。

#### 5.2.1.3 对象表达式

> **Syntax**  
> *object_expression*:  
> *base_object_specifier<sup>?</sup>* `{` (*property_definition*(, *property_definition*)*<sup>\*</sup>*)*<sup>?</sup>* `}`
>
> *base_object_specifier*:  
> *identifier*  
> | { { `identifier` } }
>
> *property_definition*:  
> *identifier<sup>?</sup> identifier* `:` *value*  
> *identifier<sup>?</sup> identifier* `=` *value*  
> *identifier<sup>?</sup> identifier* `=?` *value*

对象表达式以一个可选的 *base_object_specifier* 开头，然后是一个或多个键值对序列，以逗号分隔，并用花括号分界。它的求值结果是一个对象类型的值。

*base_object_specifier* 有两种形式：

- Live *base_object_specifier* 是一个 *identifier*。
- Rust *base_object_specifier* 是一个 *identifier* 并且用双花括号括起来。

*property_definition* 有三种形式：

- 字段属性定义以一个可选的 `identifier` 开头，接着是另一个 `identifier`，然后是字段名/值分隔符（`:`），最后是一个值。
- 实例属性定义以一个可选的 `identifier` 开头，接着是另一个 `identifier`，然后是实例名/值分隔符（`=`），最后是一个值。
- 模块属性定义以一个可选的 `identifier` 开头，接着是另一个 `identifier`，然后是模版名/值分隔符（`=?`），最后是一个值。

#### 5.2.1.4 函数表达式

> **Syntax**  
> *function_expression*:  
> `fn (`*token_list*`)` (`->` *identifier*)*<sup>?</sup>* `{` *token_list* `}`

*function_expression* 以关键字 `fn` 开头，后面是一个用圆括号分届的 `token_list`，接着是一个可选的箭头（`->`）和 *identifier*，最后是一个用花括号分界的 *token_list*。

#### 5.2.1.5 标识符表达式

> **Syntax**  
> *identifier_expression*:  
> *identifier*

*identifier_expression* 就是一个标识符。

#### 5.2.1.6 分组表达式

> **Syntax**  
> *grouped_expression*:  
> `(`*expression*`)`

*grouped_expression* 是一个用圆括号分界的表达式。

### 5.2.2 复合表达式

#### 5.2.2.1 调用表达式

> **Syntax**  
> *call_expression*:  
> *identifier* `(`(*expression*(*, expression*)*<sup>\*</sup>*)*<sup>?</sup>*`)`

*call_expression* 以 *identifier* 开头，后面是一串 *expression*，以逗号分隔，并用圆括号分界。

#### 5.2.2.2 运算符表达式

> **Syntax**  
> *operator_expression*:  
> *unary_operator expression*  
> *expression binary_operator expression*

*operator_expression* 有两种形式：

- *unary_operator*，后接 *expression*
- *expression*，后接 *binary_operator*，再接 *expression*

> 译注：*unary_operator* 是一元运算符，*binary_operator* 是二元运算符

一元运算符（`-`）可对整数、浮点数标量和向量进行运算。若操作数是标量则执行取负值操作，得到另一个标量。若操作数是向量，标量的取负运算将分别应用于向量的每个分量，其结果是另一个相同大小的向量。

二元运算符（`+`）、（`-`）、（`*`）、（`/`）可对整数、浮点数标量和向量进行运算。如果一个操作数是整数但另外一个不是，那么整数运算符将转换为浮点数运算符。转换之后，以下情况是有效的：

- 两个操作数都是标量。此时，执行操作后会产生另一个标量。
- 一个操作数是标量，另一个操作数是向量。此时，标量运算会分别应用于向量的每个分量，从而得到另一个相同大小的向量。
- 两个操作数是大小相同的向量。此时，运算符按分量作用，结果是另一个相同大小的向量。

# 6. 类型

本节将介绍 Live 代码块中每个表达式的不同类型。

Live 代码块中的每个表达式都有一个类型。表达式的类型决定了它的取值，以及可以对它执行的操作。类型分为原生类型（不能经由其他类型组成）和复合类型（可以经由其他类型组成）。接下来，将详细介绍原生类型和复合类型。

## 6.1 原生类型

### 6.1.1 布尔类型

布尔类型（`bool`）可以取 `true` 和 `false` 这两种值。

当前还不能对布尔类型执行任何操作。

### 6.1.2 整数类型

整数类型（`int`）可以取从 `-2`<sup>`63`</sup> 到 `2`<sup>`63`</sup>` - 1` 之间的任何整数值。

`int` 类型的值可以取负值，或与其他 `int` 类型的值相加/相减/相乘，从而得到另一个 `int` 类型的值。

### 6.1.3 浮点类型

浮点类型（`float`）与 IEEE754 的 "binary64" 类型相对应。

`float` 类型的值可以取负值，或与其他 `float` 类型的值相加/相减/相乘，从而得到另一个 `float` 类型的值。此外，`float` 数值可以与 `vec2`、`vec3` 或 `vec4` 类型的其他数值相加/相减/相乘/相除。此时，标量运算会分别应用于向量的每个分量，分别产生另一个 `vec2`、`vec3` 或 `vec4` 类型的值。

### 6.1.4 Vec2 类型

`vec2` 类型包括所有以二元组（2-tuples）组成的浮点类型数值。

`vec2` 类型的值可以取负值，或与其他 `vec2` 类型的值相加/相减/相乘/相除。此时，运算符是按分量作用的，结果是另一个 `vec2` 类型的值。或者，也可以将 `vec2` 类型的值与其他 `float` 类型的值相加/相减/相乘/相除。此时，标量运算会分别应用于向量的每个分量，从而产生另一个 `vec2` 类型的值。

### 6.1.5 Vec3 类型

`vec3` 类型包括所有以三元组（3-tuples）组成的浮点类型数值。

`vec3` 类型的值可以取负值，或与其他 `vec3` 类型的值相加/相减/相乘/相除。此时，运算符是按分量作用的，结果是另一个 `vec3` 类型的值。或者，也可以将 `vec3` 类型的值与其他 `float` 类型的值相加/相减/相乘/相除。此时，标量运算会分别应用于向量的每个分量，从而产生另一个 `vec3` 类型的值。

### 6.1.6 Vec4 类型

`vec4` 类型包括所有以四元组（4-tuples）组成的浮点类型数值。

`vec4` 类型的值可以取负值，或与其他 `vec4` 类型的值相加/相减/相乘/相除。此时，运算符是按分量作用的，结果是另一个 `vec4` 类型的值。或者，也可以将 `vec4` 类型的值与其他 `float` 类型的值相加/相减/相乘/相除。此时，标量运算会分别应用于向量的每个分量，从而产生另一个 `vec4` 类型的值。

### 6.1.7 颜色类型

颜色类型（`color`）由以四元组（4-tuples）组成的浮点类型数值组成，其中各分量的数值被约束（clamped）在区间 `[0.0, 1.0]` 之中。 

### 6.1.8 字符串类型

字符串类型（`string`）由所有 Unicode 字符序列组成。

当前还不能对字符串类型执行任何操作。

### 6.1.9 函数类型

函数类型（`function`）由构成函数的所有令牌序列组成。

当前还不能对函数类型执行任何操作。

## 6.2 复合类型

### 6.2.1 数组类型

数组类型（`array`）由 `N` 个 `T` 类型的值的全部序列组成。

当前还不能对数组类型执行任何操作。

### 6.2.2 对象类型

对象类型（`object`）由所有具名属性的集合组成。

当前还不能对对象类型执行任何操作。

# 7. 内部表示法

本节将介绍 Live 代码块的内部表示法。

Live 代码块定义了一个由原始值、数组、对象或表达式组成的数据结构。这种数据结构在内部表示为扁平化的节点列表（flattend list of nodes）。遍历该列表可按深度优先的顺序获得数据结构的节点。之所以用扁平化的节点列表是因为它可以在运行时极快速地更新代码。

节点代表一个对象属性，或是一个数组元素。代表着顶层属性或对象属性的节点包含该属性的名称和值。代表数组元素的节点只包含数组元素的值。

此外，每个节点都有一个原点（origin），定义了节点在令牌流（token stream）中的定义位置，还有一组标志（flags），标志有多种用途，例如表示节点所代表的属性类别（the kind of property）、包括 *字段*、*实例*和*模版*，或者指示名称是否包含前缀。请注意，节点中并不包含前缀，但可以使用原点从令牌流中获取。

值（value）是一种枚举数据类型，可以用不同的方式表示一个值，具体取决于该值是如何定义的。

如果值是用字面量表达式、数组或对象表达式定义的，则直接存储表达式的值。字面量表达式定义的原生值，如 `boo`、`int` 和 `float`，用它们自身来表示。数组或对象表达式定义的复合值（compound value），也就是数组和对象，用一个指示数组或对象起始的值来表示，然后跟着零或多个节点序列来表示数组/对象的元素/属性。这个序列由一个具有特殊值 `close` 的节点结束。

当使用函数表达式定义值时，令牌序列即构成函数。

当使用标识符表达式定义值时，存储的不是标识符的值，而是标识符本身，需要稍后进行解析。

当使用复合表达式定义值时，存储的不是表达式的值，而是表达式本身，需要稍后进行求值。运算符和内置函数调用都有自己的值类型，后面跟着固定数量的节点来表示运算符的操作数或函数调用的参数。

下表列出了节点可包含的不同的值及其含义：

| **值**      | **含义**        |
| :---------- | :-------------- |
| `bool(x)`   | bool 类型的值   |
| `int(x)`    | int 类型的值    |
| `float(x)`  | float 类型的值  |
| `vec2(x)`   | vec2 类型的值   |
| `vec3(x)`   | vec3 类型的值   |
| `vec4(x)`   | vec4 类型的值   |
| `color(x)`  | color 类型的值  |
| `string(x)` | string 类型的值 |
| `array(x)`  | array 类型的值<br /> 具有此值的节点后接零或多个节点序列来表示数组的元素。这个序列由一个具有特殊值 `close` 的节点结束。|
| `object(x)`  | object 类型的值<br /> 具有此值的节点后接零或多个节点序列来表示对象的属性。这个序列由一个具有特殊值 `close` 的节点结束。|
| `clone(id)`  | object 类型的值<br /> 它与 `object` 类似，只是继承自名为 **id** 的 DSL 对象，然后用附加属性对其进行扩展。<br /> 具有此值的节点后接零或多个节点序列来表示对象的属性。这个序列由一个具有特殊值 `close` 的节点结束。<br /> 具有此值的节点会在展开过程中被替换，并在展开过程中得到进一步解释（见下文）。|
| `class(typeid)`  | object 类型的值<br /> 它与 `object` 类似，只是继承自名为 **typeid** 的 Rust 结构体，然后用附加属性对其进行扩展。<br /> 具有此值的节点后接零或多个节点序列来表示对象的属性。这个序列由一个具有特殊值 `close` 的节点结束。<br /> 具有此值的节点会在展开过程中被替换，并在展开过程中得到进一步解释（见下文）。|
| `close` | 表示数组或对象结束的特殊值 |
| `fn(tokens)` | function 类型的值<br /> 该类型的值表示指向令牌列表的指针，根据 Rust 结构体的不同，对其的解释（interpreted）也不同。 |
| `ident(id)` | 名为 *id* 的项（item）<br /> 在获得最终值之前，需要对这种类型的值进行解析。 |
| `unop(op)` | 表示将一元运算符应用于操作数之后的结果值<br /> 具有该值的节点后面还有另一个节点，代表操作数。这种类型的值要经过求值才能得到最终的值。 |
| `binop(op)` | 表示将二元运算符应用于操作数之后的结果值<br /> 具有该值的节点后面还有两个个节点，分别代表两个操作数。这种类型的值要经过求值才能得到最终的值。 |
| `call(id, n)` | 表示将函数 *id* 应用于 *n* 个参数的结果值<br /> 具有该值的节点后面还有 *n* 个其他节点，代表参数。这种类型的值需要经过求值才能得到最终的值。 |
| `use(path)` | 表示在进行名称解析（name resolution）的过程中所使用的特殊值。<br />具有此值的节点会在展开过程中使用，并在展开过程中作进一步解释（见下文）。 |

# 8. 转译过程

本节将介绍把 Live 代码块的解析树转译成扁平化节点列表的过程。

为了帮助描述转译过程，我们将引入一些符号。以把 Live 句法形式 `2 + 3` 转移成扁平化节点列表 `binop(+) 2 3` 为例。这个转译过程可看作是关于函数 *T*，以句法形式作为输入并且返回节点列表作为输出。可以把该转译过程写成一个等式：

```math
T[[2 + 3]] = binop(+) 2 3
```

如果句法形式包含任何非终止符（non-terminals）,则需要对这些非终止符递归应用 *T*。例如，二元运算符表达式的一般转译方案是：

```math
T[[expression\ binary\_operator\ expression]] = binop(op)\ T[[expression]]\ T[[expression]]
```

其中，*op* 是操作数。

应用在等式左边句法形式上的 EBNF 运算符，如 <sup>?</sup> 和 <sup>\*</sup>，也可以应用于等式右边相应 *T* 的 结果。这应被解释为：等式左边的句法形式出现多少次，相应的 *T* 的结果就应该重复出现多少次。

例如，在下面的转译方案中：

```math
T[[expression^?]] = T[[expression]]^?
```

转译后右边的表达式 $T[[expression]]$ 只有在左边出现的时候才会出现。

类似的，以下转译方案也是如此：

```math
T[[expression^*]] = T[[expression]]^*
```

转译后表达式 $T[[expression]]$ 在右边出现的次数应该和左边的一样多。

在本节其余部分的内容里，将介绍构成 Live 代码块的不同句法形式的一般转译方案。

## 8.1 Items

### 8.1.1 使用声明

使用声明的一般转译方案是：

```math
T[[use\ initial\_path\_segment(::\ additional\_path\_segment)\ final\_path\_segment]] = use(path)
```

### 8.1.2 顶层属性定义

顶层属性定义的一般转译方案是：

```math
T[[identifier^?\ identifier\ :\ object\_expression]] = identifier^?\ identifier = T[[object\_expression]]
```

## 8.2 表达式

### 8.2.1 基础表达式

#### 8.2.1.1 字面量表达式

字面量的转译方案非常简单。只需将字面量翻译成具有字面量值的节点即可。共有：

```math
T[[boolean\_literal]] = bool(x)
```

```math
T[[integer\_literal]] = int(x)
```

```math
T[[floating\_point\_literal]] = float(x)
```

```math
T[[vec2\_literal]] = vec2(x)
```

```math
T[[vec3\_literal]] = vec3(x)
```

```math
T[[vec4\_literal]] = vec4(x)
```

```math
T[[color\_literal]] = color(x)
```

```math
T[[string\_literal]] = string(x)
```

```math
T[[raw\_string\_literal]] = string(x)
```

其中 *x* 是字面量的值。

#### 8.2.1.2 数组表达式

形如下例的数组表达式：

`[2, 3]`

可转译为：

`array int(2) int(3) close`

一般来说，数组表达式会被转译成一个表示数组开始的节点、一个表示数组结束的节点，以及中间的零或多个节点，这些节点是定义数组元素的主表达式递归转换的结果。

因此，数组表达式的一般转译方案是：

```math
T[[\ [primary\_expression(,\ primary\_expression)^*,^?]\ ]] = array\ T[[primary\_expression]]\ T[[primary\_expression]]^*\ close
```

#### 8.2.1.3 对象表达式

形如下例的对象表达式：

`{ x: 2, y: 3 }`

可转译为：

`object x: int(2) y: int(3) close`

一般来说，对象表达式会被转译成一个表示对象开始的节点、一个表示对象结束的节点，以及中间的零或多个节点，这些节点是定义对象属性的属性定义递归转换的结果。

因此，对象表达式的一般转译方案是：

```math
T[[\{\ (property\_definition(,\ property\_definition)^*)^?\ \}]] = object\ (T[[property\_definition]]\ T[[property\_definition]]^*)^?\ close
```

```math
T[[identifier\ \{\ (property\_definition(,\ property\_definition)^*)^?\ \}]] = clone(id)\ (T[[property\_definition]]\ T[[property\_definition]]^*)^?\ close
```

```math
T[[\{\ \{\ identifier\ \}\ \}\ \{\ (property\_definition(,\ property\_definition)^*)^?\ \}]] = class(id)\ (T[[property\_definition]]\ T[[property\_definition]]^*)^?\ close
```

其中 *id* 是标识符的名称。

属性定义被转译为属性节点。共有：

```math
T[[identifier^?\ identifier\ :\ object\_expression]] = identifier^?\ identifier\ :\ T[[object\_expression]]
```

```math
T[[identifier^?\ identifier = object\_expression]] = identifier^?\ identifier = T[[object\_expression]]
```

```math
T[[identifier^?\ identifier =?\ object\_expression]] =?\ identifier^?\ identifier =?\ T[[object\_expression]]
```

#### 8.2.1.4 函数表达式

函数表达式的转译方案非常简单，只需将组成表达式的令牌列表转换为指向该列表指针的节点：

```math
T[[fn\ (token\_list)\ (->\ identifier)^?\ \{\ token\_list\ \}]] = fn(tokens)
```

其中 *tokens* 是指向组成表达式的令牌列表的指针。

#### 8.2.1.5 标识符表达式

标识符表达式的翻译方案非常简单，只需要将标识符翻译成带有标识符名称的节点即可：

```math
T[[identifier]] = idnet(id)
```

#### 8.2.1.6 分组表达式

分组表达式的一般转译方案是：

```math
T[[(expression)]] = T[[expression]]
```

### 8.2.2 复合表达式

#### 8.2.2.1 调用表达式

形如下例的调用表达式：

`f(2, 3)`

可转译为：

`call(f, 2) 2 3`

调用表达式的一般转译方案是：

```math
T[[identifier\ ((expression(,\ expression)^*)^?)]] = call(identifier,\ n)\ (T[[expression]](,\ T[[expression]])^*)^?
```

其中 *n* 是调用的参数个数。

#### 8.2.2.2 运算符表达式

形如下例的一元运算符：

`-1`

可转译为：

`unop(-) 1`

一元运算符表达式的一般转译方案是：

```math
T[[unary\_operator\ expression]] = unop(op)\ T[[expression]]
```

其中 *op* 是运算符。

形如下例的二元运算符表达式：

`2 + 3`

可转译为：

`binop(+) 2 3`

二元运算符的一般转译方案是：

```math
T[[expression\ binary\_operator\ expression]] = binop(op)\ T[[expression]]\ T[[expression]]
```

其中 *op* 是运算符。

# 9. 展开

在展开过程中，代表 Live 代码块的原始节点列表会转换成展开的节点列表，其中从其他 Live 对象或 Rust 结构体中继承的对象会展开成扁平对象。

为了说明展开过程，请看下面的 Live 代码块：

```rust
A = {
    x: 2.0
}

B = A {
    y: 3.0
}
```

这个 Live 代码块的原始节点列表如下所示：

```
A: object
x: float(2.0)
close
B: clone(A)
y: float(3.0)
close
```

在展开之后，展开的节点列表如下所示：

```
A: object
x: float(2.0)
close
B: object
x: float(2.0)
y: flaot(3.0)
close
```

一般来说，展开过程如下：给定一个原始节点列表，首先创建一个空的展开节点列表。然后从左到右遍历原始节点列表中的节点。每迭代一步，都会使用以下规则将当前节点添加到展开的节点列表中：

- 若节点代表一个对象属性，且当前正在展开的对象上已经存在一个同名属性：
    - 若被重载的属性表示对象类型的值，则该属性将被重载：
        - 递归合并现有对象和新对象
    - 否则：
        - 用新值替换现有值
- 否则：
    - 将节点添加到列表末尾

上述规则有两种例外情况：如果当前节点的值是 `clone(id)` 或 `class(typeid)`，则会对该节点进行特殊处理。下面的章节中将会解释这两种情况。最后一节将解释展开过程中的名称解析过程是如何工作的。

## 9.1 克隆节点

如果当前节点的值是 `clone(id)`，那么它就代表一个继承自另外一个 Live 对象的对象的开始。我们将把这个对象称为*子对象*，把它继承的对象称为*父对象*。由于展开过程是从左向右迭代的，于是可以假定父对象已经被添加到展开的节点列表中，并且已经完全展开。因此，我们的目标是找到构成父对象的节点，并将它们复制到已展开节点列表的末尾。

为此，首先需要在展开的节点列表中找到父对象的起点。可通过在列表中向后搜索，直到找到表示父对象开始的节点。这将是一个代表对象属性的节点，其名称与父对象的 id 一致，值为 `object(id)` 或 `class(id)`。需要注意的是，由于父对象已经完全展开，它永远不会以一个值为 `clone(id)` 的节点开始。

一旦找到了表示父对象开始的节点，我们就会将构成父对象的节点复制到展开的节点列表的末尾。这包括表示父对象开始的节点和表示对象属性的节点，但不包括表示父对象结束的节点，因为我们仍要为子对象继续添加其他属性。

## 9.2 类节点

如果当前节点的值是 `class(typeid)`，它就代表一个从 Rust 结构体继承的对象的开始。一次最多只能有一个对象继承自同一个 Rust 结构体。我们称这个对象为 Live 代码块中的 Rust 结构体提供了一个定义。为了方便讨论，我们假设存在从 `typeids` 到类型描述符的映射。类型描述符（type descriptor）通过列举结构体上所有字段的名称和 `typeids` 来描述一个 Rust 结构体，这些字段的类型在 Live 代码块中都有定义。

要展开具有此值的节点，首先要将节点追加到展开的节点列表中。然后，使用节点的 `typeid` 查找相应的类型描述符。对于该类型描述符中的每个字段，都会在展开的节点列表中向后搜索，直到找到用于表示提供字段类型定义的对象开始的节点。这将是一个值为 `class(typeid)` 的节点，其中 `typeid` 与字段的 `typeid` 一致。

找到表示对象开始的节点后，我们会将组成对象的节点复制到展开的节点列表的末尾。这包括表示父对象开始的节点、表示对象属性的节点和表示父对象结束的节点。

## 9.3 名称解析

展开时使用的名称解析过程非常简单：只需在展开的节点列表中向后搜索，直到找到期望要找的节点。需要注意的是，这将带来意料中的覆盖行为（shadowing behavior）：如果在同一个对象上定义了两个同名的属性，则后定义的属性将对前定义的属性产生覆盖。此外，如果从当前节点到根节点的路径上有两个名称相同的属性，那么最靠近当前节点的属性将优先被使用。

为便于在展开过程中进行跨模块名称解析，使用声明将作为具有特殊值 `use(path)` 的节点纳入节点列表。当在名称解析过程中遇到这样的节点时，会在展开的节点列表中查找与给定路径相对应的 Live 代码块，然后在该节点列表中*向前*搜索，直到找到要找的节点。