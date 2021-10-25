# vite-plugin-legacy 1.6.3

> 生成兼容性 js 代码，集成 babel 做的事情

返回 4 个插件

```ts
return [
  legacyConfigPlugin,
  legacyGenerateBundlePlugin,
  legacyPostPlugin,
  legacyEnvPlugin,
];
```

其中核心处理逻辑是`legacyGenerateBundlePlugin`, `legacyPostPlugin`

### legacyConfigPlugin

防止 config.build 不存在，兼容一下设置空对象

```ts
/**
 * @type {import('vite').Plugin}
 */
const legacyConfigPlugin = {
  name: "vite:legacy-config",

  apply: "build",
  config(config) {
    if (!config.build) {
      config.build = {};
    }
  },
};
```

### legacyGenerateBundlePlugin

用了两个钩子，做了两件事

1. `configResolved`, 给出错误提示
2. `generateBundle`, 根据 bundle，生成兼容性代码

重点看`generateBundle`做的事情

#### generateBundle

针对入口文件

- 判断环境，生成 modern 的 polyfill
- 判断环境，生成 legacy 的 polyfill，此处用得比较多。核心方法`detectPolyfills`, `buildPolyfillChunk`
  1. 检测兼容性，设置 polyfill 的 Set。`detectPolyfills`
  2. 调用`buildPolyfillChunk`

```ts
/**
 * @param {Set<string>} imports
 * @return {import('rollup').Plugin}
 */
function polyfillsPlugin(imports, externalSystemJS) {
  return {
    name: "vite:legacy-polyfills",
    resolveId(id) {
      if (id === polyfillId) {
        return id;
      }
    },
    load(id) {
      if (id === polyfillId) {
        return (
          // imports即为各种polyfill，如
          /**
           * import "core-js/modules/es.promise"
           * import "core-js/modules/es.array.map.js"
           * import "core-js/modules/es.array.includes.js"
           *
           */
          [...imports].map((i) => `import "${i}";`).join("") +
          (externalSystemJS ? "" : `import "systemjs/dist/s.min.js";`)
        );
      }
    },
  };
}
```

### legacyPostPlugin

- `configResolved`
  修改 config, 默认场景新增 rollupOptions.output 为
  ```ts
  {
    output: [
      {
        format: "system",
        entryFileNames: "assets/[name]-legacy.[hash].js",
        chunkFileNames: "assets/[name]-legacy.[hash].js",
      },
      {},
    ];
  }
  ```
- `renderChunk`
用了 **3** 个`babel`插件，`recordAndRemovePolyfillBabelPlugin`, `replaceLegacyEnvBabelPlugin`, `wrapIIFEBabelPlugin`
  - recordAndRemovePolyfillBabelPlugin: 对每个模块都用babel转一次代码，把polyfill加到`legacyPolyfills`, 由legacyGenerateBundlePlugin生成完整的polyfill，防止重复生成
  - replaceLegacyEnvBabelPlugin: 将`__VITE_IS_LEGACY__`转成`true`, 暂时不明其意
  - wrapIIFEBabelPlugin: 包一层iife。防止作用域影响

- `transformIndexHtml`
  1. 注入现代浏览器polyfill
  2. 注入fix safiri 10 nomodule
  3. 注入兼容性代码，到body, nomodule形式
  4. 注入兼容性代码入口，到body, nomodule形式
  5. 动态注入现代性代码，module形式

这里vite会优先加载module的代码，成功之后不再加载兼容性代码
所以在个别案例中可能会挂掉，要考虑modernPolyfill

- `generateBundle`
避免重复生成静态资源

### legacyEnvPlugin

判断 vite 版本，给提示

```ts
let envInjectionFailed = false;
/**
 * @type {import('vite').Plugin}
 */
const legacyEnvPlugin = {
  name: "vite:legacy-env",

  config(config, env) {
    if (env) {
      return {
        define: {
          "import.meta.env.LEGACY":
            env.command === "serve" || config.build.ssr
              ? false
              : legacyEnvVarMarker,
        },
      };
    } else {
      envInjectionFailed = true;
    }
  },

  configResolved(config) {
    if (envInjectionFailed) {
      config.logger.warn(
        `[@vitejs/plugin-legacy] import.meta.env.LEGACY was not injected due ` +
          `to incompatible vite version (requires vite@^2.0.0-beta.69).`
      );
    }
  },
};
```

小结：
整体思路是，结合`babel`检测源码兼容性，生成对应的polyfill文件，再注入到html。
有一个注意点是：因为`renderChunk`的执行时机比`generateBundle`要早
所以是先执行`legacyPostPlugin`插件的`renderChunk`检测兼容性，接着`legacyGenerateBundlePlugin`才能在`generateBundle`生成兼容性代码，和阅读顺序相反。
