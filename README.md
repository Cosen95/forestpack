## 引言

前一段时间我把`webpack`源码大概读了一遍，`webpack` 到`4.x`版本后，其源码已经比较庞大，对各种开发场景进行了高度抽象，阅读成本也愈发昂贵。

过度分析源码对于大家并没有太大的帮助。本文主要是想通过分析`webpack`的构建流程以及实现一个简单的`webpack`来让大家对`webpack`的内部原理有一个大概的了解。（保证能看懂，不懂你打我 🙈）
![表情01](https://user-images.githubusercontent.com/26785201/89748662-db0b9200-daf6-11ea-97f7-2362b63579fd.gif)

## webpack 构建流程分析

首先，无须多言，上图～
![webpack流程图](https://user-images.githubusercontent.com/26785201/89747816-fe344280-daf2-11ea-820a-6a1a99e34f14.png)

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
![表情02](https://user-images.githubusercontent.com/26785201/89748678-f1195280-daf6-11ea-82b5-85d62402d6c6.jpg)

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
![表情03](https://user-images.githubusercontent.com/26785201/89748688-fd051480-daf6-11ea-84a7-7d9a2fa50462.jpg)

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
![表情04](https://user-images.githubusercontent.com/26785201/89748693-068e7c80-daf7-11ea-8e8e-f7bde2f5e214.jpg)

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
![表情05](https://user-images.githubusercontent.com/26785201/89748699-11491180-daf7-11ea-85b1-697b1df02905.jpg)

### 梳理下逻辑

短暂的停留一下，我们梳理下逻辑：

`Q`: 我们要做什么？

`A`: 做一个比`webpack`更强的`super webpack`（不好意思，失态了，一不小心说出了我的心声）。还是低调点（防止一会被疯狂打脸）
![表情06](https://user-images.githubusercontent.com/26785201/89748705-1c9c3d00-daf7-11ea-923a-b25d9e0465ea.jpg)

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
![表情07](https://user-images.githubusercontent.com/26785201/89748716-29209580-daf7-11ea-9dc4-4f00c6c4eeda.jpg)

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
![表情08](https://user-images.githubusercontent.com/26785201/89748721-32a9fd80-daf7-11ea-9a1e-82a643ed6be5.jpg)

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
![表情09](https://user-images.githubusercontent.com/26785201/89748761-6edd5e00-daf7-11ea-8e41-fc87dcf3905c.gif)

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
![表情10](https://user-images.githubusercontent.com/26785201/89748757-65ec8c80-daf7-11ea-88a2-d224579ed054.jpg)

那我换个说法吧：

- 经过`webpack`打包出来的是一个匿名闭包函数（`IIFE`）
- `modules`是一个数组，每一项是一个模块初始化函数
- `__webpack_require__`用来加载模块，返回`module.exports`
- 通过`WEBPACK_REQUIRE_METHOD(0)`启动程序

（小声 bb：怎么样，这样听懂了吧）
![表情11](https://user-images.githubusercontent.com/26785201/89748754-5e2ce800-daf7-11ea-9911-5ea8d9c07293.jpg)

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
![表情12](https://user-images.githubusercontent.com/26785201/89748803-a815ce00-daf7-11ea-9ff7-d818f96fe3ad.gif)

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
<img width="434" alt="forestpack演示" src="https://user-images.githubusercontent.com/26785201/89748849-e57a5b80-daf7-11ea-8358-6d8960d14d2e.png">

如你所愿，得到了我们预期的结果～
![表情13](https://user-images.githubusercontent.com/26785201/89748832-d0053180-daf7-11ea-8adb-76ec8fd5aa8d.jpg)

## 总结

通过对`webpack`构建流程的分析以及实现了一个简易的`forestpack`，相信你对`webpack`的构建原理已经有了一个清晰的认知！（当然，这里的`forestpack`和`webpack`相比还很弱很弱，，，，）
![表情14](https://user-images.githubusercontent.com/26785201/89748837-d7c4d600-daf7-11ea-9b2d-cf6c0e93198f.gif)

## 参考

本文是看过极客时间程柳锋老师的「玩转 webpack」课程后整理的。这里也十分推荐大家去学习这门课程～
![表情15](https://user-images.githubusercontent.com/26785201/89748843-deebe400-daf7-11ea-9c44-da39d1dfa8cf.jpg)

## ❤️ 爱心三连击

1.如果觉得这篇文章还不错，来个<b style="color: #ff6441">分享、点赞、在看</b>三连吧，让更多的人也看到～

2.关注公众号<b style="color: #ff6441">前端森林</b>，定期为你推送新鲜干货好文。

3.特殊阶段，带好口罩，做好个人防护。

4.添加微信<b style="color: #ff6441">fs1263215592</b>，拉你进技术交流群一起学习 🍻
![前端森林公众号二维码2](https://user-images.githubusercontent.com/26785201/89748866-faef8580-daf7-11ea-8fef-d4d62e0eaf6e.png)
