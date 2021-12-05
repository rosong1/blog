# @vite/plugin-react 1.1.0

看看该插件的feature

> - enable [Fast Refresh](https://www.npmjs.com/package/react-refresh) in development
> - use the [automatic JSX runtime](https://github.com/alloc/vite-react-jsx#faq)
> - avoid manual `import React` in `.jsx` and `.tsx` modules
> - dedupe the `react` and `react-dom` packages
> - use custom Babel plugins/presets

### 插件的职责

该插件返回的是组件，分别包含
* viteBabel
* viteReactRefresh
* viteReactJsx
涵盖了上面的5个功能点

#### viteBabel

顾名思义，则是做 `babel` 相关的事情。
判断各种环境加载对应的 `babel` 插件，支持 `类组件` , `[tj]sx` , `Reason` , `cjs` , 外部传入的 `babel` 配置, 最后用 `babel` 进行转换代码
除此之外，作为第一个插件，还做了一些环境预设的操作
* config相关，skipFastRefresh等变量
* 模块过滤。不符合条件的模块不进行代码转换，如下

```ts
import { createFilter } from '@rollup/pluginutils'

// 外部传入
let filter = createFilter(opts.include, opts.exclude)

// 匹配模块进行转换
transform(code, id, options) {
    if (filter(id)) {
        // do something
    }
    
}
```

* react-refresh环境准备，主要就是加了一个wrapper，热更新使用

```ts
if (useFastRefresh && /\$RefreshReg\$\(/.test(code)) {
    const accept = isReasonReact || isRefreshBoundary(result.ast!)
    code = addRefreshWrapper(code, id, accept)
}
```

总体对应功能点

> - use custom Babel plugins/presets
> - avoid manual `import React` in `.jsx` and `.tsx` modules

#### viteReactRefresh

`react-refresh` 是react热更新相关的库，具体为何以及如何适配，需要先看看[issue](https://github.com/facebook/react/issues/16604)

看完之后，就能大概理解源码 `fast-refresh.ts` 的一些写法

```ts
window.$RefreshReg$ = () => {}
window.$RefreshSig$ = () => (type) => type
```

这个插件主要是针对 `react-refresh` 的适配，对应功能点

> - enable [Fast Refresh](https://www.npmjs.com/package/react-refresh) in development
> - dedupe the `react` and `react-dom` packages

#### viteReactJsx

最后则是使用[automatic JSX runtime](https://github.com/alloc/vite-react-jsx#faq)
关键操作，在 `load` 阶段，针对 `react/jsx-runtime` 模块注入导出，如下

```ts
load(id: string) {
    if (id === runtimeId) {
        const runtimePath = resolve.sync(runtimeId, {
            basedir: projectRoot
        })
        const exports = ['jsx', 'jsxs', 'Fragment']
        return [
            `import * as jsxRuntime from ${JSON.stringify(runtimePath)}`,
            // We can't use `export * from` or else any callsite that uses
            // this module will be compiled to `jsxRuntime.exports.jsx`
            // instead of the more concise `jsx` alias.
            ...exports.map((name) => `export const ${name} = jsxRuntime.${name}`)
        ].join('\n')
    }
}
```

对应功能点
> - use the [automatic JSX runtime](https://github.com/alloc/vite-react-jsx#faq)
