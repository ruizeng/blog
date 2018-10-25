# 为什么？
经历了纯手工、自定义脚本，到后来的grunt/gulp，前端自动化工具来到了Webpack时代。Webpack是目前使用最广泛的前端打包工具，之所以能提到之前的grunt/gulp类的自动化构建工具，是因为它不仅解决了前端工程化中“自动化”这个问题，更很好的解决了“模块化”这个问题。

前端模块化经历了从全局变量满天飞、命名空间的时代，终于来到了模块化时代，模块化的带来是可以说是前端工程化的至关重要一步。

项目中，我们经常拿着脚手架或各种cli工具生成的Webpack配置，就开始项目的开发的构建了，几乎不太需要理解Webpack的技术细节。但是当我们需要针对项目工程进行定制的时候，看着复杂的配置，就束手无策了。知其然，还得知其所以然，否则就没法完全驾驭好你的工具。Webpack的官方文档对其原理也有非常详尽的描述，本文尝试用更直观的方式对Webpack工作原理进行分析和总结。

# 模块化
早期的js并不是一个支持模块化的语言，为了提高js工程的可维护性和可复用性，模块化是前端工程化的必要一步。于是出现了很多第三方js模块化工具，比较典型的有CommonJS、AMD、CMD等，在es6中也出现了官方的模块化支持。

无论什么js模块化工具，核心做的事情主要就两个：代码封装和依赖管理。以CommonJS为例，我们用下面的简单代码来示例（以下示例和实现只是为了方便演示的简化版本，并不代表webpack的真实实现，但是原理都是类似的）：

``` javascript
// foo.js
module.exports = function() {
    console.log('foo');
};

// bar.js 
module.exports = function() {
    console.log('bar');
}

// main.js
var foo = require("./foo");
var bar = require("./bar");

foo();
bar();
```

## 代码封装

上面的示例代码中我们可以看到，有几个变量（函数）是浏览器不存在的。他们是`module`，`exports`，`require`，那么模块化工具首先需要实现这几个变量并添加到模块代码中，以`foo.js`为例，包装后的代码可能如下：

``` javascript

define('foo.js', function(module, exports, require) {
// +++++ begin of original foo.js +++++
module.exports = function() {
     console.log('foo');
};
// +++++ end of original foo.js +++++
);

```
然后再实现好关键的`define`和`require`两个函数，就可以实现基本的模块代码封装了。这里我们思路也很简单，定一个全局对象存储模块名到具体模块实现代码的映射，这样:
* `define`函数就负责根据模块名将模块代码加载到该全局对象。
* `require`函数负责从模块映射中查找模块代码并将其exports的变量返回给引用模块。

``` javascript
var modules = {

};

function define(name, fn) {
    modules[name] = fn
};

function require(name) {
    var mod = modules[name];
    if (!mod) throw new Error('failed to require "' + name + '"');
    if (!mod.exports) {
        mod.exports = {};
        mod.call(mod.exports, 
            mod, mod.exports, require);
    }
    return mod.exports;
};
```

将上面的定义和包装后的模块代码合并到一个文件，并执行我们的入口模块，最后我们的`bundle.js`就如下了：

``` javascript
var modules = {

};

function define(name, fn) {
    ...
};

function require(name) {
    ...
};

define('./foo.js', function(module, exports, require) {
// +++++ begin of original foo.js +++++
module.exports = function() {
     console.log('foo');
};
// +++++ end of original foo.js +++++
);

define('./bar.js', function(module, exports, require) {
// +++++ begin of original foo.js +++++
module.exports = function() {
     console.log('foo');
};
// +++++ end of original foo.js +++++
);

define('./main.js', function(module, exports, require) {
// +++++ begin of original foo.js +++++
var foo = require("./foo.js");
var bar = require("./bar.js");

foo();
bar();
// +++++ end of original foo.js +++++
);

require('./main.js')
```

哇，就是这么简单，就实现了模块封装的核心思想，No more magic :-）

## 依赖管理

上面的代码有两个重要问题没有得到很好的解决：

1. 我们引用模块的时候很多时候都是采用相对路径，不同模块甚至不在同一个目录，怎么区分呢？
2. 假如模块并没有引用，我们也都打进bundle岂不是浪费么？


# 扩展模块的边界
前面的这些模块化工具之解决了js代码的模块化，但是在前端项目中，我们除了js还有各种其他资源，如html模板、css代码、甚至一些json配置文件、图片等，如果能把这些资源都通过类似模块的方式管理起来，岂不美哉？Webpack就是基于这个（创新的）想法应运而生了！

# 核心概念及实现

# HMR及webpack-dev-server