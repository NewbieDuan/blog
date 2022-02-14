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

在 Webpack 4 及之前版本中可以使用一些 loader 自带的缓存功能提升构建性能，例如 babel-loader、eslint-loader、cache-loadermodule.exports = {
 ```javascript
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
  
