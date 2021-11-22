# webpack-write

Handwritten implementation of webpack

手写实现一个 webpack

## 一、初始化项目

手写实现 webpack 前，需要安装四个依赖包：

- `@babel/parser`：用于分析通过 *fs.readFileSync* 读取的文件内容，并返回 AST (抽象语法树)；
- `@babel/traverse`：用于遍历 AST, 获取必要的数据；
- `@babel/core`：babel 核心模块，提供 *transformFromAst* 方法，用于将 AST 转化为浏览器可运行的代码；
- `@babel/preset-env`：将转换后代码转化成 ES5 代码；

初始化项目：

```bash
yarn init -y
yarn add @babel/parser @babel/traverse @babel/core @babel/preset-env
```

初始化文件：

```diff
  write_webpack
  |- node_modules
+ |- /src
+   |- consts.js
+   |- index.js
+   |- info.js
+ |- write_webpack.js
  |- package.json
```

接下里，需要实现以下三个核心方法：

- `createAssets`：收集和处理文件的代码；
- `createGraph`：根据入口文件，返回所有文件依赖图；
- `bundle`：根据依赖图整个代码并输出。

## 二、实现 createAssets 函数

### 1、读取入口文件，并转为 AST

```js
// src/index.js

import info from "./info.js";
console.log(info);
```

实现 `createAssets` 方法的*文件读取*和 *AST 转换*操作：

```js
// write_webpack.js

const path = require('path')
const fs = require('fs')
const parser = require('@babel/parser')
// 由于 traverse 采用的 ES Module 导出，通过 require 引入的话就加个 .default
const traverse = require('@babel/traverse').default
const babel = require('@babel/core')

// 为了便于区分当前操作的模块，这里声明了一个 moduleId 变量来表示；
// 在这将读取到的文件流 buffer 转为 AST 的同时，也是将 ES6 代码转换为 ES5 代码了。
let moduleId = 0

// 实现 createAssets 方法中的 文件读取 和 AST转换 操作
const createAssets = (filename) => {
  // 根据文件名，同步读取文件流
  const content = fs.readFileSync(filename, 'utf-8')

  // 将读取文件流 buffer 转换为 AST
  const ast = parser.parse(content, {
    sourceType: 'module' // 指定源码类型
  })

  console.log(ast)
}

createAssets('./src/index.js')
```

上面通过 `fs.readFileSync()` 方法，以同步方式读取指定路径下的文件流，并通过 *parser* 包的 `parse()` 方法，将读取到的文件流 buffer 转换为浏览器可以认识的代码（AST），运行 *write_webpack.js* 文件，AST 输出如下：

```js
// 命令行运行：node write_webpack.js，结果如下：
Node {
  type: 'File',
  start: 0,
  end: 48,
  loc: SourceLocation {
    start: Position { line: 1, column: 0 },
    end: Position { line: 4, column: 0 },
    filename: undefined,
    identifierName: undefined
  },
  errors: [],
  program: Node {
    type: 'Program',
    start: 0,
    end: 48,
    loc: SourceLocation {
      start: [Position],
      end: [Position],
      filename: undefined,
      identifierName: undefined
    },
    sourceType: 'module',
    interpreter: null,
    body: [ [Node], [Node] ],
    directives: []
  },
  comments: []
}
```

### 2、收集每个模块的依赖

接下来声明 *dependencies* 变量来保存收集到的文件依赖路径：通过 `traverse` 方法遍历 AST，获取每个节点依赖的路径，并 push 进 *dependencies* 数组中。

```js
// write_webpack.js

function createAssets(filename) {
  // ...
  const dependencies = [] // 用于收集文件依赖的路径

  // 通过 traverse 提供的操作 AST 的方法，获取每个节点的依赖路径
  traverse(ast, {
    ImportDeclaration: ({ node }) => {
      dependencies.push(node.source.value)
    }
  })
}
```

### 3、将 AST 转为浏览器可运行代码

在收集依赖的同时，可以将 AST 代码转为浏览器可运行代码，这就需要用到 babel 提供的 `transformFromAstSync()` 方法，同步的将 AST 转换为浏览器可运行代码：

```js
// write_webpack.js

function createAssets(filename) {
  // ...
  const { code } = babel.transformFromAstSync(ast, null, {
    presets: ['@babel/preset-env']
  })

  // 设置当前处理的模块 ID
  let id = moduleId++

  return {
    id,
    filename,
    code,
    dependencies
  }
}

console.log(createAssets('./src/index.js'))
```

到这一步，再执行 *node write_webpack.js*，输出以下内容，包含入口文件的路径 *filename*、浏览器可执行代码 *code* 和文件依赖的路径 *dependencies* 数组：

```js
// 命令行运行：node write_webpack.js，结果如下：
{
  id: 0,
  filename: './src/index.js',
  code: '"use strict";\n' +
    '\n' +
    'var _info = _interopRequireDefault(require("./info.js"));\n' +
    '\n' +
    'function _interopRequireDefault(obj) { return obj && obj.__esModule ? obj : { "default": obj }; }\n' +
    '\n' +
    'console.log(_info["default"]);',
  dependencies: [ './info.js' ]
}
```

### 4、代码小结

```js
// write_webpack.js

const path = require('path')
const fs = require('fs')
const parser = require('@babel/parser')
// 由于 traverse 采用的 ES Module 导出，通过 require 引入的话就加个 .default
const traverse = require('@babel/traverse').default
const babel = require('@babel/core')

let moduleId = 0

// 实现 createAssets 方法中的 文件读取 和 AST转换 操作
const createAssets = (filename) => {
  // 根据文件名，同步读取文件流
  const content = fs.readFileSync(filename, 'utf-8')

  // 将读取文件流 buffer 转换为 AST
  const ast = parser.parse(content, {
    sourceType: 'module' // 指定源码类型
  })

  const dependencies = [] // 用于收集文件依赖的路径

  // 通过 traverse 提供的操作 AST 的方法，获取每个节点的依赖路径
  traverse(ast, {
    ImportDeclaration: ({ node }) => {
      dependencies.push(node.source.value)
    }
  })

  // 通过 AST 将 ES6 代码转换成 ES5 代码
  const { code } = babel.transformFromAstSync(ast, null, {
    presets: ['@babel/preset-env']
  })

  // 设置当前处理的模块 ID
  let id = moduleId++

  return {
    id,
    filename,
    code,
    dependencies
  }
}

console.log(createAssets('./src/index.js'))
```

## 三、实现 createGraph 函数

在 `createGraph()` 函数中，将递归所有依赖模块，循环分析每个依赖模块的依赖，生成一份依赖图谱。

为了方便测试，先补充下 *consts.js* 和 *info.js* 文件的代码，增加一些依赖关系：

```js
// src/consts.js
export const company = 'Riot'

// src/info.js
import { company } from './consts.js'
export default `Hello, ${company}`
```

接下来开始实现 `createGraph()` 函数，它需要接收一个入口文件的路径 *entry* 作为参数：

```js
// write_webpack.js

function createGraph(entry) {
  // 获取入口文件下的内容
  const mainAsset = createAssets(entry)

  // 入口文件的结果作为第一项
  const queue = [mainAsset]

  for (const asset of queue) {
    const dirname = path.dirname(asset.filename)
    asset.mapping = {}
    asset.dependencies.forEach((relativePath) => {
      // 转换文件路径为绝对路径
      const absolutePath = path.join(dirname, relativePath)

      const child = createAssets(absolutePath)

      // 保存模块ID
      asset.mapping[relativePath] = child.id

      // 递归去遍历所有子节点的文件
      queue.push(child)
    })
  }
  return queue
}
```

上面代码中，先通过 `createAssets()` 函数读取入口文件下的内容，并作为依赖关系的队列 *queue* 数组的第一项，接着遍历队列 *queue* 每一项，再遍历将每一项中的依赖 *dependencies* 依赖数组，也是调用 `createAssets()` 函数，递归去遍历所有子节点的文件，并将结果都保存在队列 *queue* 中。

需要注意的是，*mapping* 是用来保存文件的相对路径和模块 ID 的对应关系，在 *mapping* 中，使用依赖文件的相对路径作为 *key*，id 作为 *value* 保存模块 ID。

接下里在 *write_webpack.js* 中修改启动函数：

```diff
- console.log(const result = createAssets('./src/index.js'))
+ const graph = createGraph("./src/index.js")
+ console.log(graph)
```

同理，运行 *write_webpack.js*，可以获取到一份包含所有文件依赖关系的依赖图谱：

```js
// 命令行运行：node write_webpack.js，结果如下：
[
  {
    id: 0,
    filename: './src/index.js',
    code: '"use strict";\n' +
      '\n' +
      'var _info = _interopRequireDefault(require("./info.js"));\n' +
      '\n' +
      'function _interopRequireDefault(obj) { return obj && obj.__esModule ? obj : { "default": obj }; }\n' +
      '\n' +
      'console.log(_info["default"]);',
    dependencies: [ './info.js' ],
    mapping: { './info.js': 1 }
  },
  {
    id: 1,
    filename: 'src/info.js',
    code: '"use strict";',
    dependencies: [],
    mapping: {}
  }
]
```

到这一步，已经获取了所有文件的依赖以及依赖的代码内容。下一步只要实现 `bundle()` 函数，将结果输出即可。

## 四、实现 bundle 函数

在 `bundle()` 函数中，接收一个依赖图谱 *graph* 作为参数，最后输出编译后的结果。

从 `createGraph()` 函数中可以知道，它会返回一个队列 *queue*，包含每个依赖的相关信息*（id / filename / code / dependencies）*

### 1、读取所有模块信息

首先声明一个变量 *modules*，值为字符串类型，用来将前面 *graph* 每一项处理成字符串。对参数 *graph* 进行遍历，将每一项中的 id 取出，作为 key，值为一个数组，包括执行方法和每一项的 *mapping*：

```js
// write_webpack.js

function bundle(graph) {
  let modules = ''
  graph.forEach((item) => {
    modules += `
      ${item.id}: [
        function (require, module, exports){
          ${item.code}
        },
        ${JSON.stringify(item.mapping)}
      ],
    `
  })
}
```

上面代码在 *modules* 中每一项的值中，下标为 0 的元素是个函数，接收三个参数 `require` / `module` / `exports`，为什么会需要这三个参数呢？

构建工具无法判断是否支持 `require` / `module` / `exports` 这三种模块方法，所以需要自己实现，然后方法内的 *code* 才能正常执行。

### 2、返回最终结果

接下来实现最终 `bundle` 函数返回值的处理：

```js
// write_webpack.js

function bundle(graph) {
  //...
  return `
    (function(modules){
      function require(id){
        const [fn, mapping] = modules[id];
        function localRequire(relativePath){
          return require(mapping[relativePath]);
        }

        const module = {
          exports: {}
        }

        fn(localRequire, module, module.exports);

        return module.exports;
      }
      require(0);
    })({${modules}})
  `
}
```

上面代码最终 `bundle` 函数返回值是一个字符串，内容是一个自执行函数 [IIFE](http://localhost:7777/1632758711648)，其中函数参数是一个对象，key 为 *modules*，value 为前面拼接好的 *modules* 字符串，即 `{modules: modules字符串}`

在这个自执行函数中，实现了 `require` 方法，接收一个 id 作为参数，在方法内部，分别实现了 `localRequire` / `module` / `modules.exports` 三个方法，并作为参数，传到 *modules[id]* 中的 `fn` 方法中。

### 3、代码小结

```js
// write_webpack.js

function bundle(graph) {
  let modules = ''

  graph.forEach((item) => {
    modules += `
      ${item.id}: [
        function (require, module, exports){
          ${item.code}
        },
        ${JSON.stringify(item.mapping)}
      ],
    `
  })

  return `
    (function(modules){
      function require(id){
        const [fn, mapping] = modules[id];
        function localRequire(relativePath){
          return require(mapping[relativePath]);
        }
        const module = {
          exports: {}
        }
        fn(localRequire, module, module.exports);
        return module.exports;
      }
      require(0);
    })({${modules}})
  `
}
```

## 五、执行代码

在 *write_webpack.js* 中修改启动函数：

```diff
- console.log(graph)
+ const result = bundle(graph)
+ console.log(result)
```

然后运行代码：

```js
// 命令行运行：node write_webpack.js，结果如下：
(function(modules){
  function require(id){
    const [fn, mapping] = modules[id];
    function localRequire(relativePath){
      return require(mapping[relativePath]);
    }
    const module = {
      exports: {}
    }
    fn(localRequire, module, module.exports);
    return module.exports;
  }
  require(0);
})({
  0: [
    function (require, module, exports){
      "use strict";

var _info = _interopRequireDefault(require("./info.js"));

function _interopRequireDefault(obj) { return obj && obj.__esModule ? obj : { "default": obj }; }

console.log(_info["default"]);
    },
    {"./info.js":1}
  ],

  1: [
    function (require, module, exports){
      "use strict";
    },
    {}
  ],
})
```

到这里，一个简单的 webpack 构建工具就实现了。
