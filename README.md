# webpack-write

Handwritten implementation of webpack

手写实现一个 webpack，实现以下三个核心方法：

- `createAssets`：收集和处理文件的代码；
- `createGraph`：根据入口文件，返回所有文件依赖图；
- `bundle`：根据依赖图整个代码并输出。
