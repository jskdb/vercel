# Vercel CLI `importBuilders` 机制深度分析

## 概述

`importBuilders` 是 Vercel CLI 中用于**动态加载和管理 Builder 包**的核心机制。它实现了一种**按需加载、自动安装**的插件架构，使得各语言的运行时支持可以作为独立的 npm 包进行管理和分发。

---

## 一、核心架构设计

### 1.1 模块化设计理念

```
┌─────────────────────────────────────────────────────────────────────┐
│                         Vercel CLI (vercel)                          │
│                                                                      │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐ │
│  │ @vercel/go  │  │@vercel/node │  │@vercel/python│ │@vercel/ruby │ │
│  │   Builder   │  │   Builder   │  │   Builder   │  │   Builder   │ │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘ │
│         │                │                │                │        │
│         └────────────────┴────────────────┴────────────────┘        │
│                                   │                                  │
│                    ┌──────────────┴──────────────┐                  │
│                    │    @vercel/build-utils      │                  │
│                    │    (共享构建工具库)           │                  │
│                    └─────────────────────────────┘                  │
└─────────────────────────────────────────────────────────────────────┘
```

### 1.2 包分离策略

```
CLI Package (vercel)                    Builder Packages
├── 核心命令逻辑                          ├── @vercel/go@3.4.0
├── DevServer 服务器                      ├── @vercel/node@5.5.32
├── 配置解析                              ├── @vercel/python@6.7.0
└── Builder 加载框架                      ├── @vercel/ruby@...
    (importBuilders)                      ├── @vercel/rust@...
                                          └── @vercel/static-build@...
```

---

## 二、`importBuilders` 完整流程

### 2.1 流程图

```
                              importBuilders(builderSpecs, cwd)
                                         │
                                         ▼
                           ┌─────────────────────────────┐
                           │    resolveBuilders()        │
                           │    解析 Builder 规格        │
                           └─────────────┬───────────────┘
                                         │
                    ┌────────────────────┼────────────────────┐
                    ▼                    ▼                    ▼
              ┌──────────┐        ┌──────────────┐     ┌──────────────┐
              │ 静态资源  │        │ 已安装的     │     │ 未安装的     │
              │@vercel/  │        │ Builder      │     │ Builder      │
              │ static   │        │              │     │              │
              └────┬─────┘        └──────┬───────┘     └──────┬───────┘
                   │                     │                    │
                   ▼                     ▼                    ▼
              内置返回              require() 加载       buildersToAdd
                   │                     │                    │
                   │                     │                    ▼
                   │                     │          ┌─────────────────┐
                   │                     │          │ installBuilders │
                   │                     │          │ npm install     │
                   │                     │          └────────┬────────┘
                   │                     │                   │
                   │                     │                   ▼
                   │                     │          resolveBuilders()
                   │                     │          (第二次)
                   └─────────────────────┴───────────────────┘
                                         │
                                         ▼
                           Map<string, BuilderWithPkg>
```

### 2.2 核心代码解析

#### 2.2.1 入口函数 `importBuilders`

```typescript:39:65:packages/cli/src/util/build/import-builders.ts
export async function importBuilders(
  builderSpecs: Set<string>,
  cwd: string
): Promise<Map<string, BuilderWithPkg>> {
  // 1. 构建目录路径：项目/.vercel/builders
  const buildersDir = join(cwd, VERCEL_DIR, 'builders');

  // 2. 第一次尝试解析 Builders
  let importResult = await resolveBuilders(buildersDir, builderSpecs);

  // 3. 如果有需要安装的 Builders
  if ('buildersToAdd' in importResult) {
    // 4. 执行 npm install 安装缺失的 Builders
    const installResult = await installBuilders(
      buildersDir,
      importResult.buildersToAdd
    );

    // 5. 第二次解析（安装后）
    importResult = await resolveBuilders(
      buildersDir,
      builderSpecs,
      installResult.resolvedSpecs
    );

    if ('buildersToAdd' in importResult) {
      throw new Error('Something went wrong!');
    }
  }

  return importResult.builders;
}
```

#### 2.2.2 解析函数 `resolveBuilders`

```typescript:67:193:packages/cli/src/util/build/import-builders.ts
export async function resolveBuilders(
  buildersDir: string,
  builderSpecs: Set<string>,
  resolvedSpecs?: Map<string, string>
): Promise<ResolveBuildersResult> {
  const builders = new Map<string, BuilderWithPkg>();
  const buildersToAdd = new Set<string>();

  for (const spec of builderSpecs) {
    const resolvedSpec = resolvedSpecs?.get(spec) || spec;
    const parsed = npa(resolvedSpec);  // npm-package-arg 解析

    const { name } = parsed;
    
    // 特殊处理：静态资源 Builder（内置）
    if (isStaticRuntime(name)) {
      builders.set(name, {
        builder: staticBuilder,
        pkg: { name },
        path: '',
        pkgPath: '',
      });
      continue;
    }

    try {
      let pkgPath: string | undefined;
      let builderPkg: PackageJson | undefined;

      try {
        // 策略1: 先尝试从 .vercel/builders/node_modules 加载
        pkgPath = join(buildersDir, 'node_modules', name, 'package.json');
        builderPkg = await readJSON(pkgPath);
      } catch (error: unknown) {
        // 策略2: 尝试从 CLI 本地依赖加载
        pkgPath = require_.resolve(`${name}/package.json`, {
          paths: [__dirname],
        });
        builderPkg = await readJSON(pkgPath);
      }

      // 版本验证...
      
      // 加载 Builder 模块
      const path = join(dirname(pkgPath), builderPkg.main || 'index.js');
      const builder = require_(path);

      builders.set(spec, {
        builder,
        pkg: { name, ...builderPkg },
        path,
        pkgPath,
      });
    } catch (err: any) {
      if (err.code === 'MODULE_NOT_FOUND' && !resolvedSpecs) {
        buildersToAdd.add(spec);  // 标记需要安装
      } else {
        throw err;
      }
    }
  }

  if (buildersToAdd.size > 0) {
    return { buildersToAdd };  // 返回需要安装的列表
  }

  return { builders };  // 返回已加载的 Builders
}
```

#### 2.2.3 安装函数 `installBuilders`

```typescript:195:290:packages/cli/src/util/build/import-builders.ts
async function installBuilders(
  buildersDir: string,
  buildersToAdd: Set<string>
) {
  const resolvedSpecs = new Map<string, string>();
  const buildersPkgPath = join(buildersDir, 'package.json');
  
  // 1. 创建空的 package.json（如果不存在）
  try {
    const emptyPkgJson = {
      private: true,
      license: 'UNLICENSED',
    };
    await outputJSON(buildersPkgPath, emptyPkgJson, { flag: 'wx' });
  } catch (err: any) {
    if (err.code !== 'EEXIST') throw err;
  }

  // 2. 执行 npm install
  output.log(`Installing Builder(s): ${Array.from(buildersToAdd).join(', ')}`);
  await execa(
    'npm',
    ['install', '@vercel/build-utils', ...buildersToAdd],
    { cwd: buildersDir, stdio: 'pipe', reject: true }
  );

  // 3. 创建兼容性符号链接 @now/build-utils -> @vercel/build-utils
  const nowScopePath = join(buildersDir, 'node_modules/@now');
  await mkdirp(nowScopePath);
  await symlink('../@vercel/build-utils', join(nowScopePath, 'build-utils'));

  // 4. 解析 URL 安装的包名
  const buildersPkg = await readJSONFile<PackageJson>(buildersPkgPath);
  for (const spec of buildersToAdd) {
    for (const [name, version] of Object.entries(buildersPkg.dependencies || {})) {
      if (version === spec) {
        resolvedSpecs.set(spec, name);
      }
    }
  }

  return { resolvedSpecs };
}
```

---

## 三、Builder 加载优先级

`importBuilders` 使用**两级查找策略**来加载 Builder：

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Builder 加载优先级                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  优先级 1: 项目级安装                                                 │
│  位置: <project>/.vercel/builders/node_modules/@vercel/go           │
│  场景: 用户指定特定版本，或首次运行时自动安装                          │
│                                                                      │
│  优先级 2: CLI 内置依赖                                               │
│  位置: <cli-install-path>/node_modules/@vercel/go                   │
│  场景: CLI 包自带的 Builder 版本                                      │
│                                                                      │
│  优先级 3: 自动安装                                                   │
│  动作: npm install @vercel/go                                        │
│  场景: 都找不到时，从 npm registry 下载最新版                          │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 代码实现：

```typescript
try {
  // 优先级 1: .vercel/builders/node_modules
  pkgPath = join(buildersDir, 'node_modules', name, 'package.json');
  builderPkg = await readJSON(pkgPath);
} catch (error: unknown) {
  if (error.code !== 'ENOENT') throw error;
  
  // 优先级 2: CLI 本地依赖
  pkgPath = require_.resolve(`${name}/package.json`, {
    paths: [__dirname],
  });
  builderPkg = await readJSON(pkgPath);
}
// 如果都失败，优先级 3: 标记为需要安装
```

---

## 四、为什么要分成独立的 npm 包？

### 4.1 设计动机

| 设计目标 | 解释 |
|---------|------|
| **独立版本管理** | 每个 Builder 可以独立发布和升级，不需要整体发布 CLI |
| **按需加载** | 用户只使用 Go，就不需要加载 Python/Ruby 等 Builder |
| **体积优化** | CLI 包保持轻量，Builders 按需下载 |
| **解耦开发** | 不同语言的 Builder 可以由不同团队独立开发 |
| **自定义扩展** | 用户可以创建和使用自定义 Builder |

### 4.2 包分离架构

```
┌─────────────────────────────────────────────────────────────────────┐
│                        npm Registry                                  │
│                                                                      │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐               │
│  │ vercel       │  │ @vercel/go   │  │ @vercel/node │               │
│  │ 50.12.3      │  │ 3.4.0        │  │ 5.5.32       │               │
│  │ (CLI Core)   │  │ (Go Builder) │  │(Node Builder)│               │
│  └──────────────┘  └──────────────┘  └──────────────┘               │
│                                                                      │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐               │
│  │@vercel/python│  │ @vercel/ruby │  │ @vercel/rust │               │
│  │ 6.7.0        │  │ ...          │  │ ...          │               │
│  │(Python Build)│  │(Ruby Builder)│  │(Rust Builder)│               │
│  └──────────────┘  └──────────────┘  └──────────────┘               │
│                                                                      │
│  ┌──────────────────────────────────────────────────┐               │
│  │              @vercel/build-utils                  │               │
│  │              (共享构建工具库)                      │               │
│  └──────────────────────────────────────────────────┘               │
└─────────────────────────────────────────────────────────────────────┘
```

### 4.3 各包职责

| 包名 | 版本 | 职责 |
|------|------|------|
| `vercel` | 50.12.3 | CLI 核心，命令处理，DevServer |
| `@vercel/go` | 3.4.0 | Go 语言编译和运行时支持 |
| `@vercel/node` | 5.5.32 | Node.js 函数打包和运行 |
| `@vercel/python` | 6.7.0 | Python WSGI/ASGI 支持 |
| `@vercel/ruby` | - | Ruby 函数支持 |
| `@vercel/rust` | - | Rust/Wasm 支持 |
| `@vercel/static-build` | - | 静态站点框架构建 |
| `@vercel/build-utils` | - | 共享构建工具（Lambda、Files 等） |

---

## 五、CLI 的依赖方式

### 5.1 CLI package.json 中的 Builder 依赖

```json:37:64:packages/cli/package.json
{
  "dependencies": {
    "@vercel/backends": "workspace:*",
    "@vercel/build-utils": "workspace:*",
    "@vercel/go": "workspace:*",          // Go Builder
    "@vercel/node": "workspace:*",        // Node.js Builder
    "@vercel/python": "workspace:*",      // Python Builder
    "@vercel/ruby": "workspace:*",        // Ruby Builder
    "@vercel/rust": "workspace:*",        // Rust Builder
    "@vercel/static-build": "workspace:*", // Static Builder
    // ... 框架特定 Builders
    "@vercel/next": "workspace:*",
    "@vercel/remix-builder": "workspace:*",
    // ...
  }
}
```

### 5.2 `workspace:*` 的含义

在 pnpm monorepo 中：
- `workspace:*` 表示使用工作区内的本地包
- 发布到 npm 时会被替换为实际版本号
- 保证开发时使用最新代码，发布时版本锁定

---

## 六、Builder 包的结构

以 `@vercel/go` 为例：

### 6.1 package.json

```json:1:43:packages/go/package.json
{
  "name": "@vercel/go",
  "version": "3.4.0",
  "license": "Apache-2.0",
  "main": "./dist/index",           // 入口文件
  "files": [
    "dist",                          // 编译后的 JS
    "*.go"                           // Go 模板文件
  ],
  "devDependencies": {
    "@vercel/build-utils": "workspace:*",  // 共享工具
    "execa": "^1.0.0",                      // 进程执行
    "fs-extra": "^7.0.0",                   // 文件操作
    // ...
  }
}
```

### 6.2 目录结构

```
packages/go/
├── package.json
├── src/
│   ├── index.ts          # Builder 主入口，导出 build/startDevServer
│   ├── go-helpers.ts     # Go 编译辅助函数
│   └── ...
├── dist/                  # 编译后的 JavaScript
│   └── index.js
├── dev-server.go          # Dev 模式模板
├── vc-init.go             # Build 模式桥接模板
└── go-bridge.go           # Lambda 桥接代码
```

### 6.3 Builder API 导出

```typescript
// packages/go/src/index.ts
export const version = 3;

export async function build(options: BuildOptions): Promise<BuildResult> {
  // 编译 Go 代码为 Lambda
}

export async function startDevServer(options: StartDevServerOptions): Promise<StartDevServerResult> {
  // 启动 Go 开发服务器
}

export async function prepareCache(options: PrepareCacheOptions): Promise<Files> {
  // 准备缓存
}
```

---

## 七、动态加载的优势

### 7.1 按需安装场景

```
场景: 用户项目只包含 Go 代码

传统方式 (单一包):
  npm install vercel
  └── 下载所有语言支持 (Go + Python + Ruby + Rust + ...)
  └── 安装体积大，依赖复杂

importBuilders 方式:
  npm install vercel
  └── 下载 CLI 核心 (轻量)
  
  vercel dev
  └── 检测到 api/*.go
  └── 只加载 @vercel/go (按需)
```

### 7.2 版本隔离

```
场景: 不同项目需要不同版本的 Builder

项目 A/.vercel/builders/
└── node_modules/@vercel/go@3.4.0

项目 B/.vercel/builders/
└── node_modules/@vercel/go@3.3.0

两个项目可以使用不同版本的 Builder，互不影响
```

### 7.3 自定义 Builder

```json
// vercel.json
{
  "builds": [
    {
      "src": "api/*.go",
      "use": "my-custom-go-builder@1.0.0"  // 使用自定义 Builder
    }
  ]
}
```

`importBuilders` 会自动从 npm 下载 `my-custom-go-builder`。

---

## 八、调用时机

### 8.1 dev 命令

```typescript
// packages/cli/src/util/dev/builder.ts
export async function getBuildMatches(...) {
  const builds = vercelConfig.builds || [{ src: '**', use: '@vercel/static' }];
  
  // 收集所有需要的 Builder 规格
  const builderSpecs = new Set(builds.map(b => b.use).filter(Boolean));
  // 例如: Set { '@vercel/go', '@vercel/node' }
  
  // 动态加载 Builders
  const buildersWithPkgs = await importBuilders(builderSpecs, cwd);
  
  // 使用加载的 Builders 处理请求...
}
```

### 8.2 build 命令

```typescript
// packages/cli/src/commands/build/index.ts
async function build(...) {
  // 类似逻辑，加载需要的 Builders
  const builders = await importBuilders(builderSpecs, cwd);
  
  // 执行构建...
  for (const [spec, builderWithPkg] of builders) {
    await builderWithPkg.builder.build(options);
  }
}
```

---

## 九、安装目录结构

执行 `vercel dev` 后，`.vercel/builders/` 目录结构：

```
<project>/
└── .vercel/
    └── builders/
        ├── package.json           # 记录已安装的 Builders
        └── node_modules/
            ├── @vercel/
            │   ├── go/            # Go Builder
            │   ├── build-utils/   # 共享工具
            │   └── ...
            └── @now/
                └── build-utils -> ../@vercel/build-utils  # 兼容符号链接
```

---

## 十、总结

### 10.1 核心设计理念

| 理念 | 实现 |
|------|------|
| **插件架构** | 每个语言运行时作为独立 npm 包 |
| **按需加载** | `importBuilders` 动态检测和加载 |
| **自动安装** | 缺失的 Builder 自动从 npm 安装 |
| **版本隔离** | 项目级 `.vercel/builders/` 目录 |
| **向后兼容** | `@now/*` → `@vercel/*` 符号链接 |

### 10.2 流程总结

```
1. 解析 vercel.json 或检测文件类型
   └── 确定需要的 Builders (如 @vercel/go)

2. 调用 importBuilders()
   └── resolveBuilders() 第一次尝试

3. 查找策略
   ├── 项目级: .vercel/builders/node_modules/
   └── CLI 内置: CLI 安装目录/node_modules/

4. 如果找不到
   └── installBuilders() 执行 npm install

5. resolveBuilders() 第二次
   └── 从安装位置加载

6. 返回 Map<string, BuilderWithPkg>
   └── 包含 builder 模块和包信息

7. 使用 Builder
   ├── dev: builder.startDevServer()
   └── build: builder.build()
```

### 10.3 这种设计的意义

1. **CLI 保持轻量** - 核心包不包含所有语言运行时代码
2. **独立版本控制** - 各 Builder 可以独立发布修复
3. **社区扩展** - 任何人可以创建自定义 Builder
4. **项目隔离** - 不同项目可以使用不同版本
5. **灵活性** - 支持从 npm、URL、git 等多种来源安装 Builder
