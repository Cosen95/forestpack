## 引言

前一段时间我把`webpack`源码大概读了一遍，`webpack` 到`4.x`版本后，其源码已经比较庞大，对各种开发场景进行了高度抽象，阅读成本也愈发昂贵。

过度分析源码对于大家并没有太大的帮助。本文主要是想通过分析`webpack`的构建流程以及实现一个简单的`webpack`来让大家对`webpack`的内部原理有一个大概的了解。（保证能看懂，不懂你打我 🙈）
![](https://imgkr2.cn-bj.ufileos.com/149a5f7e-08e3-4681-b3fd-76ab4e085761.gif?UCloudPublicKey=TOKEN_8d8b72be-579a-4e83-bfd0-5f6ce1546f13&Signature=0u7YWTEYuEBkSCK6qCpj2O7KbKs%253D&Expires=1596960593)

## webpack 构建流程分析

首先，无须多言，上图～
![](https://imgkr2.cn-bj.ufileos.com/4419ad0f-f458-4254-b1c0-36e235c4b943.png?UCloudPublicKey=TOKEN_8d8b72be-579a-4e83-bfd0-5f6ce1546f13&Signature=ZGCvg6PRQU5HxRltkD5OKLX8Ks4%253D&Expires=1596859272)
`webpack` 的运行流程是一个串行的过程，从启动到结束会依次执行以下流程：首先会从配置文件和 `Shell` 语句中读取与合并参数，并初始化需要使用的插件和配置插件等执行环境所需要的参数；初始化完成后会调用`Compiler`的`run`来真正启动`webpack`编译构建过程，`webpack`的构建流程包括`compile`、`make`、`build`、`seal`、`emit`阶段，执行完这些阶段就完成了构建过程。

### 初始化

#### entry-options 启动

从配置文件和 `Shell` 语句中读取与合并参数，得出最终的参数。

#### run 实例化

`compiler`：用上一步得到的参数初始化 `Compiler` 对象，加载所有配置的插件，执行对象的 `run` 方法开始执行编译

### 编译构建

#### `entry 确定入口`

根据配置中的 `entry` 找出所有的入口文件

#### `make 编译模块`

从入口文件出发，调用所有配置的 `Loader` 对模块进行翻译，再找出该模块依赖的模块，再递归本步骤直到所有入口依赖的文件都经过了本步骤的处理

#### `build module 完成模块编译`

经过上面一步使用 `Loader` 翻译完所有模块后，得到了每个模块被翻译后的最终内容以及它们之间的依赖关系

#### `seal 输出资源`

根据入口和模块之间的依赖关系，组装成一个个包含多个模块的 `Chunk`，再把每个 `Chunk` 转换成一个单独的文件加入到输出列表，这步是可以修改输出内容的最后机会

#### `emit 输出完成`

在确定好输出内容后，根据配置确定输出的路径和文件名，把文件内容写入到文件系统

分析完构建流程，下面让我们自己动手实现一个简易的`webpack`吧～
![](https://imgkr2.cn-bj.ufileos.com/b36095fe-974b-4ee7-8d4b-52050dfb915b.jpg?UCloudPublicKey=TOKEN_8d8b72be-579a-4e83-bfd0-5f6ce1546f13&Signature=4PYKnhpZNQ%252BFv%252Ft6hLlI0f5iex4%253D&Expires=1596960846)

## 实现一个简易的 webpack

### 准备工作

#### 目录结构

我们先来初始化一个项目，结构如下：

```
|-- forestpack
    |-- dist
    |   |-- bundle.js
    |   |-- index.html
    |-- lib
    |   |-- compiler.js
    |   |-- index.js
    |   |-- parser.js
    |   |-- test.js
    |-- src
    |   |-- greeting.js
    |   |-- index.js
    |-- forstpack.config.js
    |-- package.json
```

这里我先解释下每个文件/文件夹对应的含义：

- `dist`：打包目录
- `lib`：核心文件，主要包括`compiler`和`parser`
  - `compiler.js`：编译相关。`Compiler`为一个类, 并且有`run`方法去开启编译，还有构建`module`（`buildModule`）和输出文件（`emitFiles`）
  - `parser.js`：解析相关。包含解析`AST`（`getAST`）、收集依赖（`getDependencies`）、转换（`es6转es5`）
  - `index.js`：实例化`Compiler`类，并将配置参数（对应`forstpack.config.js`）传入
  - `test.js`：测试文件，用于测试方法函数打`console`使用
- `src`：源代码。也就对应我们的业务代码
- `forstpack.config.js`： 配置文件。类似`webpack.config.js`
- `package.json`：这个就不用我多说了～～～（什么，你不知道？？）

#### 先完成“造轮子”前 30%的代码

项目搞起来了，但似乎还少点东西～～
![](https://imgkr2.cn-bj.ufileos.com/7d3953fe-2c50-45cd-a123-97c2c3bd91b8.jpg?UCloudPublicKey=TOKEN_8d8b72be-579a-4e83-bfd0-5f6ce1546f13&Signature=JbOSS4fcLN3DUD5mlcji%252BEoF6No%253D&Expires=1596961059)

对了！基础的文件我们需要先完善下：`forstpack.config.js`和`src`。

首先是`forstpack.config.js`：

```js
const path = require("path");

module.exports = {
  entry: path.join(__dirname, "./src/index.js"),
  output: {
    path: path.join(__dirname, "./dist"),
    filename: "bundle.js",
  },
};
```

内容很简单，定义一下入口、出口（你这也太简单了吧！！别急，慢慢来嘛）
![](https://imgkr2.cn-bj.ufileos.com/af4c4c38-4eb7-4482-888a-fa1f664a0e6d.jpg?UCloudPublicKey=TOKEN_8d8b72be-579a-4e83-bfd0-5f6ce1546f13&Signature=r6JWZqps0uGSQy0D8kYHI3%252FwWVA%253D&Expires=1596961198)

其次是`src`，这里在`src`目录下定义了两个文件：

- `greeting.js`：

```js
// greeting.js
export function greeting(name) {
  return "你好" + name;
}
```

- `index.js`：

```js
import { greeting } from "./greeting.js";

document.write(greeting("森林"));
```

ok，到这里我们已经把需要准备的工作都完成了。（问：为什么这么基础？答：当然要基础了，我们的核心是“造轮子”！！）
![](https://imgkr2.cn-bj.ufileos.com/3228f76d-f998-41cd-aff5-2898dabba038.jpg?UCloudPublicKey=TOKEN_8d8b72be-579a-4e83-bfd0-5f6ce1546f13&Signature=RqmNFby94ZRQ4Df%252BVwXeG%252Bh2XX8%253D&Expires=1596961329)

### 梳理下逻辑

短暂的停留一下，我们梳理下逻辑：

`Q`: 我们要做什么？

`A`: 做一个比`webpack`更强的`super webpack`（不好意思，失态了，一不小心说出了我的心声）。还是低调点（防止一会被疯狂打脸）
![](https://imgkr2.cn-bj.ufileos.com/714a8dad-84d9-4ffd-8b5a-f802faa38e1f.jpg?UCloudPublicKey=TOKEN_8d8b72be-579a-4e83-bfd0-5f6ce1546f13&Signature=sfQTwVdH5tCuGRLLmfBdAi6MrO8%253D&Expires=1596961382)

`Q`: 怎么去做？

`A`: 看下文（23333）

`Q`: 整个的流程是什么？

`A`: 哎嘿，大概流程就是：

- 读取入口文件
- 分析入口文件，递归的去读取模块所依赖的文件内容，生成`AST`语法树。
- 根据`AST`语法树，生成浏览器能够运行的代码

### 正式开工

#### compile.js 编写

```js
const path = require("path");
const fs = require("fs");

module.exports = class Compiler {
  // 接收通过lib/index.js new Compiler(options).run()传入的参数，对应`forestpack.config.js`的配置
  constructor(options) {
    const { entry, output } = options;
    this.entry = entry;
    this.output = output;
    this.modules = [];
  }
  // 开启编译
  run() {}
  // 构建模块相关
  buildModule(filename, isEntry) {
    // filename: 文件名称
    // isEntry: 是否是入口文件
  }
  // 输出文件
  emitFiles() {}
};
```

`compile.js`主要做了几个事情：

- 接收`forestpack.config.js`配置参数，并初始化`entry`、`output`
- 开启编译`run`方法。处理构建模块、收集依赖、输出文件等。
- `buildModule`方法。主要用于构建模块（被`run`方法调用）
- `emitFiles`方法。输出文件（同样被`run`方法调用）

到这里，`compiler.js`的大致结构已经出来了，但是得到模块的源码后, 需要去解析,替换源码和获取模块的依赖项, 也就对应我们下面需要完善的`parser.js`。

#### parser.js 编写

```js
const fs = require("fs");
// const babylon = require("babylon");
const parser = require("@babel/parser");
const traverse = require("@babel/traverse").default;
const { transformFromAst } = require("babel-core");
module.exports = {
  // 解析我们的代码生成AST抽象语法树
  getAST: (path) => {
    const source = fs.readFileSync(path, "utf-8");

    return parser.parse(source, {
      sourceType: "module", //表示我们要解析的是ES模块
    });
  },
  // 对AST节点进行递归遍历
  getDependencies: (ast) => {
    const dependencies = [];
    traverse(ast, {
      ImportDeclaration: ({ node }) => {
        dependencies.push(node.source.value);
      },
    });
    return dependencies;
  },
  // 将获得的ES6的AST转化成ES5
  transform: (ast) => {
    const { code } = transformFromAst(ast, null, {
      presets: ["env"],
    });
    return code;
  },
};
```

看完这代码是不是有点懵（说好的保证让看懂的 😤）

别着急，你听我辩解！！😷
![](https://imgkr2.cn-bj.ufileos.com/4cc98b74-bf46-43ed-bddf-b5f3e502ff80.jpg?UCloudPublicKey=TOKEN_8d8b72be-579a-4e83-bfd0-5f6ce1546f13&Signature=1Rs%252B1EAGEiLqEDsPVAjNVwrA8Tg%253D&Expires=1596961508)

这里要先着重说下用到的几个`babel`包：

- `@babel/parser`：用于将源码生成`AST`
- `@babel/traverse`：对`AST`节点进行递归遍历
- `babel-core`/`@babel/preset-env`：将获得的`ES6`的`AST`转化成`ES5`

`parser.js`中主要就三个方法：

- `getAST`： 将获取到的模块内容 解析成`AST`语法树
- `getDependencies`：遍历`AST`，将用到的依赖收集起来
- `transform`：把获得的`ES6`的`AST`转化成`ES5`

#### 完善 compiler.js

在上面我们已经将`compiler.js`中会用到的函数占好位置，下面我们需要完善一下`compiler.js`，当然会用到`parser.js`中的一些方法（废话，不然我上面干嘛要先把`parser.js`写完～～）
![](https://imgkr2.cn-bj.ufileos.com/5e9cd4de-d2c7-4879-be73-0409b317a889.jpg?UCloudPublicKey=TOKEN_8d8b72be-579a-4e83-bfd0-5f6ce1546f13&Signature=smL4rdl8wg5tgISpbEjIRF1wHRA%253D&Expires=1596961635)

直接上代码：

```js
const { getAST, getDependencies, transform } = require("./parser");
const path = require("path");
const fs = require("fs");

module.exports = class Compiler {
  constructor(options) {
    const { entry, output } = options;
    this.entry = entry;
    this.output = output;
    this.modules = [];
  }
  // 开启编译
  run() {
    const entryModule = this.buildModule(this.entry, true);
    this.modules.push(entryModule);
    this.modules.map((_module) => {
      _module.dependencies.map((dependency) => {
        this.modules.push(this.buildModule(dependency));
      });
    });
    // console.log(this.modules);
    this.emitFiles();
  }
  // 构建模块相关
  buildModule(filename, isEntry) {
    let ast;
    if (isEntry) {
      ast = getAST(filename);
    } else {
      const absolutePath = path.join(process.cwd(), "./src", filename);
      ast = getAST(absolutePath);
    }

    return {
      filename, // 文件名称
      dependencies: getDependencies(ast), // 依赖列表
      transformCode: transform(ast), // 转化后的代码
    };
  }
  // 输出文件
  emitFiles() {
    const outputPath = path.join(this.output.path, this.output.filename);
    let modules = "";
    this.modules.map((_module) => {
      modules += `'${_module.filename}' : function(require, module, exports) {${_module.transformCode}},`;
    });

    const bundle = `
        (function(modules) {
          function require(fileName) {
            const fn = modules[fileName];
            const module = { exports:{}};
            fn(require, module, module.exports)
            return module.exports
          }
          require('${this.entry}')
        })({${modules}})
    `;

    fs.writeFileSync(outputPath, bundle, "utf-8");
  }
};
```

关于`compiler.js`的内部函数，上面我说过一遍，这里主要来看下`emitFiles`：

```js
emitFiles() {
    const outputPath = path.join(this.output.path, this.output.filename);
    let modules = "";
    this.modules.map((_module) => {
      modules += `'${_module.filename}' : function(require, module, exports) {${_module.transformCode}},`;
    });

    const bundle = `
        (function(modules) {
          function require(fileName) {
            const fn = modules[fileName];
            const module = { exports:{}};
            fn(require, module, module.exports)
            return module.exports
          }
          require('${this.entry}')
        })({${modules}})
    `;

    fs.writeFileSync(outputPath, bundle, "utf-8");
  }

```

这里的`bundle`一大坨，什么鬼？
![](https://imgkr2.cn-bj.ufileos.com/e2179db1-a23a-4cf7-9ba5-85591f71d3ff.gif?UCloudPublicKey=TOKEN_8d8b72be-579a-4e83-bfd0-5f6ce1546f13&Signature=t9aakeP7WQW13NVu56KXCwKZm0g%253D&Expires=1596961716)

我们先来了解下`webpack`的文件 📦 机制。下面一段代码是经过`webpack`打包精简过后的代码：

```js
// dist/index.xxxx.js
(function(modules) {
  // 已经加载过的模块
  var installedModules = {};

  // 模块加载函数
  function __webpack_require__(moduleId) {
    if(installedModules[moduleId]) {
      return installedModules[moduleId].exports;
    }
    var module = installedModules[moduleId] = {
      i: moduleId,
      l: false,
      exports: {}
    };
    modules[moduleId].call(module.exports, module, module.exports, __webpack_require__);
    module.l = true;
    return module.exports;
  }
  __webpack_require__(0);
})([
/* 0 module */
(function(module, exports, __webpack_require__) {
  ...
}),
/* 1 module */
(function(module, exports, __webpack_require__) {
  ...
}),
/* n module */
(function(module, exports, __webpack_require__) {
  ...
})]);

```

简单分析下：

- `webpack` 将所有模块(可以简单理解成文件)包裹于一个函数中，并传入默认参数，将所有模块放入一个数组中，取名为 `modules`，并通过数组的下标来作为 `moduleId`。
- 将 `modules` 传入一个自执行函数中，自执行函数中包含一个 `installedModules` 已经加载过的模块和一个模块加载函数，最后加载入口模块并返回。
- `__webpack_require__` 模块加载，先判断 `installedModules` 是否已加载，加载过了就直接返回 `exports` 数据，没有加载过该模块就通过 `modules[moduleId].call(module.exports, module, module.exports, __webpack_require__)` 执行模块并且将 `module.exports` 给返回。

（你上面说的这一坨又是什么鬼？我听不懂啊啊啊啊！！！）
![](https://imgkr2.cn-bj.ufileos.com/1e348390-d96b-4862-99e3-679efcb97fd4.jpg?UCloudPublicKey=TOKEN_8d8b72be-579a-4e83-bfd0-5f6ce1546f13&Signature=s8wu4JUN8sqvDBN54m76tQdRLrI%253D&Expires=1596961813)

那我换个说法吧：

- 经过`webpack`打包出来的是一个匿名闭包函数（`IIFE`）
- `modules`是一个数组，每一项是一个模块初始化函数
- `__webpack_require__`用来加载模块，返回`module.exports`
- 通过`WEBPACK_REQUIRE_METHOD(0)`启动程序

（小声 bb：怎么样，这样听懂了吧）
![](https://imgkr2.cn-bj.ufileos.com/f9c56a67-bdcb-4952-b3dd-d29a13cd0164.jpg?UCloudPublicKey=TOKEN_8d8b72be-579a-4e83-bfd0-5f6ce1546f13&Signature=A7d2V1LJ7yBJ7hq%252BA3LAPwROsJQ%253D&Expires=1596961909)

#### lib/index.js 入口文件编写

到这里，就剩最后一步了（似乎见到了胜利的曙光）。在`lib`目录创建`index.js`：

```js
const Compiler = require("./compiler");
const options = require("../forestpack.config");

new Compiler(options).run();
```

这里逻辑就比较简单了：实例化`Compiler`类，并将配置参数（对应`forstpack.config.js`）传入。

运行`node lib/index.js`就会在`dist`目录下生成`bundle.js`文件。

```js
(function (modules) {
  function require(fileName) {
    const fn = modules[fileName];
    const module = { exports: {} };
    fn(require, module, module.exports);
    return module.exports;
  }
  require("/Users/fengshuan/Desktop/workspace/forestpack/src/index.js");
})({
  "/Users/fengshuan/Desktop/workspace/forestpack/src/index.js": function (
    require,
    module,
    exports
  ) {
    "use strict";

    var _greeting = require("./greeting.js");

    document.write((0, _greeting.greeting)("森林"));
  },
  "./greeting.js": function (require, module, exports) {
    "use strict";

    Object.defineProperty(exports, "__esModule", {
      value: true,
    });
    exports.greeting = greeting;

    function greeting(name) {
      return "你好" + name;
    }
  },
});
```

和上面用`webpack`打包生成的`js`文件作下对比，是不是很相似呢？
![](https://imgkr2.cn-bj.ufileos.com/0548df18-4bec-4300-ac02-8a5da66e5ffb.gif?UCloudPublicKey=TOKEN_8d8b72be-579a-4e83-bfd0-5f6ce1546f13&Signature=Oktqpkw6UHmUQktu2y6Ugn1aEP4%253D&Expires=1596961993)

#### 来吧！展示

我们在`dist`目录下创建`index.html`文件，引入打包生成的`bundle.js`文件：

```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>Document</title>
  </head>
  <body>
    <script src="./bundle.js"></script>
  </body>
</html>
```

此时打开浏览器：
![](https://imgkr2.cn-bj.ufileos.com/02327ea5-8da7-4669-b213-e14e077b6268.png?UCloudPublicKey=TOKEN_8d8b72be-579a-4e83-bfd0-5f6ce1546f13&Signature=v%252BXAiZ1N%252B2%252BoLd1D1XbBdSjLq5s%253D&Expires=1596960289)

如你所愿，得到了我们预期的结果～
![](https://imgkr2.cn-bj.ufileos.com/765940eb-2fca-45ba-9745-8d9afedee303.jpg?UCloudPublicKey=TOKEN_8d8b72be-579a-4e83-bfd0-5f6ce1546f13&Signature=ohXD2zBJRbXlX4PNE9EkUyLqJas%253D&Expires=1596962082)

## 总结

通过对`webpack`构建流程的分析以及实现了一个简易的`forestpack`，相信你对`webpack`的构建原理已经有了一个清晰的认知！（当然，这里的`forestpack`和`webpack`相比还很弱很弱，，，，）
![](https://imgkr2.cn-bj.ufileos.com/f8177cf8-8b12-4271-b4ea-1ccd5ada1b1b.gif?UCloudPublicKey=TOKEN_8d8b72be-579a-4e83-bfd0-5f6ce1546f13&Signature=WScIbtHRFyq6m%252BWqPTdL8twFOiA%253D&Expires=1596962480)

## 参考

本文是看过极客时间程柳锋老师的「玩转 webpack」课程后整理的。这里也十分推荐大家去学习这门课程～
![](https://imgkr2.cn-bj.ufileos.com/3263c8a5-319e-4c4e-a263-45fe1cfb4709.jpg?UCloudPublicKey=TOKEN_8d8b72be-579a-4e83-bfd0-5f6ce1546f13&Signature=H7d8OTXqEuaHg7txrT4UIr4Gq9g%253D&Expires=1596962725)

## ❤️ 爱心三连击

1.如果觉得这篇文章还不错，来个<b style="color: #ff6441">分享、点赞、在看</b>三连吧，让更多的人也看到～

2.关注公众号<b style="color: #ff6441">前端森林</b>，定期为你推送新鲜干货好文。

3.特殊阶段，带好口罩，做好个人防护。

4.添加微信<b style="color: #ff6441">fs1263215592</b>，拉你进技术交流群一起学习 🍻
![](https://imgkr2.cn-bj.ufileos.com/40ac6fd2-89cf-4fc6-b154-3c3fd4956f2d.png?UCloudPublicKey=TOKEN_8d8b72be-579a-4e83-bfd0-5f6ce1546f13&Signature=3bxgYcPfdxDGcGIskOHykGXHqOo%253D&Expires=1596962818)
