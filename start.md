### 前提

[vite github](https://github.com/vitejs/vite)

后面的文件地址都在这里面

### vite 如何加载配置文件

`packages/vite/src/node/config.ts` 中的 `resovleConfig`

进行一些必要的变量声明后，我们进入到`解析配置`逻辑中

```ts
let { configFile } = config;
if (configFile !== false) {
  // 默认都会走到下面加载配置文件的逻辑，除非你手动指定 configFile 为 false
  /**
   * loadConfigFromFile 会从你指定的文件中去获取配置
   */
  const loadResult = await loadConfigFromFile(
    configEnv,
    configFile,
    config.root,
    config.logLevel,
  );
  if (loadResult) {
    // 解析配置文件的内容后，和命令行配置合并
    config = mergeConfig(loadResult.config, config);
    configFile = loadResult.path;
    configFileDependencies = loadResult.dependencies;
  }
}
```

> 第一步是解析配置文件的内容，然后与命令行配置合并。值得注意的是，后面有一个记录 `configFileDependencies` 的操作。因为配置文件代码可能会有第三方库的依赖，所以当第三方库依赖的代码更改时，Vite 可以通过 HMR 处理逻辑中记录的 `configFileDependencies` 检测到更改，再重启 `DevServer` ，来保证当前生效的配置永远是`最新`的

**loadConfigFromFile 实现**

```ts
// 这是我们的文件命名
export const DEFAULT_CONFIG_FILES = [
  'vite.config.js',
  'vite.config.mjs',
  'vite.config.ts',
  'vite.config.cjs',
  'vite.config.mts',
  'vite.config.cts',
];

export async function loadConfigFromFile(
  configEnv: ConfigEnv,
  configFile?: string,
  configRoot: string = process.cwd(),
  logLevel?: LogLevel,
): Promise<{
  path: string;
  config: UserConfig;
  dependencies: string[];
} | null> {
  const start = performance.now();
  const getTime = () => `${(performance.now() - start).toFixed(2)}ms`;

  // 处理的config 文件lujing
  let resolvedPath: string | undefined;

  if (configFile) {
    // 存在 用户自定义的 --config
    resolvedPath = path.resolve(configFile);
  } else {
    // 不存在就从 DEFAULT_CONFIG_FILES 去匹配 , 默认是在根目录下的
    for (const filename of DEFAULT_CONFIG_FILES) {
      const filePath = path.resolve(configRoot, filename);
      // 不存在就是 继续遍历
      if (!fs.existsSync(filePath)) continue;

      // 找到了就退出
      resolvedPath = filePath;
      break;
    }
  }

  // 如果不存在 就抛出
  if (!resolvedPath) {
    debug('no config file found.');
    return null;
  }

  // 判断是否是 esmodule
  let isESM = false;
  if (/\.m[jt]s$/.test(resolvedPath)) {
    // 文件后缀 是 mjs/mts 说明是 esmodule
    isESM = true;
  } else if (/\.c[jt]s$/.test(resolvedPath)) {
    // cjs / cts 说明不是
    isESM = false;
  } else {
    // 会尝试获取 package.json 中的 type 字段 如果是 module 也说明是 esmodule
    try {
      const pkg = lookupFile(configRoot, ['package.json']);
      isESM = !!pkg && JSON.parse(pkg).type === 'module';
    } catch (e) {}
  }

  try {
    /**
     * bundleConfigFile 会把配置文件 使用esbuild 打包
     */
    const bundled = await bundleConfigFile(resolvedPath, isESM);
    /**
     * 加载编译后的 bundle 代码 返回 用户的config (对象或者 方法)
     */
    const userConfig = await loadConfigFromBundledFile(
      resolvedPath,
      bundled.code,
      isESM,
    );
    debug(`bundled config file loaded in ${getTime()}`);

    // 赋值
    const config = await (typeof userConfig === 'function'
      ? userConfig(configEnv)
      : userConfig);
    if (!isObject(config)) {
      throw new Error(`config must export or return an object.`);
    }
    return {
      path: normalizePath(resolvedPath),
      config,
      dependencies: bundled.dependencies, // 记录依赖
    };
  } catch (e) {
    createLogger(logLevel).error(
      colors.red(`failed to load config from ${resolvedPath}`),
      { error: e },
    );
    throw e;
  }
}
```

**loadConfigFromBundledFile 实现**

```ts
const _require = createRequire(import.meta.url);
async function loadConfigFromBundledFile(
  fileName: string,
  bundledCode: string,
  isESM: boolean,
): Promise<UserConfigExport> {
  if (isESM) {
    // 如果是 esmodule
    /**
     *  会将编译后的 js 代码写入临时文件，通过 Node 原生 ESM Import 来读取这个临时的内容，以获取到配置内容，再直接删掉临时文件
     */
    const fileBase = `${fileName}.timestamp-${Date.now()}`;
    const fileNameTmp = `${fileBase}.mjs`;
    const fileUrl = `${pathToFileURL(fileBase)}.mjs`;
    fs.writeFileSync(fileNameTmp, bundledCode);
    try {
      return (await dynamicImport(fileUrl)).default;
    } finally {
      try {
        fs.unlinkSync(fileNameTmp);
      } catch {}
    }
  } else {
    // 处理 cjs/cts
    // 大体的思路是通过拦截原生 require.extensions 的加载函数来实现对 bundle 后配置代码的加载
    const extension = path.extname(fileName); // 取出后缀
    const realFileName = fs.realpathSync(fileName);
    const loaderExt = extension in _require.extensions ? extension : '.js'; // 是否存在对应的loader
    const defaultLoader = _require.extensions[loaderExt]!; // 默认的加载器
    // 拦截原生 require 对于`.js`或者`.ts`的加载
    _require.extensions[loaderExt] = (module: NodeModule, filename: string) => {
      if (filename === realFileName) {
        // 针对 vite 配置文件的加载特殊处理
        (module as NodeModuleWithCompile)._compile(bundledCode, filename);
      } else {
        defaultLoader(module, filename);
      }
    };
    // 清除这个文件的缓存
    delete _require.cache[_require.resolve(fileName)];
    // 在调用完 module._compile 编译完配置代码后，进行一次手动的 require，即可拿到配置对象
    const raw = _require(fileName);
    // 恢复原生的加载方法
    _require.extensions[loaderExt] = defaultLoader;
    return raw.__esModule ? raw.default : raw;
  }
}

// 原生 require 对于 js 文件的加载代码
Module._extensions['.js'] = function (module, filename) {
  var content = fs.readFileSync(filename, 'utf8');
  module._compile(stripBOM(content), filename);
};

// module._compile 相当于手动编译一个模块，该方法在 Node 内部的实现如下
Module.prototype._compile = function (content, filename) {
  var self = this;
  var args = [self.exports, require, self, filename, dirname];
  return compiledWrapper.apply(self.exports, args);
};
```

第二个重点环节是 解析用户插件。首先，我们通过 `apply` 参数 过滤出需要生效的用户插件。为什么这么做呢？因为有些插件只在开发阶段生效，或者说只在生产环境生效，我们可以通过 apply: 'serve' 或 'build' 来指定它们，同时也可以将 apply 配置为一个函数，来自定义插件生效的条件

```ts
const filterPlugin = (p: Plugin) => {
  if (!p) {
    // 不存在plugin
    return false;
  } else if (!p.apply) {
    // 没有实现 apply
    return true;
  } else if (typeof p.apply === 'function') {
    // apply 是一个方法
    return p.apply({ ...config, mode }, configEnv);
  } else {
    // apply 等于当前的 command command只有两个值 build/serve
    return p.apply === command;
  }
};

// 解析用户插件
/**
 * asyncFlatten 拍平 异步的 plugin
 */
const rawUserPlugins = (
  (await asyncFlatten(config.plugins || [])) as Plugin[]
).filter(filterPlugin);

// 通过sortUserPlugins 排序
const [prePlugins, normalPlugins, postPlugins] =
  sortUserPlugins(rawUserPlugins);
```

接着，Vite 会拿到这些过滤且排序完成的插件，依次调用插件 `config` 钩子，进行配置合并

```ts
const userPlugins = [...prePlugins, ...normalPlugins, ...postPlugins];
// 这里的config 是合并完成的config
config = await runConfigHook(config, userPlugins, configEnv);

// runConfig实现
async function runConfigHook(
  config: InlineConfig,
  plugins: Plugin[],
  configEnv: ConfigEnv,
): Promise<InlineConfig> {
  let conf = config;
  /**
   * getSortedPluginsByHook,两个参数 hookName, plugin数组
   * 返回的是 plugins 中的 实现了指定的 hookName 的plugin, 并且按照 pre post 普通排序的数组
   */
  for (const p of getSortedPluginsByHook('config', plugins)) {
    const hook = p.config; // 取出 config 钩子
    const handler = hook && 'handler' in hook ? hook.handler : hook; // 看是否有handler
    if (handler) {
      const res = await handler(conf, configEnv); // 调用hook
      if (res) {
        conf = mergeConfig(conf, res); // 合并配置
      }
    }
  }

  return conf;
}
```

然后解析项目的根目录即 `root` 参数，默认取 `process.cwd()`的结果

```ts
const resolvedRoot = normalizePath(
  config.root ? path.resolve(config.root) : process.cwd(),
);
```

紧接着处理 `alias` ，这里需要加上一些内置的 `alias` 规则，如`@vite/env`、`@vite/client` 这种直接重定向到 Vite 内部的模块

```ts
const clientAlias = [
  { find: /^\/?@vite\/env/, replacement: ENV_ENTRY },
  { find: /^\/?@vite\/client/, replacement: CLIENT_ENTRY },
];

const resolvedAlias = normalizeAlias(
  mergeAlias(clientAlias, config.resolve?.alias || []),
);

const resolveOptions: ResolvedConfig['resolve'] = {
  mainFields: config.resolve?.mainFields ?? DEFAULT_MAIN_FIELDS,
  browserField: config.resolve?.browserField ?? true,
  conditions: config.resolve?.conditions ?? [],
  extensions: config.resolve?.extensions ?? DEFAULT_EXTENSIONS,
  dedupe: config.resolve?.dedupe ?? [],
  preserveSymlinks: config.resolve?.preserveSymlinks ?? false,
  alias: resolvedAlias,
};
```

### vite 加载环境变量

```ts
// load .env files
const envDir = config.envDir
  ? normalizePath(path.resolve(resolvedRoot, config.envDir))
  : resolvedRoot;
const userEnv =
  inlineConfig.envFile !== false &&
  loadEnv(mode, envDir, resolveEnvPrefix(config));

const userNodeEnv = process.env.VITE_USER_NODE_ENV;
if (!isNodeEnvSet && userNodeEnv) {
  if (userNodeEnv === 'development') {
    process.env.NODE_ENV = 'development';
  } else {
    // NODE_ENV=production is not supported as it could break HMR in dev for frameworks like Vue
    logger.warn(
      `NODE_ENV=${userNodeEnv} is not supported in the .env file. ` +
        `Only NODE_ENV=development is supported to create a development build of your project. ` +
        `If you need to set process.env.NODE_ENV, you can set it in the Vite config instead.`,
    );
  }
}
```

loadEnv 其实就是扫描 `process.env `与 `.env` 文件，解析出 env 对象，值得注意的是，这个对象的属性最终会被挂载到 `import.meta.env` 这个全局对象上

**解析 env 对象的实现思路如下**

- 遍历 process.env 的属性，拿到指定前缀开头的属性（默认指定为 `VITE_`），并挂载 env 对象上
- 遍历 .env 文件，解析文件，然后往 env 对象挂载那些以指定前缀开头的属性。遍历的文件先后顺序如下(下面的 mode 开发阶段为 development，生产环境为 production)
  - .env.${mode}.local
  - .env.${mode}
  - .env.local
  - .env

> 特殊情况: 如果中途遇到 NODE_ENV 属性，则挂到 process.env.VITE_USER_NODE_ENV，Vite 会优先通过这个属性来决定是否走生产环境的构建。

接下来是对资源公共路径即 `base URL` 的处理

```ts
const isProduction = process.env.NODE_ENV === 'production';

// resolve public base url
const isBuild = command === 'build';
const relativeBaseShortcut = config.base === '' || config.base === './';

const resolvedBase = relativeBaseShortcut
  ? !isBuild || config.build?.ssr
    ? '/'
    : './'
  : resolveBaseUrl(config.base, isBuild, logger) ?? '/';

/**
 * logger 是在 resolveRoot 之前初始化的
 */
const resolvedBuildOptions = resolveBuildOptions(config.build, logger);

/**
 * resolveBaseUrl 实现
 */
export function resolveBaseUrl(
  base: UserConfig['base'] = '/',
  isBuild: boolean,
  logger: Logger,
): string {
  // .开头的路径，自动重写为 /
  if (base.startsWith('.')) {
    logger.warn(
      colors.yellow(
        colors.bold(
          `(!) invalid "base" option: ${base}. The value can only be an absolute ` +
            `URL, ./, or an empty string.`,
        ),
      ),
    );
    return '/';
  }

  // isExternalUrl https/http开头的字符串
  const isExternal = isExternalUrl(base);
  if (!isExternal && !base.startsWith('/')) {
    logger.warn(
      colors.yellow(
        colors.bold(`(!) "base" option should start with a slash.`),
      ),
    );
  }

  if (!isBuild || !isExternal) {
    // 下面的 new URL 尽可能的保证了 是"/"
    base = new URL(base, 'http://vitejs.dev').pathname;
    // 不是以 / 开头 都要以/ 开头
    if (!base.startsWith('/')) {
      base = '/' + base;
    }
  }

  return base;
}
```

当然，还有对 `cacheDir` 的解析，这个路径相对于在 Vite 预编译时写入依赖产物的路径

```ts
const pkgPath = lookupFile(resolvedRoot, [`package.json`], { pathOnly: true });
// 默认为 node_module/.vite
const cacheDir = normalizePath(
  config.cacheDir
    ? path.resolve(resolvedRoot, config.cacheDir)
    : pkgPath
    ? path.join(path.dirname(pkgPath), `node_modules/.vite`)
    : path.join(resolvedRoot, `.vite`),
);
```

紧接着处理用户配置的 `assetsInclude`，将其转换为一个过滤器函数

```ts
const assetsFilter =
  config.assetsInclude &&
  (!Array.isArray(config.assetsInclude) || config.assetsInclude.length)
    ? createFilter(config.assetsInclude)
    : () => false;
```

Vite 后面会将用户传入的 `assetsInclude` 和内置的规则合并

```ts
// 这个配置决定是否让 Vite 将对应的后缀名视为静态资源文件（asset）来处理
assetsInclude(file: string) {
  return DEFAULT_ASSETS_RE.test(file) || assetsFilter(file)
},
```

### vite 路径解析器工厂

这里所说的`路径解析器`，是指调用`插件容器`进行路径解析的函数

```ts
// 用于处理特殊场景
// 比如用于处理 optimizer, css @imports
const createResolver: ResolvedConfig['createResolver'] = (options) => {
  let aliasContainer: PluginContainer | undefined;
  let resolverContainer: PluginContainer | undefined;
  // 返回的函数可以理解为一个解析器
  return async (id, importer, aliasOnly, ssr) => {
    let container: PluginContainer;
    if (aliasOnly) {
      container =
        aliasContainer ||
        // 新建 aliasContainer
        (aliasContainer = await createPluginContainer({
          ...resolved,
          plugins: [aliasPlugin({ entries: resolved.resolve.alias })],
        }));
    } else {
      container =
        resolverContainer ||
        // 新建 resolverContainer
        (resolverContainer = await createPluginContainer({
          ...resolved,
          plugins: [
            aliasPlugin({ entries: resolved.resolve.alias }),
            resolvePlugin({
              ...resolved.resolve,
              root: resolvedRoot,
              isProduction,
              isBuild: command === 'build',
              ssrConfig: resolved.ssr,
              asSrc: true,
              preferRelative: false,
              tryIndex: true,
              ...options,
            }),
          ],
        }));
    }
    return (
      await container.resolveId(id, importer, {
        ssr,
        scan: options?.scan,
      })
    )?.id;
  };
};
```

这个解析器未来会在依赖预构建的时候用上，具体用法如下:

```ts
const resolve = config.createResolver();
// 调用以拿到 react 路径
rseolve('react', undefined, undefined, false);
```

这里有 `aliasContainer` 和 `resolverContainer` 两个工具对象，它们都含有 `resolveId` 这个专门解析路径的方法，可以被 Vite 调用来获取解析结果

接着会顺便处理一个 `public` 目录，也就是 Vite 作为静态资源服务的目录

```ts
const { publicDir } = config;
const resolvedPublicDir =
  publicDir !== false && publicDir !== ''
    ? path.resolve(
        resolvedRoot,
        typeof publicDir === 'string' ? publicDir : 'public',
      )
    : '';
```

中间还有 处理 `server`, `workerConfig` 自己瞅瞅源码

最后通过 `resolved` 对象来整理一下

```ts
const resolvedConfig: ResolvedConfig = {
  configFile: configFile ? normalizePath(configFile) : undefined,
  configFileDependencies: configFileDependencies.map((name) =>
    normalizePath(path.resolve(name)),
  ),
  inlineConfig,
  root: resolvedRoot,
  base: resolvedBase.endsWith('/') ? resolvedBase : resolvedBase + '/',
  rawBase: resolvedBase,
  resolve: resolveOptions,
  publicDir: resolvedPublicDir,
  cacheDir,
  command,
  mode,
  ssr,
  isWorker: false,
  mainConfig: null,
  isProduction,
  plugins: userPlugins,
  server,
  build: resolvedBuildOptions,
  preview: resolvePreviewOptions(config.preview, server),
  env: {
    ...userEnv,
    BASE_URL,
    MODE: mode,
    DEV: !isProduction,
    PROD: isProduction,
  },
  assetsInclude(file: string) {
    return DEFAULT_ASSETS_RE.test(file) || assetsFilter(file);
  },
  logger,
  packageCache: new Map(),
  createResolver,
  optimizeDeps: {
    disabled: 'build',
    ...optimizeDeps,
    esbuildOptions: {
      preserveSymlinks: resolveOptions.preserveSymlinks,
      ...optimizeDeps.esbuildOptions,
    },
  },
  worker: resolvedWorkerOptions,
  appType: config.appType ?? (middlewareMode === 'ssr' ? 'custom' : 'spa'),
  experimental: {
    importGlobRestoreExtension: false,
    hmrPartialAccept: false,
    ...config.experimental,
  },
  getSortedPlugins: undefined!,
  getSortedPluginHooks: undefined!,
};
const resolved: ResolvedConfig = {
  ...config,
  ...resolvedConfig,
};
```

### vite 生成插件流水线

```ts
(resolved.plugins as Plugin[]) = await resolvePlugins(
  resolved,
  prePlugins,
  normalPlugins,
  postPlugins,
);

// call configResolved hooks
await Promise.all([
  ...resolved
    .getSortedPluginHooks('configResolved')
    .map((hook) => hook(resolved)),
  ...resolvedConfig.worker
    .getSortedPluginHooks('configResolved')
    .map((hook) => hook(workerResolved)),
]);
```

先生成完整插件列表传给 `resolve.plugins`，而后调用每个插件的 `configResolved` 钩子函数
