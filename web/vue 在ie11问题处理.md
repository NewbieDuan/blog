### 介绍

Babel默认只转换新的JavaScript句法（syntax），而不转换新的API，比如Iterator、Generator、Set、Maps、Proxy、Reflect、Symbol、Promise等全局对象，以及一些定义在全局对象上的方法（比如Object.assign）都不会转码

在vue项目中如果使用了ES6+相关的API，在IE11就会出现不兼容问题，页面出现白屏、控制台会报堆栈溢出等相关错误，处理兼容问题主要操作是将ES6语法转换成IE11可识别的代码，其中就需要使用polyfill给当前环境提供对应API

### 处理
在官方的vue-cli中有处理方法<https://cli.vuejs.org/zh/guide/browser-compatibility.html>

#### browserslist
package.json 文件里的 browserslist 字段 (或一个单独的 .browserslistrc 文件)，指定了项目的目标浏览器的范围。这个值会被 @babel/preset-env 和 Autoprefixer 用来确定需要转译的 JavaScript 特性和需要添加的 CSS 浏览器前缀

.browserslistrc
~~~ini
> 1%
last 2 versions
not ie < 11
~~~

#### polyfill 
#### babel7.4.0以后的方式，使用的code-js3.0.0以上版本：
如果该依赖交付 ES5 代码，但使用了 ES6+ 特性且没有显式地列出需要的 polyfill (例如 Vuetify)：请使用 useBuiltIns: 'entry' 然后在入口文件添加 import 'core-js/stable'; import 'regenerator-runtime/runtime';。这会根据 browserslist 目标导入所有 polyfill，这样你就不用再担心依赖的 polyfill 问题了，但是因为包含了一些没有用到的 polyfill 所以最终的包大小可能会增加。

babel.config.js
~~~javascript
{
    presets: [
        [
            '@vue/cli-plugin-babel/preset',
            // 配置信息
            {
                // 指定corejs版本
                corejs: '3',
                // 使用corejs的方式 "usage" 表示按需加载
                useBuiltIns: 'entry'
            }
        ]
    ]
};
~~~
#### babel7.4.0之前：

- babel.config.js
~~~javascript
{
  "presets": [
    [
      "@babel/preset-env",
      {
        "useBuiltIns": "entry"
      }
    ]
  ]

~~~

- 安装 '@babel/polyfill' ，并在入口文件添加 import '@babel/polyfill'
~~~
$ npm install --save @babel/polyfill
~~~
- main.js
~~~javascirpt
import '@babel/polyfill'
~~~
