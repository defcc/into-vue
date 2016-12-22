## Vue.js 模版解析过程分析

### 模版和解析说明

先来看一段 Vue.js 的模版

```html
<div class="cls-name" id="app">
  <p v-html="123"></p>
  <span>hello</span>
</div>
```

Vue.js 对于 HTML 模版的解析基于 http://ejohn.org/files/htmlparser.js ，并在其之上做了一些精简和调整。Vue 基于此进行 HTML 字符串解析得到节点对象，并对节点的属性进行分析，生成想要的 AST。

本文将会从两个方面进行说明，纯 HTML 字符串解析 以及 Vue.js 的 AST 的生成。

### 纯 HTML 解析

HTML 节点有 tagName，attributeList <name: value> 等，一个基本的 HTML 解析器需要分析出 HTML 节点信息，最终产出一个节点树。通常为了更好的控制解析过程，会基于状态机机制进行分析。

HTMLParser 中提供了三个接口：

1. option.start，在解析到新的节点时调用，包括节点 tagName, attributes 等信息
2. option.end 在节点解析结束时调用，包括节点 tagName 等信息
3. option.chars 在文本解析完成时调用，包括文本自身

拿 `<span v-show="true">123</span>` 来说

1. 当解析到 <span v-show="true"> 时，会调用 start

  `option.start('span', [{name: 'v-show', value: 'true'}])`

2. 当解析到 123 时，会调用 chars 方法

  `option.chars('123')`

3. 当解析到 </span> 时，会调用 end 方法
  
  `option.end('span')`

其中有一些细节需要在解析时注意

1. 一些标签不需要手动闭合，比如 p，td，th 等。对于这些标签，如果出现了形如 `<p>123<p>` ，需要闭合第一个 p 标签。
2. 一些标签不能出现在 p 中，比如 div li 等块级标签。对于这些标签，如果出现了形如 `<p>123<div>`，需要闭合第一个 p 标签。

事实上，我们可以实现一个更严格更语义化的解析器，比如 table 里只允许有 tbody，tr 等子元素，tr 里只允许有 th，td 等字元素。只是处于代码体积方面考虑，Vue.js 目前的解析没有这么严格。

HTMLParser 在解析模版的过程中，将节点信息通过 start， end 和 chars 三个方法传递给 Vue.js 的 ast 构建程序，边解析边同时构建模版的 ast。

### AST 生成

拿到模版基本节点信息后，Vue.js 将对其进行模版语法的解析，比如 if 分支，for 循环，插值表达式，slot 等，处理为 AST 节点，最终生成一个完整的 AST 树

模版的 AST 节点有 3 种类型 ASTElement，ASTText 和 ASTExpression，可以在 flow 目录的 [compiler.js](https://github.com/vuejs/vue/blob/dev/flow/compiler.js#L63) 看到节点定义信息。

1. ASTElement 基于 option.start 调用时的 tag 节点信息所创建。
2. ASTText 和 ASTExpression 两种节点是在 option.char 时创建，ASTText 即为纯文本节点，ASTExpression 包含了插值表达式。
模版语法层面的解析包含了基本语法解析，如 pre，v-for，v-if，v-else-fi，v-else，v-once，key, ref, is，以及 v-bind，v-on 和 自定义 directive 的处理。具体解析代码在 [https://github.com/vuejs/vue/blob/dev/src/compiler/parser/index.js](https://github.com/vuejs/vue/blob/dev/src/compiler/parser/index.js)

为方便扩展，Vue.js 为模版语法的解析过程提供了三个接口 preTransformNode、transformNode、postTransformNode。

1. preTransformNode，在建立 ASTElement 基本信息后，进行节点的信息转换处理
2. transformNode，方便各个平台无关性模块扩展，在处理完 Vue.js 内置的基本语法解析后调用，如 class 模块和 style 模块即在这个阶段处理分析完成。而 web platform 和 weex platform 对这两个模块的定义是不一样的，基于 transformNode 的机制变可以很好的实现核心的可扩展性。
3. postTransformNode，即在节点解析完后调用

其中 preTransformNode 和 postTransformNode 只是预留了这两个接口，目前在 Vue.js 的核心中还没使用到。

至此即得到了对应于 HTML 模版的 AST 树，下一篇将会介绍基于 AST 树的优化。

==================

#### 附录补充，关于 ast 和解析相关的一些资料：

1. [ast 介绍](https://en.wikipedia.org/wiki/Abstract_syntax_tree)
2. [estree](https://link.zhihu.com/?target=https%3A//github.com/estree/estree)
3. [esprima pasring demo](https://link.zhihu.com/?target=http%3A//esprima.org/demo/parse.html)
4. [top down operator precedence](https://link.zhihu.com/?target=http%3A//javascript.crockford.com/tdop/tdop.html)
