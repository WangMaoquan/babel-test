### 前景

某些低版本浏览器并没有提供 Promise 语法环境以及对象和数组的各种 API，甚至不支持箭头函数语法，代码直接报错，从而导致线上白屏事故的发生，尤其是需要兼容到 `IE 11、iOS 9 以及 Android 4.4` 的场景中很容易会遇到

旧版浏览器的语法兼容问题主要分两类: 语法降级问题和 `Polyfill` 缺失问题。前者比较好理解，比如某些浏览器不支持箭头函数，我们就需要将其转换为 `function(){}`语法；而对后者来说，`Polyfill` 本身可以翻译为垫片，也就是为浏览器提前注入一些 API 的实现代码

这两类问题本质上是通过前端的编译工具链(如 Babel)及 JS 的基础 Polyfill 库(如 corejs)来解决的，不会跟具体的构建工具所绑定

### 底层工具链

#### 工具概览

解决上述提到的两类语法兼容问题，主要需要用到两方面的工具，分别包括:

- 编译时工具。代表工具有`@babel/preset-env` 和`@babel/plugin-transform-runtime`
- 运行时基础库。代表库包括 `core-js `和 `regenerator-runtime`

编译时工具的作用是在代码编译阶段进行语法降级及添加 polyfill 代码的引用语句

```ts
import 'core-js/modules/es6.set.js';
```

由于这些工具只是编译阶段用到，运行时并不需要，我们需要将其放入 package.json 中的 `devDependencies` 中

而运行时基础库是根据 ESMAScript 官方语言规范提供各种 `Polyfill` 实现代码，主要包括 `core-js` 和 `regenerator-runtime` 两个基础库，不过在 babel 中也会有一些上层的封装，包括

- [@babel/polyfill](https://babeljs.io/docs/en/babel-polyfill)
- [@babel/runtime](https://babeljs.io/docs/en/babel-runtime)
- [@babel/runtime-corejs2](https://babeljs.io/docs/en/babel-runtime-corejs2)
- [@babel/runtime-corejs3](https://babeljs.io/docs/en/babel-runtime-corejs3)

看似各种运行时库眼花缭乱，其实都是 `core-js` 和 `regenerator-runtime` 不同版本的封装罢了(`@babel/runtime` 是个特例，不包含 core-js 的 Polyfill)。这类库是项目运行时必须要使用到的，因此一定要放到 package.json 中的 dependencies 中！

#### 实际使用

**安装依赖**

```
pnpm i @babel/cli @babel/core @babel/preset-env
```

- `@babel/cli`: 为 babel 官方的脚手架工具
- `@babel/core`: babel 核心编译库
- `@babel/preset-env`: babel 的预设工具集，基本为 babel 必装的库
