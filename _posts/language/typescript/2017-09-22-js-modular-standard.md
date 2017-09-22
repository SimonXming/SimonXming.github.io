---
layout: post
title: Javascript 模块化标准
category: 语言
tags: javascript 模块化
keywords: javascript
description:
---

## 前言

最近正打算看一下 typescript 编译器怎么用，然后我输入了以下命令:

```shell
➜  example-tmp ./node_modules/typescript/bin/tsc --help
Version 2.3.4
Syntax:   tsc [options] [file ...]

...

 -w, --watch                                        Watch input files.
 -t VERSION, --target VERSION                       Specify ECMAScript target version: 'ES3' (default), 'ES5', 'ES2015', 'ES2016', 'ES2017', or 'ESNEXT'.
 -m KIND, --module KIND                             Specify module code generation: 'commonjs', 'amd', 'system', 'umd' or 'es2015'.
 --lib                                              Specify library files to be included in the compilation:
                                                      'es5' 'es6' 'es2015' 'es7' 'es2016' 'es2017' 'esnext' 'dom' 'dom.iterable' 'webworker' 'scripthost' 'es2015.core' 'es2015.collection' 'es2015.generator' 'es2015.iterable' 'es2015.promise' 'es2015.proxy' 'es2015.reflect' 'es2015.symbol' 'es2015.symbol.wellknown' 'es2016.array.include' 'es2017.object' 'es2017.sharedmemory' 'es2017.string' 'esnext.asynciterable'
```

对于 `-m` 和 `--lib` 参数意味的东西我很感兴趣，就顺便查了一下详细资料。

## -m KIND 参数

简单来说，这个参数的由来是因为 Javascript 模块化。网页越来越像桌面程序，需要一个团队分工协作、进度管理、单元测试等等... 开发者不得不使用软件工程的方法，管理网页的业务逻辑。

但是，Javascript 不是一种模块化编程语言，它不支持 " 类 "（class），更遑论" 模块 "（module）了。（ECMAScript 标准第六版，将正式支持 "类" 和 "模块"，但还需要很长时间才能投入实用。）
Javascript 社区做了很多努力，在现有的运行环境中，实现 "模块" 的效果。

### CommonJS

CommonJS 最开始是 Mozilla 的工程师于 2009 年开始的一个项目，它的目的是让浏览器之外的 JavaScript （比如服务器端或者桌面端）能够通过模块化的方式来开发和协作。

在 CommonJS 的规范中，每个 JavaScript 文件就是一个独立的模块上下文（module context），在这个上下文中默认创建的属性都是私有的。也就是说，在一个文件定义的变量（还包括函数和类），都是私有的，对其他文件是不可见的。

如果想在多个文件分享变量，第一种方法是声明为 global 对象的属性。但是这样做是不推荐的，因为大家都给 global 加属性，还是可能冲突。

推荐的做法是，通过 module.exports 对象来暴露对外的接口。

Node 就采用了 CommonJS 规范来实现模块依赖。

我们可以这样创建一个最简单的模块：

```js

function myModule() {
  this.hello = function() {
    return 'hello!';
  }

  this.goodbye = function() {
    return 'goodbye!';
  }
}

module.exports = myModule;
```

我们可以注意到，在定义了自己的 function 之后，通过 module.exports 来暴露了出去。为什么我们可以在没有定义 module 的情况下就使用它？因为 module 是 CommonJS 规范中预先已经定义好的对象，就像 global 一样。

如果其他代码想使用我们的 myModule 模块，只需要 require 它就可以了。

```js
var myModule = require('myModule');

var myModuleInstance = new myModule();
myModuleInstance.hello(); // 'hello!'
myModuleInstance.goodbye(); // 'goodbye!'
```

这种做法有两个明显的优势：

1. 避免全局命名空间污染，require 进来的模块可以被赋值到自己随意定义的局部变量中，所以即使是同一个模块的不同版本也可以完美兼容
2. 让各个模块的依赖关系变得很清晰

有的时候我们实现一个模块需要的代码量比较大，会再次分解成若干文件，然后放在一个目录中。这时，我们需要在该目录中放置一个 package.json 文件，在 main 字段中指定一个入口文件（比如 index.js）。

这样，其他人就能够使用 require 方法，加载整个目录。

需要注意的是，CommonJS 规范的主要适用场景是服务器端编程，所以采用同步加载模块的策略。如果我们依赖 3 个模块，代码会一个一个依次加载它们。

因为服务器端的模块加载主要来源硬盘、或者内存，所以加载速度比较快，同步加载并不是很大问题。但是如果是在浏览器场景中，同步加载就有很大问题，因为从网络中加载一个模块比从硬盘加载慢得多。在等待加载的过程中，浏览器会挂起当前的进程，直到模块下载完成。

Node 开发社区的 SubStack 大神开发了一个 browserify 工具。browserify 是一个开发侧解决方案，它可以把需要 require 进来的 a/b/c 等模块文件全部打包合并到一个单独的 JavaScript 文件中（这个过程称为 bundle）。

### AMD

介绍了同步方案，我们当然也有异步方案。在浏览器端，我们更常用 AMD 来实现模块化开发。AMD 是 Asynchronous Module Definition 的简称，即 “异步模块定义”。

我们看一下 AMD 模块的使用方式：

```js
define(['myModule', 'myOtherModule'], function(myModule, myOtherModule) {
  console.log(myModule.hello());
});
```

在这里，我们使用了 define 函数，并且传入了两个参数。

第一个参数是一个数组，数组中有两个字符串也就是需要依赖的模块名称。AMD 会以一种非阻塞的方式，通过 appendChild 将这两个模块插入到 DOM 中。在两个模块都加载成功之后，define 会调用第二个参数中的回调函数，一般是函数主体。

第二个参数也就是回调函数，函数接受了两个参数，正好跟前一个数组里面的两个模块名一一对应。因为这里只是一种参数注入，所以我们使用自己喜欢的名称也是完全没问题的。

同时，define 既是一种引用模块的方式，也是定义模块的方式。

例如，myModule 的代码可能看上去是这样：

```js
define([], function() {

  return {
    hello: function() {
      console.log('hello');
    },
    goodbye: function() {
      console.log('goodbye');
    }
  };
});
```

所以我们可以看到，AMD 优先照顾浏览器的模块加载场景，使用了异步加载和回调的方式，这跟 CommonJS 是截然不同的。

### UMD

对于需要同时支持 AMD 和 CommonJS 的模块而言，可以使用 UMD（Universal Module Definition）。

```js
(function (root, factory) {
  if (typeof define === 'function' && define.amd) {
      // AMD
    define(['myModule', 'myOtherModule'], factory);
  } else if (typeof exports === 'object') {
      // CommonJS
    module.exports = factory(require('myModule'), require('myOtherModule'));
  } else {
    // Browser globals (Note: root is window)
    root.returnExports = factory(root.myModule, root.myOtherModule);
  }
}(this, function (myModule, myOtherModule) {
  // Methods
  function notHelloOrGoodbye(){}; // A private method
  function hello(){}; // A public method because it's returned (see below)
  function goodbye(){}; // A public method because it's returned (see below)

  // Exposed public methods
  return {
      hello: hello,
      goodbye: goodbye
  }
}));
```

在执行 UMD 规范时，会优先判断是当前环境是否支持 AMD 环境，然后再检验是否支持 CommonJS 环境，否则认为当前环境为浏览器环境（window）。

如果你写了一个小工具库，你想让它及支持 AMD 规范，又想让他支持 CommonJS 规范，那么采用 UMD 规范对你的代码进行包装吧。

### ES6 模块 (ES2015)

可能你已经注意到了，上面所有这些模型定义，没有一种是 JavaScript 语言原生支持的。无论是 AMD 还是 CommonJS，这些都是 JavaScript 函数来模拟的。

幸运的是，ES6 开始引入了原生的模块功能。

ES6 的原生模块功能非常棒，它兼顾了规范、语法简约性和异步加载功能。它还支持循环依赖。

最棒的是，import 进来的模块对于调用它的模块来是说是一个活的只读视图，而不是像 CommonJS 一样是一个内存的拷贝。

下面是一个 ES6 模块的示例：

```js
// lib/counter.js
export let counter = 1;

export function increment() {
  counter++;
}

export function decrement() {
  counter--;
}

// src/main.js
import * as counter from '../../counter';

console.log(counter.counter); // 1
counter.increment();
console.log(counter.counter); // 2
```

如果只希望导出某个模块的部分属性，或者希望处理命名冲突的问题，可以有这样一些导入方式：

```js
import {detectCats} from "kittydar.js";

//or
import {detectCats, Kittydar} from "kittydar.js";

//or
import {flip as flipOmelet} from "eggs.js";

```

### SystemJs and jspm

JavaScript 的软件包管理器（又名 jspm）是一个软件包管理工具，工作在 systemjs 通用模块加载器之上。它不是一个完全新的包管理工具，而是基于已经存在的包资源进行的，它与 GitHub 和 NPM 协同工作。因为大多数的 bower 安装包 基于 GitHub，我们可以很好的使用 jspm 安装这些软件包。它有一个 注册表 ，列出了最常用的前端模块，使用户方便安装。像 NPM，它可以区分安装包是在开发环境或者生产环境。

systemjs 是模块加载器，可以导入任何流行格式的模块（CommonJS、UMD、AMD、ES6）。它是工作在 [ES6 模块加载 polyfill](https://github.com/ModuleLoader/es-module-loader) 之上，它能够很好的处理和检测所使用的格式。 systemjs 也能使用插件转换 es6（ 用 Babel 或者 Traceur）或者转换 TypeScript 和 CoffeeScript 代码。你只需要在导入你的模块之前使用 System.config({ ... }) 进行系统配置。
