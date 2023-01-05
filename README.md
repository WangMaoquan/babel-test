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

新建 `.babelrc.json`,

```json
{
  "presets": [
    [
      "@babel/preset-env",
      {
        // 指定兼容的浏览器版本
        "targets": {
          "ie": "11"
        },
        // 基础库 core-js 的版本，一般指定为最新的大版本
        "corejs": 3,
        /*
          Polyfill 注入策略
          它决定了添加 Polyfill 策略，默认是 false，即不添加任何的 Polyfill
          只有两个 entry / usage

          entry配置规定你必须在入口文件手动添加一行这样的代码
          import 'core-js';
          执行下面代码
          npx babel src --out-dir dist

          usage 按需
          npx babel src --out-dir dist
        */
        "useBuiltIns": "usage",
        // 不将 ES 模块语法转换为其他模块语法
        "modules": false
      }
    ]
  ]
}
```

我们利用`@babel/preset-env` 进行了目标浏览器语法的降级和 Polyfill 注入，同时用到了 `core-js` 和 `regenerator-runtime` 两个核心的运行时库。但@babel/preset-env 的方案也存在一定局限性

- 如果使用新特性，往往是通过基础库(如 core-js)往全局环境添加 Polyfill，如果是开发应用没有任何问题，如果是开发第三方工具库，则很可能会对全局空间造成污染
- 很多工具函数的实现代码(如上面示例中的`_defineProperty` 方法)，会在许多文件中重现出现，造成文件体积冗余

#### 更优的 Polyfill 注入方案: transform-runtime

> 需要提前说明的是，`transform-runtime` 方案可以作为`@babel/preset-env` 中 `useBuiltIns` 配置的替代品，也就是说，一旦使用 transform-runtime 方案，你应该把 `useBuiltIns` 属性设为 `false`

**安装必要的依赖**

```
pnpm i @babel/plugin-transform-runtime -D
pnpm i @babel/runtime-corejs3 -S
```

前者是编译时工具，用来转换语法和添加 Polyfill，后者是运行时基础库，封装了 core-js、regenerator-runtime 和各种语法转换用到的工具函数

> core-js 有三种产物，分别是 `core-js`、`core-js-pure` 和 `core-js-bundle`。第一种是全局 Polyfill 的做法，@babel/preset-env 就是用的这种产物；第二种不会把 Polyfill 注入到全局环境，可以按需引入；第三种是打包好的版本，包含所有的 Polyfill，不太常用。@babel/runtime-corejs3 使用的是第二种产物

接着我们修改 `.babelrc.json`

```json
{
  "plugins": [
    // 添加 transform-runtime 插件
    [
      "@babel/plugin-transform-runtime",
      {
        "corejs": 3
      }
    ]
  ],
  "presets": [
    [
      "@babel/preset-env",
      {
        "targets": {
          "ie": "11"
        },
        "corejs": 3,
        // 关闭 @babel/preset-env 默认的 Polyfill 注入
        "useBuiltIns": false,
        "modules": false
      }
    ]
  ]
}
```

**执行终端命令:**

```
npx babel src --out-dir dist
```

经过对比我们不难发现，`transform-runtime` 一方面能够让我们在代码中使用非全局版本的 `Polyfill`，这样就避免全局空间的污染，这也得益于 core-js 的 pure 版本产物特性；另一方面对于 `asyncToGeneator `这类的工具函数，它也将其转换成了一段引入语句，不再将完整的实现放到文件中，节省了编译后文件的体积。

另外，`transform-runtime` 方案引用的基础库也发生了变化，不再是直接引入 core-js 和 regenerator-runtime，而是引入@babel/runtime-corejs3
