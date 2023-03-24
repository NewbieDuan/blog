## AMD CMD CommonJS ESModule 总结

| 模块化	| 场景|	特点 |	语法 |
|  ----  | ----  | ----|----|
| CommonJS | node、webpack	| 同步加载、磁盘读取速度快 | 导出：通过module.exports或exports导出 引用：require(' xx ') |
| AMD	| 浏览器端| 	异步加载，依赖前置，预先加载所有的依赖	| 导出：通过define定义模块 引用：require()|
| CMD	| 浏览器端 | 	异步加载，依赖就近，按需加载 | 导出：通过define定义模块 引用：require() |
| ES Module	| 目前浏览器端的默认标准 |	静态编译 | 导出：通过export或export default输出模块 引用：import .. from 'xxx'| 

#### 注：

- requirejs实现了AMD规范，seajs实现了CMD规范
- AMD 推崇依赖前置、提前执行，CMD推崇就近依赖、延迟执行。比如AMD会在申明依赖的第一个参数中列出所有的模块，CMD是在需要该模块的时候再进行require引入，从而实现了按需加载
- Commonjs模块输出的是一个值的拷贝，ES6模块输出的是值的引用
- Commonjs模块是运行时加载，ES6模块是编译时输出接口
- Node verison 13.2.0 起开始正式支持 ES Modules 特性，在 Node 中使用 ESM 有两种方式：1）在 package.json 中，增加 "type": "module" 配置；2）在 .mjs 文件可以直接使用 import 和 export；ES Modules 导入的模块会被预解析，以便在代码运行前导入;
在 CommonJS 中，模块将在运行时解析；
