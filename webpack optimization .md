### 1.tree shaking

- webpack5默认进行tree shaking
- 使用支持 Tree Shaking 的包,例如：使用 lodash-es 替代 lodash ，或者使用 babel-plugin-lodash 实现类似效果

### 2.SplitChunkPlugin

### 3.使用 Cache 提升构建性能
```javascript
  module.exports = {
    //...
    cache: {
        type: 'filesystem'
    },
    //...
  };
```
- cache.type：缓存类型，支持 'memory' | 'filesystem'，需要设置 filesystem 才能开启持久缓存
- cache.cacheDirectory：缓存文件存放的路径，默认为 node_modules/.cache/webpack
- cache.buildDependencies：额外的依赖文件，当这些文件内容发生变化时，缓存会完全失效而执行完整的编译构建，通常可设置为项目配置文件

在 Webpack 4 及之前版本中可以使用一些 loader 自带的缓存功能提升构建性能，例如 babel-loader、eslint-loader、cache-loader
```javascript
module.exports = {
    //babel-loader、eslint-loader
    // ...
    module: {
        rules: [{
            test: /\.m?js$/,
            loader: 'babel-loader',
            options: {
                cacheDirectory: true,
            },
        }]
    },
    // ...
    
    //cache-loader
    const path = require("path");
    const webpack = require("webpack");

    module.exports = {
        // ...
        module: {
            rules: [{
                test: /\.js$/,
                use: ['cache-loader', 'babel-loader', 'eslint-loader']
            }]
        },
        // ...
    };
  ```
  
 ### 4.多进程打包
- HappyPack：多进程方式运行资源加载逻辑(https://github.com/amireh/happypack)，（不在维护）
- Thread-loader：Webpack 官方出品，同样以多进程方式运行资源加载逻辑
- TerserWebpackPlugin：支持多进程方式执行代码压缩、uglify 功能
- Parallel-Webpack：多进程方式运行多个 Webpack 构建实例

### 其他
- 配置 resolve 控制资源搜索范围
- 针对 npm 包设置 module.noParse 跳过编译步骤
- 配置 module.rules.exclude 或 module.rules.include 降低 Loader 工作量
- 配置 watchOption.ignored 减少监听文件数量
- 优化 ts 类型检查逻辑
- 慎重选择 source-map 值  
