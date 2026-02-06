# Vercel CLI 支持 Go 语言的完整解析

本文档详细分析 Vercel CLI 中 `dev` 命令和 `build` 命令对 Go 语言的支持，包括本地开发和生产部署的完整流程。

## 目录

### 第一部分：dev 命令（本地开发）

1. [概述](#1-概述)
2. [整体架构](#2-整体架构)
3. [执行流程详解](#3-执行流程详解)
4. [两种运行模式](#4-两种运行模式)
5. [核心源码文件](#5-核心源码文件)
6. [Go 版本管理与工具链](#6-go-版本管理与工具链)
7. [请求处理流程](#7-请求处理流程)
8. [热重载机制](#8-热重载机制)
9. [关键模板文件](#9-关键模板文件)
10. [框架检测与配置](#10-框架检测与配置)

### 第二部分：build 命令（生产构建）

11. [Build 命令概述](#11-build-命令概述)
12. [Build 整体架构](#12-build-整体架构)
13. [Build 执行流程详解](#13-build-执行流程详解)
14. [生产构建两种模式](#14-生产构建两种模式)
15. [Lambda 包装和 go-bridge](#15-lambda-包装和-go-bridge)
16. [缓存机制 prepareCache](#16-缓存机制-preparecache)
17. [构建输出结构](#17-构建输出结构)

### 第三部分：dev 与 build 的对比

18. [dev 与 build 的区别和联系](#18-dev-与-build-的区别和联系)
19. [完整对比总结表](#19-完整对比总结表)

---

# 第一部分：dev 命令（本地开发）

---

## 1. 概述

Vercel CLI 的 `dev` 命令通过 `@vercel/go` 运行时包支持 Go 语言的本地开发。它提供两种运行模式：

| 模式 | 适用场景 | 特点 |
|------|---------|------|
| **Serverless Function 模式** | 传统 API 函数 (`api/*.go`) | 每次请求启动独立进程，自动包装 HTTP Handler |
| **Standalone Server 模式** | 完整 Go HTTP 服务器 | 直接运行 `go run`，支持自定义服务器 |

---

## 2. 整体架构

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           用户请求 (HTTP)                                │
└───────────────────────────────────┬─────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                        DevServer (server.ts)                            │
│  • HTTP 服务器 & 代理                                                    │
│  • 路由匹配                                                              │
│  • 文件监控 (chokidar)                                                   │
│  • Builder 调度                                                          │
└───────────────────────────────────┬─────────────────────────────────────┘
                                    │
           ┌────────────────────────┴────────────────────────┐
           │                                                  │
           ▼                                                  ▼
┌──────────────────────────┐                    ┌──────────────────────────┐
│  Serverless Function     │                    │  Standalone Server       │
│  模式 (index.ts)         │                    │  模式 (standalone-       │
│                          │                    │  server.ts)              │
│  • 复制入口文件           │                    │                          │
│  • 生成 dev-server.go    │                    │  • 执行 `go run .`       │
│  • go build              │                    │  • 设置 PORT 环境变量     │
│  • 启动编译后的二进制     │                    │  • 反向代理               │
└──────────────────────────┘                    └──────────────────────────┘
           │                                                  │
           └────────────────────────┬─────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                        Go HTTP Server                                   │
│                      (用户代码运行)                                       │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 3. 执行流程详解

### 3.1 CLI 入口 (`packages/cli/src/commands/dev/dev.ts`)

```typescript
export default async function dev(
  client: Client,
  opts: Partial<Options>,
  args: string[],
  telemetry: DevTelemetryClient
) {
  const [dir = '.'] = args;
  let cwd = resolve(dir);
  const listen = parseListen(opts['--listen'] || '3000');

  // 1. 获取项目链接
  let link = await getLinkedProject(client, cwd);

  // 2. 如果未链接，引导用户设置
  if (link.status === 'not_linked' && !process.env.__VERCEL_SKIP_DEV_CMD) {
    link = await setupAndLink(client, cwd, { ... });
  }

  // 3. 获取项目设置和环境变量
  let projectSettings: ProjectSettings | undefined;
  let envValues: Record<string, string> = {};
  if (link.status === 'linked') {
    projectSettings = project;
    envValues = (await pullEnvRecords(client, project.id, 'vercel-cli:dev')).env;
  }

  // 4. 创建并启动 DevServer
  const devServer = new DevServer(cwd, {
    projectSettings,
    envValues,
    repoRoot,
    services,
  });

  await devServer.start(...listen);
}
```

**关键步骤：**
1. 解析命令行参数（监听地址，默认 3000 端口）
2. 获取/创建项目链接
3. 拉取环境变量
4. 检测多服务配置（实验性功能）
5. 创建 `DevServer` 实例并启动

### 3.2 DevServer 初始化 (`packages/cli/src/util/dev/server.ts`)

```typescript
export default class DevServer {
  public cwd: string;
  public proxy: httpProxy;
  public envConfigs: EnvConfigs;
  public files: BuilderInputs;
  private server: http.Server;
  private buildMatches: Map<string, BuildMatch>;
  private watcher?: FSWatcher;
  // ... 更多属性

  constructor(cwd: string, options: DevServerOptions) {
    this.cwd = cwd;
    this.envConfigs = { buildEnv: {}, runEnv: {}, allEnv: {} };
    this.envValues = options.envValues || {};
    this.projectSettings = options.projectSettings;

    // 创建 HTTP 代理
    this.proxy = httpProxy.createProxyServer({
      changeOrigin: true,
      ws: true,
      xfwd: true,
    });

    // ... 初始化其他组件
  }
}
```

### 3.3 Builder 检测与匹配

DevServer 通过 `detectBuilders` 函数自动检测项目中的 Go 文件并匹配相应的 Builder：

```typescript
// 零配置模式：自动检测 builders
if (!vercelConfig.builds || vercelConfig.builds.length === 0) {
  let {
    builders,
    warnings,
    errors,
    defaultRoutes,
    // ...
  } = await detectBuilders(files, pkg, {
    tag: 'latest',
    functions: vercelConfig.functions,
    projectSettings: projectSettings || this.projectSettings,
    workPath: this.cwd,
  });

  if (builders) {
    vercelConfig.builds = vercelConfig.builds || [];
    vercelConfig.builds.push(...builders);
  }
}
```

对于 Go 文件，会自动添加如下构建配置：
```json
{
  "src": "api/**/*.go",
  "use": "@vercel/go"
}
```

---

## 4. 两种运行模式

### 4.1 模式判断逻辑

```typescript
// packages/go/src/index.ts
export async function startDevServer(
  opts: StartDevServerOptions
): Promise<StartDevServerResult> {
  const { entrypoint, workPath, config, meta = {} } = opts;

  // 模式判断：framework 为 'go' 或 'services' 时使用 Standalone 模式
  if (config?.framework === 'go' || config?.framework === 'services') {
    const resolvedEntrypoint = await detectGoEntrypoint(workPath, entrypointWithExt);
    if (!resolvedEntrypoint) {
      throw new Error(`No Go entrypoint found. Expected one of: ${GO_CANDIDATE_ENTRYPOINTS.join(', ')}`);
    }
    return startStandaloneDevServer(opts, resolvedEntrypoint);
  }

  // 否则使用传统 Serverless Function 模式
  // ...
}
```

### 4.2 Serverless Function 模式（传统模式）

**适用场景：** `api/` 目录下的 Go 函数

**执行流程：**

```typescript
// packages/go/src/index.ts (startDevServer 函数)

// 1. 创建临时目录
const tmp = join(devCacheDir, 'go', Math.random().toString(32).substring(2));
const tmpPackage = join(tmp, entrypointDir);
await mkdirp(tmpPackage);

// 2. 分析入口文件获取函数名和包名
const analyzed = await getAnalyzedEntrypoint({
  entrypoint: entrypointWithExt,
  modulePath,
  workPath,
});

// 3. 准备构建文件
await Promise.all([
  copyEntrypoint(entrypointWithExt, tmpPackage),     // 复制入口文件
  copyDevServer(analyzed.functionName, tmpPackage),   // 生成 dev-server.go
  writeGoMod({ destDir: tmp, goModPath, packageName: analyzed.packageName }),
  writeGoWork(tmp, workPath, modulePath),
]);

// 4. 构建可执行文件
const go = await createGo({ modulePath, opts: { cwd: tmp, env }, workPath });
await go.build('./...', executable);

// 5. 启动开发服务器
const child = spawn(executable, [], {
  cwd: tmp,
  env,
  stdio: ['ignore', 'inherit', 'inherit', 'pipe'],  // FD 3 用于接收端口号
});

// 6. 等待端口信息
const portPipe = child.stdio[3];
portPipe.setEncoding('utf8');
portPipe.once('data', d => {
  resolve({ port: Number(d) });
});
```

**dev-server.go 模板 (`packages/go/dev-server.go`)：**

```go
package main

import (
	"io/ioutil"
	"net"
	"net/http"
	"os"
	"strconv"
)

func main() {
	// 创建处理器（占位符在构建时被替换为实际函数名）
	handler := http.HandlerFunc(__HANDLER_FUNC_NAME)

	// 监听随机端口
	listener, err := net.Listen("tcp", "127.0.0.1:0")
	if err != nil {
		panic(err)
	}

	port := listener.Addr().(*net.TCPAddr).Port
	portBytes := []byte(strconv.Itoa(port))

	// 通过 FD 3 发送端口号给父进程
	file := os.NewFile(3, "pipe")
	_, err2 := file.Write(portBytes)
	if err2 != nil {
		// 备用方案：写入端口文件
		portFile := os.Getenv("VERCEL_DEV_PORT_FILE")
		os.Unsetenv("VERCEL_DEV_PORT_FILE")
		ioutil.WriteFile(portFile, portBytes, 0644)
	}

	// 启动 HTTP 服务
	panic(http.Serve(listener, handler))
}
```

### 4.3 Standalone Server 模式（新模式）

**适用场景：** 完整的 Go HTTP 服务器（`main.go` 或 `cmd/api/main.go`）

**执行流程：**

```typescript
// packages/go/src/standalone-server.ts
export async function startStandaloneDevServer(
  opts: StartDevServerOptions,
  resolvedEntrypoint: string
): Promise<StartDevServerResult> {
  const { workPath, meta = {} } = opts;

  // 1. 使用随机端口（临时端口范围）
  const port = Math.floor(Math.random() * (65535 - 49152) + 49152);

  // 2. 设置环境变量
  const env = cloneEnv(process.env, meta.env, {
    PORT: String(port),  // 用户代码应该读取这个端口
  });

  // 3. 确定运行目标
  // - main.go 在根目录: go run .
  // - cmd/api/main.go: go run ./cmd/api
  const runTarget = resolvedEntrypoint === 'main.go' 
    ? '.' 
    : './' + dirname(resolvedEntrypoint);

  debug(`Starting standalone Go dev server: go run ${runTarget} (port ${port})`);

  // 4. 直接执行 go run
  const child = spawn('go', ['run', runTarget], {
    cwd: workPath,
    env,
    stdio: ['ignore', 'inherit', 'inherit'],
  });

  // 5. 等待服务器启动（简单等待 2 秒）
  await new Promise(resolve => setTimeout(resolve, 2000));

  return {
    port,
    pid: child.pid!,
  };
}
```

**入口点检测逻辑 (`packages/go/src/entrypoint.ts`)：**

```typescript
export const GO_CANDIDATE_ENTRYPOINTS = [
  'main.go',
  'cmd/api/main.go',
  'cmd/server/main.go',
];

export async function detectGoEntrypoint(
  workPath: string,
  configuredEntrypoint: string
): Promise<string | null> {
  // 1. 优先使用配置的入口点
  if (await pathExists(join(workPath, configuredEntrypoint))) {
    debug(`Using configured Go entrypoint: ${configuredEntrypoint}`);
    return configuredEntrypoint;
  }

  // 2. 搜索候选位置
  for (const candidate of GO_CANDIDATE_ENTRYPOINTS) {
    if (await pathExists(join(workPath, candidate))) {
      debug(`Detected Go entrypoint: ${candidate}`);
      return candidate;
    }
  }

  return null;
}
```

---

## 5. 核心源码文件

| 文件路径 | 说明 |
|---------|------|
| `packages/cli/src/commands/dev/dev.ts` | CLI `dev` 命令入口 |
| `packages/cli/src/util/dev/server.ts` | DevServer 核心实现（HTTP 服务器、路由、构建管理） |
| `packages/cli/src/util/dev/builder.ts` | Builder 执行和 Lambda 创建 |
| `packages/go/src/index.ts` | Go Builder 主文件（`build()`, `startDevServer()`, `prepareCache()`） |
| `packages/go/src/standalone-server.ts` | Standalone Server 模式实现 |
| `packages/go/src/entrypoint.ts` | Go 入口点检测 |
| `packages/go/src/go-helpers.ts` | Go SDK 管理、AST 分析、构建包装 |
| `packages/go/dev-server.go` | Serverless 开发服务器模板 |
| `packages/go/main.go` | 生产 Lambda 包装模板 |
| `packages/go/vc-init.go` | Standalone 模式的 Bootstrap 包装器（处理 IPC 协议） |
| `packages/frameworks/src/frameworks.ts` | 框架检测和配置定义 |

---

## 6. Go 版本管理与工具链

### 6.1 版本映射

```typescript
// packages/go/src/go-helpers.ts
const versionMap = new Map([
  ['1.24', '1.24.5'],
  ['1.23', '1.23.11'],
  ['1.22', '1.22.12'],
  ['1.21', '1.21.13'],
  ['1.20', '1.20.14'],
  ['1.19', '1.19.13'],
  ['1.18', '1.18.10'],
  ['1.17', '1.17.13'],
  ['1.16', '1.16.15'],
  ['1.15', '1.15.15'],
  ['1.14', '1.14.15'],
  ['1.13', '1.13.15'],
]);
```

### 6.2 Go 工具链查找顺序

```typescript
// packages/go/src/go-helpers.ts
export async function createGo({ modulePath, opts = {}, workPath }: CreateGoOptions): Promise<GoWrapper> {
  // 1. 解析 go.mod 中的版本要求
  let goPreferredVersion: GoVersions | undefined;
  if (modulePath) {
    goPreferredVersion = await parseGoModVersionFromModule(modulePath);
  }

  // 2. 选择版本（优先使用 go.mod 中指定的版本）
  const goSelectedVersion = goPreferredVersion
    ? goPreferredVersion.toolchain || goPreferredVersion.go
    : Array.from(versionMap.values())[0];  // 默认使用最新版本

  // 3. 按顺序查找 Go 安装
  const goDirs = {
    'local cache': goCacheDir,           // .vercel/cache/golang
    'global cache': goGlobalCacheDir,    // ~/.cache/com.vercel.cli/golang
    'system PATH': null,                 // 系统 PATH
  };

  for (const [label, goDir] of Object.entries(goDirs)) {
    try {
      // 检查版本是否匹配
      const { stdout } = await execa('go', ['version'], { env });
      const { version } = parseGoVersionString(stdout);
      
      if (version === goSelectedVersion) {
        debug(`Selected go ${version} (from ${label})`);
        return new GoWrapper(env, opts);
      }
    } catch {
      debug(`Go not found in ${label}`);
    }
  }

  // 4. 下载并缓存所需版本
  await download({ dest: goGlobalCacheDir, version: goSelectedVersion });
  return new GoWrapper(env, opts);
}
```

### 6.3 GoWrapper 类

```typescript
// packages/go/src/go-helpers.ts
export class GoWrapper {
  private env: Env;
  private opts: execa.Options;

  constructor(env: Env, opts: execa.Options = {}) {
    this.env = env;
    this.opts = opts;
  }

  private execute(...args: string[]) {
    debug(`Exec: go ${args.join(' ')}`);
    return execa('go', args, { stdio: 'inherit', ...this.opts, env: this.env });
  }

  // go mod tidy
  mod() {
    return this.execute('mod', 'tidy');
  }

  // go get
  get(src?: string) {
    const args = ['get'];
    if (src) args.push(src);
    return this.execute(...args);
  }

  // go build -ldflags '-s -w' -o <dest> <sources>
  build(src: string | string[], dest: string) {
    debug(`Building optimized 'go' binary ${src} -> ${dest}`);
    const sources = Array.isArray(src) ? src : [src];
    const flags = GO_FLAGS;  // ['-ldflags', '-s -w'] 优化二进制大小
    return this.execute('build', ...flags, '-o', dest, ...sources);
  }
}
```

### 6.4 AST 分析器

Go Builder 使用一个 Go 编写的 AST 分析工具来解析入口文件：

```typescript
// packages/go/src/go-helpers.ts
export async function getAnalyzedEntrypoint({
  entrypoint,
  modulePath,
  workPath,
}: { entrypoint: string; modulePath?: string; workPath: string }): Promise<Analyzed> {
  const bin = join(__dirname, `analyze${OUT_EXTENSION}`);

  try {
    // 如果分析器不存在，先构建它
    const isAnalyzeExist = await pathExists(bin);
    if (!isAnalyzeExist) {
      debug(`Building analyze bin: ${bin}`);
      const src = join(__dirname, '../analyze.go');
      const go = await createGo({ modulePath, opts: { cwd: __dirname }, workPath });
      await go.build(src, bin);
    }
  } catch (err) {
    console.error('Failed to build the Go AST analyzer');
    throw err;
  }

  // 执行分析器
  const args = [`-modpath=${modulePath}`, join(workPath, entrypoint)];
  analyzed = await execa.stdout(bin, args);

  // 返回分析结果：{ functionName: "Handler", packageName: "handler" }
  return JSON.parse(analyzed) as Analyzed;
}
```

---

## 7. 请求处理流程

### 7.1 DevServer 请求处理

当 HTTP 请求到达 DevServer 时：

```typescript
// packages/cli/src/util/dev/server.ts (约第 1960-2063 行)

// 检查 Builder 是否支持 startDevServer() 优化
const { builder, pkg: builderPkg } = match.builderWithPkg;
if (builder.version === 3 && typeof builder.startDevServer === 'function') {
  let devServerResult: StartDevServerResult = null;
  
  try {
    // 调用 Builder 的 startDevServer
    devServerResult = await builder.startDevServer({
      files,
      entrypoint: match.entrypoint,
      workPath,
      config: match.config || {},
      repoRootPath: this.repoRoot,
      meta: {
        isDev: true,
        requestPath,
        devCacheDir,
        env: { ...envConfigs.runEnv },
        buildEnv: { ...envConfigs.buildEnv },
      },
    });
  } catch (err: unknown) {
    output.prettyError(err);
    await this.sendError(req, res, requestId, 'NO_RESPONSE_FROM_FUNCTION', 502);
    return;
  }

  if (devServerResult) {
    const { port, pid, shutdown } = devServerResult;
    this.shutdownCallbacks.set(pid, shutdown);

    // 请求完成后关闭开发服务器
    res.once('close', () => {
      this.killBuilderDevServer(pid);
    });

    debug(`Proxying to "${builderPkg.name}" dev server (port=${port}, pid=${pid})`);

    // 代理请求到 Go 开发服务器
    return proxyPass(req, res, `http://127.0.0.1:${port}`, this, requestId, false);
  }
}
```

### 7.2 请求流程图

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│   用户请求    │────▶│  DevServer   │────▶│ 路由匹配     │────▶│ Builder匹配  │
│  HTTP        │     │  接收请求     │     │              │     │              │
└──────────────┘     └──────────────┘     └──────────────┘     └──────────────┘
                                                                      │
                     ┌────────────────────────────────────────────────┘
                     │
                     ▼
         ┌─────────────────────┐
         │ builder.version=3 ? │
         │ startDevServer存在? │
         └──────────┬──────────┘
                    │
        ┌───────────┴───────────┐
        │ YES                   │ NO
        ▼                       ▼
┌──────────────────┐    ┌──────────────────┐
│ startDevServer() │    │ 传统构建流程      │
│                  │    │ build() + Lambda │
└────────┬─────────┘    └──────────────────┘
         │
         ▼
┌──────────────────┐
│ Go 开发服务器启动 │
│ (临时端口)       │
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ proxyPass()      │
│ 代理请求到 Go    │
│ 开发服务器       │
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ 响应返回用户     │
│ 关闭 Go 进程     │
└──────────────────┘
```

---

## 8. 热重载机制

### 8.1 文件监控

DevServer 使用 `chokidar` 监控文件变化：

```typescript
// 创建文件监控器
this.watcher = watch(this.cwd, {
  ignored: (path: string) => !this.filter(path),
  ignoreInitial: true,
});

// 监听变化事件
this.watcher.on('add', (path: string) => {
  this.enqueueFsEvent('add', path);
});

this.watcher.on('change', (path: string) => {
  this.enqueueFsEvent('change', path);
});

this.watcher.on('unlink', (path: string) => {
  this.enqueueFsEvent('unlink', path);
});
```

### 8.2 事件聚合

为避免频繁重建，DevServer 会聚合文件系统事件：

```typescript
private enqueueFsEvent(type: string, path: string) {
  this.watchAggregationEvents.push({ type, path });
  
  if (this.watchAggregationId) {
    clearTimeout(this.watchAggregationId);
  }
  
  // 500ms 延迟聚合
  this.watchAggregationId = setTimeout(() => {
    this.handleFsEvents(this.watchAggregationEvents);
    this.watchAggregationEvents = [];
  }, 500);
}
```

### 8.3 重建触发

对于 Go 函数，当源文件变化时：
- **Serverless 模式**：下次请求时自动重新编译
- **Standalone 模式**：需要手动重启（`go run` 不支持热重载）

---

## 9. 关键模板文件

### 9.1 dev-server.go（Serverless 开发服务器）

用于 Serverless Function 模式，包装用户的 Handler 函数：

```go
// packages/go/dev-server.go
package main

import (
	"net"
	"net/http"
	"os"
	"strconv"
)

func main() {
	// __HANDLER_FUNC_NAME 会被替换为实际的处理函数名
	handler := http.HandlerFunc(__HANDLER_FUNC_NAME)

	// 监听随机端口
	listener, _ := net.Listen("tcp", "127.0.0.1:0")
	port := listener.Addr().(*net.TCPAddr).Port

	// 通过 FD 3 发送端口号（进程间通信）
	file := os.NewFile(3, "pipe")
	file.Write([]byte(strconv.Itoa(port)))

	// 启动服务
	http.Serve(listener, handler)
}
```

### 9.2 main.go（生产 Lambda 包装器）

用于生产部署，使用 Vercel Go Bridge：

```go
// packages/go/main.go
package main

import (
	"net/http"
	"os"
	"syscall"

	"__VC_HANDLER_PACKAGE_NAME"  // 用户包的导入路径
	vc "github.com/vercel/go-bridge/go/bridge"
)

func checkForLambdaWrapper() {
	wrapper := os.Getenv("AWS_LAMBDA_EXEC_WRAPPER")
	if wrapper == "" {
		return
	}
	os.Setenv("AWS_LAMBDA_EXEC_WRAPPER", "")
	argv := append([]string{wrapper}, os.Args...)
	syscall.Exec(wrapper, argv, os.Environ())
}

func main() {
	checkForLambdaWrapper()
	// __VC_HANDLER_FUNC_NAME 会被替换为实际函数名
	vc.Start(http.HandlerFunc(__VC_HANDLER_FUNC_NAME))
}
```

### 9.3 vc-init.go（Standalone Bootstrap）

用于 Standalone Server 生产部署，处理 Vercel IPC 协议：

```go
// packages/go/vc-init.go
// vc-init.go - Bootstrap wrapper for standalone Go servers on Vercel
// 
// 工作流程:
// 1. 连接 VERCEL_IPC_PATH Unix socket
// 2. 在内部端口启动用户服务器
// 3. 发送 "server-started" IPC 消息
// 4. 反向代理请求到用户服务器
// 5. 处理 /_vercel/ping 健康检查
// 6. 每次请求后发送 "end" IPC 消息

package main

import (
	"net"
	"net/http"
	"net/http/httputil"
	"os"
	"os/exec"
	// ...
)

func main() {
	startTime = time.Now()

	// 连接 IPC socket
	connectIPC()

	// 找一个空闲端口运行用户服务器
	userPort, _ := findFreePort()

	// 启动用户的服务器二进制
	cmd := exec.CommandContext(ctx, "./user-server")
	cmd.Env = append(os.Environ(), fmt.Sprintf("PORT=%d", userPort))
	cmd.Start()

	// 等待用户服务器就绪
	waitForServer(userPort, 30*time.Second)

	// 创建反向代理
	proxy := httputil.NewSingleHostReverseProxy(targetURL)

	// 发送 server-started IPC 消息
	sendIPCMessage(StartMessage{
		Type: "server-started",
		Payload: StartPayload{
			InitDuration: int(time.Since(startTime).Milliseconds()),
			HTTPPort:     3000,
		},
	})

	// 启动代理服务器
	server.ListenAndServe()
}
```

---

## 10. 框架检测与配置

### 10.1 Go 框架定义

```typescript
// packages/frameworks/src/frameworks.ts (约第 4275-4330 行)
{
  name: 'Go',
  slug: 'go',
  experimental: true,        // 实验性功能
  runtimeFramework: true,    // 运行时框架标记
  logo: 'https://api-frameworks.vercel.sh/framework-logos/go.svg',
  tagline: 'An open-source programming language supported by Google.',
  description: 'A generic Go application deployed as a serverless function.',
  website: 'https://go.dev',
  
  // 运行时配置
  useRuntime: { 
    src: 'index.go', 
    use: '@vercel/go' 
  },
  ignoreRuntimes: ['@vercel/go'],
  
  // 检测规则
  detectors: {
    every: [
      { path: 'go.mod' },   // 必须有 go.mod
    ],
    some: [
      { path: 'main.go' },           // 任一入口点
      { path: 'cmd/api/main.go' },
      { path: 'cmd/server/main.go' },
    ],
  },
  
  // 默认设置
  settings: {
    installCommand: {
      placeholder: '`go mod download`',
    },
    buildCommand: {
      placeholder: 'None',
      value: null,
    },
    devCommand: {
      placeholder: '`go run .` or `go run ./cmd/api`',
      value: null,
    },
    outputDirectory: {
      value: 'N/A',
    },
  },
  
  // 默认路由
  defaultRoutes: [
    { handle: 'filesystem' },
    { src: '/(.*)', dest: '/' },
  ],
}
```

### 10.2 检测流程

1. **文件检测**：检查项目中是否存在 `go.mod` + (`main.go` 或 `cmd/api/main.go` 或 `cmd/server/main.go`)
2. **框架匹配**：如果满足条件，自动设置 `framework: 'go'`
3. **Builder 分配**：为匹配的 Go 文件分配 `@vercel/go` Builder

---

## 11. 完整调用链路图

```
vercel dev
    │
    ▼
┌─────────────────────────────────────────────────────────────────┐
│  packages/cli/src/commands/dev/dev.ts                           │
│  • 解析参数 (--listen, --yes)                                    │
│  • 获取项目链接 (getLinkedProject)                               │
│  • 拉取环境变量 (pullEnvRecords)                                 │
│  • 检测服务 (tryDetectServices)                                  │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│  new DevServer(cwd, options)                                    │
│  packages/cli/src/util/dev/server.ts                            │
│  • 创建 HTTP 代理 (httpProxy)                                    │
│  • 初始化配置                                                    │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│  devServer.start(host, port)                                    │
│  • 启动 HTTP 服务器                                              │
│  • 初始化文件监控 (chokidar)                                     │
│  • 加载 Vercel 配置 (_getVercelConfig)                          │
│  • 检测 Builders (detectBuilders)                               │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│  detectBuilders (packages/fs-detectors)                         │
│  • 扫描项目文件                                                  │
│  • 匹配 Go 文件 → { src: "api/**/*.go", use: "@vercel/go" }     │
│  • 或检测到 Go 框架 → config.framework = 'go'                    │
└────────────────────────────┬────────────────────────────────────┘
                             │
    ┌────────────────────────┴────────────────────────┐
    │                                                  │
    ▼                                                  ▼
┌─────────────────────┐                    ┌─────────────────────┐
│  HTTP 请求到达       │                    │  文件变化事件        │
│  handleRequest()    │                    │  enqueueFsEvent()   │
└──────────┬──────────┘                    └──────────┬──────────┘
           │                                          │
           ▼                                          ▼
┌─────────────────────┐                    ┌─────────────────────┐
│  路由匹配           │                    │  500ms 聚合         │
│  devRouter()        │                    │  handleFsEvents()   │
└──────────┬──────────┘                    └──────────┬──────────┘
           │                                          │
           ▼                                          ▼
┌─────────────────────┐                    ┌─────────────────────┐
│  Builder 匹配       │                    │  triggerBuild()     │
│  getBuildMatches()  │                    │  重新构建            │
└──────────┬──────────┘                    └─────────────────────┘
           │
           ▼
┌─────────────────────────────────────────────────────────────────┐
│  @vercel/go Builder                                             │
│  packages/go/src/index.ts                                       │
│                                                                 │
│  if (builder.version === 3 && builder.startDevServer) {        │
│      // 使用 startDevServer 优化                                 │
│  }                                                              │
└────────────────────────────┬────────────────────────────────────┘
                             │
        ┌────────────────────┴────────────────────┐
        │                                          │
        ▼                                          ▼
┌─────────────────────┐                ┌─────────────────────┐
│ config.framework    │                │ config.framework    │
│ !== 'go'            │                │ === 'go'            │
│                     │                │ || 'services'       │
│ Serverless 模式     │                │                     │
│                     │                │ Standalone 模式     │
└──────────┬──────────┘                └──────────┬──────────┘
           │                                       │
           ▼                                       ▼
┌─────────────────────┐                ┌─────────────────────┐
│ 1. 创建临时目录      │                │ 1. 检测入口点        │
│ 2. AST 分析入口      │                │    detectGoEntry-   │
│    getAnalyzedEntry │                │    point()          │
│    point()          │                │                     │
│ 3. 复制入口文件      │                │ 2. 设置 PORT 环境    │
│ 4. 生成 dev-server  │                │    变量              │
│    .go              │                │                     │
│ 5. 写入 go.mod/     │                │ 3. spawn('go',      │
│    go.work          │                │    ['run', target]) │
│ 6. go build         │                │                     │
│ 7. spawn 编译后的    │                │ 4. 等待 2 秒        │
│    二进制            │                │                     │
│ 8. FD 3 接收端口     │                │ 5. 返回 {port, pid} │
│ 9. 返回 {port, pid} │                │                     │
└──────────┬──────────┘                └──────────┬──────────┘
           │                                       │
           └───────────────────┬───────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│  proxyPass(req, res, `http://127.0.0.1:${port}`)               │
│  • 代理请求到 Go 开发服务器                                      │
│  • 添加 Vercel 平台请求头                                        │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│  Go HTTP Server 处理请求                                        │
│  • Serverless: dev-server.go 包装的 Handler                     │
│  • Standalone: 用户的完整 HTTP 服务器                            │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│  响应返回                                                        │
│  • res.once('close') → killBuilderDevServer(pid)                │
│  • Serverless: 进程被终止                                        │
│  • Standalone: 进程继续运行                                      │
└─────────────────────────────────────────────────────────────────┘
```

---

## 12. 总结

Vercel CLI 的 `dev` 命令对 Go 语言的支持是一个精心设计的系统，包含以下核心特性：

### 12.1 双模式支持

| 特性 | Serverless Function 模式 | Standalone Server 模式 |
|------|-------------------------|----------------------|
| **适用场景** | API 函数 (`api/*.go`) | 完整 HTTP 服务器 |
| **触发条件** | 默认模式 | `framework: 'go'` 或 `'services'` |
| **启动方式** | `go build` + 执行二进制 | `go run <target>` |
| **进程生命周期** | 请求结束后终止 | 持续运行 |
| **端口通信** | FD 3 或端口文件 | PORT 环境变量 |
| **热重载** | 自动（下次请求重编译） | 手动重启 |

### 12.2 智能版本管理

- 自动解析 `go.mod` 中的版本要求
- 多级缓存（项目本地 → 全局 → 自动下载）
- 支持 Go 1.13 - 1.24 版本

### 12.3 高效的开发体验

- 按需编译（每次请求触发）
- 文件监控与事件聚合（500ms 延迟）
- 自动端口分配和进程管理

### 12.4 生产就绪

- AST 分析确保正确的函数导出
- IPC 协议支持 Vercel 平台集成
- 支持 Lambda 包装器和响应流
- 二进制优化 (`-ldflags '-s -w'`)

---

## 附录：快速参考

### 启动开发服务器

```bash
# 默认端口 3000
vercel dev

# 指定端口
vercel dev --listen 8080

# 跳过链接确认
vercel dev --yes
```

### 项目结构示例

**Serverless Function 模式：**
```
project/
├── api/
│   └── handler.go    # package handler, func Handler(w, r)
├── go.mod
└── vercel.json       # 可选
```

**Standalone Server 模式：**
```
project/
├── main.go           # package main, func main()
├── go.mod
└── vercel.json       # framework: "go"
```

或

```
project/
├── cmd/
│   └── api/
│       └── main.go   # package main, func main()
├── go.mod
└── vercel.json       # framework: "go"
```

### 环境变量

| 变量 | 说明 |
|-----|------|
| `PORT` | Standalone 模式下用户服务器应监听的端口 |
| `VERCEL_DEV_PORT_FILE` | Serverless 模式下端口文件路径（备用） |
| `GO111MODULE` | Go 模块模式开关（自动管理） |
| `GOROOT` | Go 安装路径（自动设置） |
| `GO_BUILD_FLAGS` | 自定义构建标志 |

---

# 第二部分：build 命令（生产构建）

## 11. Build 命令概述

`@vercel/go` 的 `build()` 函数负责将 Go 代码编译为可以在 Vercel 平台上运行的 Lambda 函数。与 `dev` 命令不同，`build` 命令是为生产环境优化的，会进行交叉编译并生成适用于 AWS Lambda 的二进制文件。

### 核心特点

| 特性 | 说明 |
|------|------|
| **目标平台** | Linux x86_64 或 ARM64 |
| **输出格式** | AWS Lambda `provided.al2023` 运行时 |
| **包装方式** | 使用 `go-bridge` 库处理 Lambda 事件 |
| **优化选项** | 二进制压缩 (`-ldflags '-s -w'`) |
| **缓存支持** | 支持 Go 模块缓存和工具链缓存 |

---

## 12. Build 整体架构

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         Vercel Build System                              │
│                    (vercel build / vercel deploy)                        │
└───────────────────────────────────┬─────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                        @vercel/go build()                                │
│  • 下载源文件                                                            │
│  • 分析入口点 (AST)                                                      │
│  • 设置交叉编译环境                                                       │
│  • 选择构建模式                                                          │
└───────────────────────────────────┬─────────────────────────────────────┘
                                    │
           ┌────────────────────────┴────────────────────────┐
           │                                                  │
           ▼                                                  ▼
┌──────────────────────────────┐              ┌──────────────────────────────┐
│  传统 Serverless Function    │              │  Standalone Server           │
│  模式 (buildHandler*)        │              │  模式 (buildStandaloneServer)│
│                              │              │                              │
│  • 生成 main.go 包装器       │              │  • 构建用户服务器二进制      │
│  • 使用 go-bridge 库         │              │  • 构建 vc-init.go 包装器    │
│  • 重命名 Handler 函数       │              │  • 打包两个可执行文件        │
│  • go build -> bootstrap     │              │                              │
└──────────────────────────────┘              └──────────────────────────────┘
           │                                                  │
           └────────────────────────┬─────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                            Lambda 输出                                   │
│  • files: { bootstrap, ... }  或  { executable, user-server, ... }      │
│  • handler: 'bootstrap' 或 'executable'                                 │
│  • runtime: 'provided.al2023' 或 'executable'                           │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                          Vercel 部署平台                                 │
│                    (AWS Lambda / Edge Runtime)                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 13. Build 执行流程详解

### 13.1 build() 函数入口 (`packages/go/src/index.ts`)

```typescript
export async function build(options: BuildOptions) {
  const { files, config, workPath, meta = {} } = options;
  let { entrypoint } = options;

  // 1. 准备工作目录
  const goPath = await getWriteableDirectory();
  const srcPath = join(goPath, 'src', 'lambda');
  const downloadPath = meta.skipDownload ? workPath : srcPath;
  await download(files, downloadPath, meta);

  // 2. 初始化撤销操作栈（用于清理）
  const undo: UndoActions = {
    fileActions: [],
    directoryCreation: [],
    functionRenames: [],
  };

  // 3. 设置交叉编译环境变量
  const env = cloneEnv(process.env, meta.env, {
    GOARCH: 'amd64',    // 目标架构：x86_64
    GOOS: 'linux',      // 目标系统：Linux
  });

  try {
    // 4. 初始化 Git 凭证（如果有私有仓库依赖）
    if (env.GIT_CREDENTIALS) {
      await initPrivateGit(env.GIT_CREDENTIALS);
    }

    // 5. 判断构建模式
    if (config?.framework === 'go' || config?.framework === 'services') {
      // Standalone Server 模式
      return buildStandaloneServer({ ...options, entrypoint: resolvedEntrypoint });
    }

    // 6. 传统 Serverless Function 模式
    // ... 详见下文
  } finally {
    // 7. 清理临时文件
    await cleanupFileSystem(undo);
  }
}
```

### 13.2 传统 Serverless Function 构建流程

```typescript
// 1. 重命名入口文件（处理 index.go 特殊情况）
const renamedEntrypoint = getRenamedEntrypoint(entrypoint);
if (renamedEntrypoint) {
  await move(join(workPath, entrypoint), join(workPath, renamedEntrypoint));
  entrypoint = renamedEntrypoint;
}

// 2. 查找 go.mod 文件
const { goModPath, isGoModInRootDir } = await findGoModPath(entrypointDirname, workPath);

// 3. AST 分析入口文件
const analyzed = await getAnalyzedEntrypoint({
  entrypoint,
  modulePath: goModPath ? dirname(goModPath) : undefined,
  workPath,
});

// 4. 验证包名（传统模式不允许 package main）
const packageName = analyzed.packageName;
if (goModPath && packageName === 'main') {
  throw new Error('Please change `package main` to `package handler`');
}

// 5. 重命名 Handler 函数（避免命名冲突）
const originalFunctionName = analyzed.functionName;  // 如 "Handler"
const handlerFunctionName = getNewHandlerFunctionName(originalFunctionName, entrypoint);
// 结果如: "Handler_api_hello_go"
await renameHandlerFunction(entrypointAbsolute, originalFunctionName, handlerFunctionName);

// 6. 处理 includeFiles 配置
const includedFiles: Files = {};
if (config && config.includeFiles) {
  const patterns = Array.isArray(config.includeFiles) ? config.includeFiles : [config.includeFiles];
  for (const pattern of patterns) {
    const fsFiles = await glob(pattern, entrypointDirname);
    for (const [assetName, asset] of Object.entries(fsFiles)) {
      includedFiles[assetName] = asset;
    }
  }
}

// 7. 创建 Go 工具包装器
const go = await createGo({ modulePath, opts: { cwd: entrypointDirname, env }, workPath });

// 8. 根据包名选择构建方式
if (packageName === 'main') {
  await buildHandlerAsPackageMain(buildOptions);   // 遗留模式
} else {
  await buildHandlerWithGoMod(buildOptions);       // Go Modules 模式
}

// 9. 创建 Lambda 输出
const runtime = await getProvidedRuntime();  // 'provided.al2023'
const lambda = new Lambda({
  files: { ...(await glob('**', outDir)), ...includedFiles },
  handler: HANDLER_FILENAME,    // 'bootstrap'
  runtime,
  runtimeLanguage: 'go',
  supportsWrapper: true,
  environment: {},
});

return { output: lambda };
```

### 13.3 buildHandlerWithGoMod() - Go Modules 构建

```typescript
async function buildHandlerWithGoMod({
  entrypoint,
  entrypointDirname,
  go,
  goModPath,
  handlerFunctionName,
  isGoModInRootDir,
  outDir,
  packageName,
  undo,
}: BuildHandlerOptions): Promise<void> {
  debug('Building Go handler with go.mod');

  const goModDirname = goModPath ? dirname(goModPath) : undefined;

  // 1. 确定包导入路径
  const relPackagePath = goModDirname ? posix.relative(goModDirname, entrypointDirname) : '';
  const goPackageName = relPackagePath ? posix.join(packageName, relPackagePath) : `${packageName}/${packageName}`;
  const goFuncName = `${packageName}.${handlerFunctionName}`;

  // 2. 确定 main.go 位置
  let mainGoFile: string;
  if (goModPath && isGoModInRootDir) {
    mainGoFile = join(downloadPath, MAIN_GO_FILENAME);
  } else if (goModDirname && !isGoModInRootDir) {
    mainGoFile = join(goModDirname, MAIN_GO_FILENAME);
  } else {
    mainGoFile = join(entrypointDirname, MAIN_GO_FILENAME);
  }

  // 3. 生成入口文件和 go.mod
  await Promise.all([
    writeEntrypoint(mainGoFile, goPackageName, goFuncName),  // 生成 main.go
    writeGoMod({ destDir: goModDirname || entrypointDirname, goModPath, packageName }),
  ]);

  // 4. 移动用户文件到包目录
  const finalDestination = join(entrypointDirname, packageName, basename(entrypoint));
  await move(entrypointAbsolute, finalDestination);

  // 5. 执行 go mod tidy
  await go.mod();

  // 6. 执行 go build
  const destPath = join(outDir, HANDLER_FILENAME);  // 'bootstrap'
  const src = [join(baseGoModPath, MAIN_GO_FILENAME)];
  await go.build(src, destPath);
}
```

### 13.4 buildHandlerAsPackageMain() - 遗留模式构建

```typescript
async function buildHandlerAsPackageMain({
  entrypointAbsolute,
  entrypointDirname,
  go,
  handlerFunctionName,
  outDir,
  undo,
}: BuildHandlerOptions): Promise<void> {
  debug('Building Go handler as package "main" (legacy)');

  // 1. 生成 main.go 到入口目录
  await writeEntrypoint(join(entrypointDirname, MAIN_GO_FILENAME), '', handlerFunctionName);

  // 2. 下载依赖
  await go.get();

  // 3. 构建二进制
  const destPath = join(outDir, HANDLER_FILENAME);
  const src = [
    join(entrypointDirname, MAIN_GO_FILENAME),
    entrypointAbsolute,
  ].map(file => normalize(file));
  await go.build(src, destPath);
}
```

---

## 14. 生产构建两种模式

### 14.1 模式判断逻辑

```typescript
// packages/go/src/index.ts
if (config?.framework === 'go' || config?.framework === 'services') {
  // Standalone Server 模式
  const resolvedEntrypoint = await detectGoEntrypoint(workPath, entrypoint);
  if (!resolvedEntrypoint) {
    throw new Error(`No Go entrypoint found. Expected one of: ${GO_CANDIDATE_ENTRYPOINTS.join(', ')}`);
  }
  return buildStandaloneServer({ ...options, entrypoint: resolvedEntrypoint });
}

// 否则使用传统 Serverless Function 模式
```

### 14.2 传统 Serverless Function 模式

**适用场景：** `api/` 目录下的 Go 函数

**特点：**
- 用户代码使用 `package handler`（非 main）
- 导出一个 `func Handler(w http.ResponseWriter, r *http.Request)` 函数
- 构建时生成包装器 `main.go`，使用 `go-bridge` 库
- 输出单个 `bootstrap` 二进制文件

**生成的 main.go 模板：**

```go
// packages/go/main.go
package main

import (
    "net/http"
    "os"
    "syscall"

    "__VC_HANDLER_PACKAGE_NAME"                    // 用户包的导入路径
    vc "github.com/vercel/go-bridge/go/bridge"     // Vercel Lambda Bridge
)

func checkForLambdaWrapper() {
    wrapper := os.Getenv("AWS_LAMBDA_EXEC_WRAPPER")
    if wrapper == "" {
        return
    }
    os.Setenv("AWS_LAMBDA_EXEC_WRAPPER", "")
    argv := append([]string{wrapper}, os.Args...)
    syscall.Exec(wrapper, argv, os.Environ())
}

func main() {
    checkForLambdaWrapper()
    vc.Start(http.HandlerFunc(__VC_HANDLER_FUNC_NAME))  // 如 handler.Handler_api_hello_go
}
```

### 14.3 Standalone Server 模式

**适用场景：** 完整的 Go HTTP 服务器

**特点：**
- 用户代码使用 `package main`，有自己的 `main()` 函数
- 用户服务器需要监听 `PORT` 环境变量指定的端口
- 构建时生成两个二进制：用户服务器 + Bootstrap 包装器
- Bootstrap 负责 IPC 协议和请求代理

**buildStandaloneServer() 实现：**

```typescript
// packages/go/src/standalone-server.ts
export async function buildStandaloneServer({
  files, entrypoint, config, workPath, meta = {},
}: BuildOptions): Promise<{ output: Lambda }> {
  debug(`Building standalone Go server: ${entrypoint}`);

  await download(files, workPath, meta);

  // 1. 获取 Lambda 配置（内存、超时、区域、架构）
  const lambdaOptions = await getLambdaOptionsFromFunction({
    sourceFile: entrypoint,
    config,
  });
  const architecture = lambdaOptions?.architecture || 'x86_64';

  // 2. 设置交叉编译环境
  const env = cloneEnv(process.env, meta.env, {
    GOARCH: architecture === 'arm64' ? 'arm64' : 'amd64',
    GOOS: 'linux',
    CGO_ENABLED: '0',  // 禁用 CGO 以确保静态链接
  });

  // 3. 创建 Go 工具包装器
  const { goModPath } = await findGoModPath(workPath, workPath);
  const modulePath = goModPath ? dirname(goModPath) : workPath;
  const go = await createGo({ modulePath, opts: { cwd: workPath, env }, workPath });

  const outDir = await getWriteableDirectory();
  const userServerPath = join(outDir, 'user-server');
  const bootstrapPath = join(outDir, 'executable');

  // 4. 构建用户服务器
  const buildTarget = entrypoint === 'main.go' ? '.' : './' + dirname(entrypoint);
  debug(`Building user Go server (${architecture}): go build ${buildTarget} -> ${userServerPath}`);
  await go.build(buildTarget, userServerPath);

  // 5. 构建 Bootstrap 包装器 (vc-init.go)
  const bootstrapSrc = join(__dirname, '../vc-init.go');
  const bootstrapBuildDir = await getWriteableDirectory();
  const bootstrapGoFile = join(bootstrapBuildDir, 'main.go');
  await copy(bootstrapSrc, bootstrapGoFile);
  await writeFile(join(bootstrapBuildDir, 'go.mod'), 'module vc-init\n\ngo 1.21\n');

  const bootstrapGo = await createGo({
    modulePath: bootstrapBuildDir,
    opts: { cwd: bootstrapBuildDir, env },
    workPath: bootstrapBuildDir,
  });
  await bootstrapGo.build('.', bootstrapPath);

  // 6. 处理 includeFiles
  const includedFiles: Files = {};
  if (config && config.includeFiles) {
    // ... 同上
  }

  // 7. 读取二进制文件并创建 Lambda
  const [userServerData, bootstrapData] = await Promise.all([
    readFile(userServerPath),
    readFile(bootstrapPath),
  ]);

  const lambda = new Lambda({
    ...lambdaOptions,
    files: {
      ...includedFiles,
      executable: new FileBlob({ mode: 0o755, data: bootstrapData }),
      'user-server': new FileBlob({ mode: 0o755, data: userServerData }),
    },
    handler: 'executable',
    runtime: 'executable',
    supportsResponseStreaming: true,
    architecture,
    runtimeLanguage: 'go',
  });

  return { output: lambda };
}
```

---

## 15. Lambda 包装和 go-bridge

### 15.1 go-bridge 库

`go-bridge` 是 Vercel 提供的 Go 库，用于在 AWS Lambda 环境中运行 Go HTTP Handler。

**核心功能：**
- 处理 Lambda Runtime API 通信
- 将 Lambda 事件转换为 `http.Request`
- 将 `http.ResponseWriter` 输出转换为 Lambda 响应
- 支持 Lambda Wrapper（如 Datadog、New Relic 等监控工具）

**使用方式：**

```go
import (
    "net/http"
    vc "github.com/vercel/go-bridge/go/bridge"
)

func Handler(w http.ResponseWriter, r *http.Request) {
    w.Write([]byte("Hello, World!"))
}

func main() {
    vc.Start(http.HandlerFunc(Handler))
}
```

### 15.2 vc-init.go Bootstrap 包装器

用于 Standalone Server 模式，处理 Vercel 的 IPC 协议：

```go
// packages/go/vc-init.go (简化版)
package main

import (
    "encoding/json"
    "fmt"
    "net"
    "net/http"
    "net/http/httputil"
    "os"
    "os/exec"
    "time"
)

var (
    ipcConn   net.Conn
    startTime time.Time
)

type StartMessage struct {
    Type    string       `json:"type"`
    Payload StartPayload `json:"payload"`
}

type StartPayload struct {
    InitDuration int `json:"initDuration"`
    HTTPPort     int `json:"httpPort"`
}

func main() {
    startTime = time.Now()

    // 1. 连接 IPC Socket
    ipcPath := os.Getenv("VERCEL_IPC_PATH")
    ipcConn, _ = net.Dial("unix", ipcPath)

    // 2. 找一个空闲端口运行用户服务器
    userPort := findFreePort()

    // 3. 启动用户服务器
    cmd := exec.Command("./user-server")
    cmd.Env = append(os.Environ(), fmt.Sprintf("PORT=%d", userPort))
    cmd.Start()

    // 4. 等待用户服务器就绪
    waitForServer(userPort, 30*time.Second)

    // 5. 创建反向代理
    targetURL, _ := url.Parse(fmt.Sprintf("http://127.0.0.1:%d", userPort))
    proxy := httputil.NewSingleHostReverseProxy(targetURL)

    // 6. 发送 server-started IPC 消息
    startMsg := StartMessage{
        Type: "server-started",
        Payload: StartPayload{
            InitDuration: int(time.Since(startTime).Milliseconds()),
            HTTPPort:     3000,  // Vercel 期望的端口
        },
    }
    json.NewEncoder(ipcConn).Encode(startMsg)

    // 7. 启动 HTTP 服务器，代理请求到用户服务器
    http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
        // 健康检查
        if r.URL.Path == "/_vercel/ping" {
            w.WriteHeader(http.StatusOK)
            return
        }

        // 代理请求
        proxy.ServeHTTP(w, r)

        // 发送请求结束消息
        endMsg := map[string]string{"type": "end"}
        json.NewEncoder(ipcConn).Encode(endMsg)
    })

    http.ListenAndServe(":3000", nil)
}
```

### 15.3 Lambda Wrapper 支持

`go-bridge` 支持 AWS Lambda Wrapper，允许在函数执行前注入监控代码：

```go
// main.go 中的检查
func checkForLambdaWrapper() {
    wrapper := os.Getenv("AWS_LAMBDA_EXEC_WRAPPER")
    if wrapper == "" {
        return
    }
    // 清除环境变量，避免无限递归
    os.Setenv("AWS_LAMBDA_EXEC_WRAPPER", "")
    // 使用 wrapper 重新执行自己
    argv := append([]string{wrapper}, os.Args...)
    syscall.Exec(wrapper, argv, os.Environ())
}
```

---

## 16. 缓存机制 prepareCache

### 16.1 prepareCache() 函数

```typescript
// packages/go/src/index.ts
export async function prepareCache({ workPath }: PrepareCacheOptions): Promise<Files> {
  // 缓存目录路径
  const goCacheDir = join(workPath, localCacheDir);  // .vercel/cache/golang

  // 检测是否为符号链接
  const stat = await lstat(goCacheDir);
  if (stat.isSymbolicLink()) {
    // 首次构建时，Go SDK 从全局缓存符号链接到本地
    // 需要将全局缓存移动到本地以便持久化
    const goGlobalCacheDir = await readlink(goCacheDir);
    debug(`Preparing cache by moving ${goGlobalCacheDir} -> ${goCacheDir}`);
    await unlink(goCacheDir);
    await move(goGlobalCacheDir, goCacheDir);
  }

  // 返回缓存目录下的所有文件
  const cache = await glob(`${localCacheDir}/**`, workPath);
  return cache;
}
```

### 16.2 缓存策略

```
首次构建：
┌─────────────────┐     下载      ┌─────────────────────────────────┐
│   Vercel 平台   │ ────────────▶ │ ~/.cache/com.vercel.cli/golang  │
│   请求构建      │               │ (全局缓存目录)                   │
└─────────────────┘               └───────────────┬─────────────────┘
                                                  │ symlink
                                                  ▼
                                  ┌─────────────────────────────────┐
                                  │ .vercel/cache/golang            │
                                  │ (本地符号链接)                   │
                                  └─────────────────────────────────┘

prepareCache() 执行后：
┌─────────────────┐     move      ┌─────────────────────────────────┐
│ 全局缓存        │ ────────────▶ │ .vercel/cache/golang            │
│ (被移动)        │               │ (实际文件，可被持久化)            │
└─────────────────┘               └─────────────────────────────────┘

后续构建：
┌─────────────────┐     恢复      ┌─────────────────────────────────┐
│   Vercel 平台   │ ────────────▶ │ .vercel/cache/golang            │
│   恢复缓存      │               │ (直接使用，除非版本变化)          │
└─────────────────┘               └─────────────────────────────────┘
```

### 16.3 缓存内容

| 缓存项 | 路径 | 说明 |
|-------|------|------|
| Go 编译器 | `.vercel/cache/golang/go<version>/` | 完整的 Go SDK |
| 模块缓存 | `.vercel/cache/golang/pkg/mod/` | 下载的依赖包 |
| 构建缓存 | `.vercel/cache/golang/go-build/` | 编译中间产物 |

---

## 17. 构建输出结构

### 17.1 传统 Serverless Function 模式

```typescript
const lambda = new Lambda({
  files: {
    'bootstrap': FileFsRef,       // 编译后的 Go 二进制（包含 go-bridge）
    // ... 其他通过 includeFiles 指定的文件
  },
  handler: 'bootstrap',           // Lambda 入口点
  runtime: 'provided.al2023',     // 或 'provided.al2'
  runtimeLanguage: 'go',
  supportsWrapper: true,          // 支持 Lambda Wrapper
  environment: {},
});
```

**Lambda 目录结构：**

```
/var/task/
├── bootstrap              # Go 二进制（可执行）
└── [includeFiles...]      # 用户指定的额外文件
```

### 17.2 Standalone Server 模式

```typescript
const lambda = new Lambda({
  ...lambdaOptions,              // memory, maxDuration, regions
  files: {
    'executable': FileBlob,      // Bootstrap 包装器（处理 IPC）
    'user-server': FileBlob,     // 用户的 Go 服务器
    // ... 其他通过 includeFiles 指定的文件
  },
  handler: 'executable',         // Lambda 入口点
  runtime: 'executable',         // 可执行文件 runtime
  supportsResponseStreaming: true,
  architecture,                  // 'x86_64' 或 'arm64'
  runtimeLanguage: 'go',
});
```

**Lambda 目录结构：**

```
/var/task/
├── executable             # Bootstrap 包装器（vc-init.go 编译后）
├── user-server            # 用户的 Go 服务器
└── [includeFiles...]      # 用户指定的额外文件
```

### 17.3 Lambda 配置选项

通过 `vercel.json` 或函数注释配置：

```json
{
  "functions": {
    "api/*.go": {
      "memory": 1024,           // 内存大小 (MB)
      "maxDuration": 30,        // 最大执行时间 (秒)
      "regions": ["iad1"],      // 部署区域
      "architecture": "arm64"   // CPU 架构 (x86_64 | arm64)
    }
  }
}
```

---

# 第三部分：dev 与 build 的对比

## 18. dev 与 build 的区别和联系

### 18.1 核心架构对比

```
                    dev 命令                              build 命令
                    ────────                              ──────────

┌──────────────────────────────┐        ┌──────────────────────────────┐
│     DevServer (CLI)          │        │     Vercel Build System      │
│   本地 HTTP 服务器            │        │   云端构建环境                │
└──────────────┬───────────────┘        └──────────────┬───────────────┘
               │                                       │
               ▼                                       ▼
┌──────────────────────────────┐        ┌──────────────────────────────┐
│  @vercel/go startDevServer() │        │  @vercel/go build()          │
│  本地开发服务器               │        │  生产构建                     │
└──────────────┬───────────────┘        └──────────────┬───────────────┘
               │                                       │
       ┌───────┴───────┐                       ┌───────┴───────┐
       │               │                       │               │
       ▼               ▼                       ▼               ▼
┌────────────┐  ┌────────────┐          ┌────────────┐  ┌────────────┐
│ Serverless │  │ Standalone │          │ Serverless │  │ Standalone │
│ dev 模式   │  │ dev 模式   │          │ build 模式 │  │ build 模式 │
└────────────┘  └────────────┘          └────────────┘  └────────────┘
       │               │                       │               │
       ▼               ▼                       ▼               ▼
┌────────────┐  ┌────────────┐          ┌────────────┐  ┌────────────┐
│dev-server  │  │ go run .   │          │ main.go +  │  │ vc-init +  │
│.go         │  │            │          │ go-bridge  │  │ user-server│
└────────────┘  └────────────┘          └────────────┘  └────────────┘
       │               │                       │               │
       ▼               ▼                       ▼               ▼
   本地进程         本地进程               Lambda          Lambda
   (macOS/Win/Linux)                       (Linux)        (Linux)
```

### 18.2 模板文件对比

| 模式 | dev 模板 | build 模板 | 区别 |
|------|---------|-----------|------|
| **Serverless Function** | `dev-server.go` | `main.go` | dev 直接监听本地端口；build 使用 go-bridge |
| **Standalone Server** | 无（直接 `go run`） | `vc-init.go` | dev 直接运行；build 需要 IPC 包装 |

### 18.3 编译环境对比

| 特性 | dev | build |
|------|-----|-------|
| **目标 OS** | 本地系统 (macOS/Windows/Linux) | `GOOS=linux` |
| **目标架构** | 本地架构 | `GOARCH=amd64` 或 `arm64` |
| **CGO** | 可启用 | `CGO_ENABLED=0` (Standalone) |
| **优化标志** | 无 | `-ldflags '-s -w'` |

### 18.4 通信协议对比

| 特性 | dev | build |
|------|-----|-------|
| **进程间通信** | FD 3 / 端口文件 | go-bridge / IPC Socket |
| **请求来源** | HTTP localhost | Lambda Runtime API |
| **响应方式** | HTTP Response | Lambda Response |

### 18.5 生命周期对比

**dev 命令：**
```
用户请求 → DevServer → startDevServer() → 启动 Go 进程 → 代理请求 → 响应 → 终止进程
          ↑                                                              │
          └──────────────────── 下一个请求 ←────────────────────────────┘
```

**build 命令部署后：**
```
Lambda 冷启动 → bootstrap 启动 → go-bridge/vc-init 初始化 → 等待请求
                                        │
                                        ▼
                        请求到达 → 处理 → 响应 → 等待下一个请求
                                        │
                                        ▼
                        (一段时间无请求后) Lambda 销毁
```

### 18.6 共享组件

尽管 `dev` 和 `build` 有很多差异，它们也共享许多核心组件：

| 共享组件 | 位置 | 用途 |
|---------|------|------|
| **Go 版本管理** | `go-helpers.ts` | 版本检测、下载、缓存 |
| **AST 分析器** | `analyze.go` | 分析入口文件，提取函数名和包名 |
| **入口点检测** | `entrypoint.ts` | 检测 `main.go`、`cmd/*/main.go` |
| **模式判断** | `index.ts` | 根据 `framework` 配置选择模式 |
| **GoWrapper 类** | `go-helpers.ts` | 封装 `go build`、`go mod`、`go get` |

### 18.7 调用链路对比

**dev 命令调用链路：**
```
vercel dev
    │
    ▼
DevServer.start()
    │
    ▼
DevServer.handleRequest()
    │
    ▼
builder.startDevServer()  ◄── @vercel/go
    │
    ├── Serverless: 编译 dev-server.go → 启动二进制 → 返回端口
    │
    └── Standalone: spawn('go', ['run', target]) → 等待启动 → 返回端口
    │
    ▼
proxyPass(request, response, port)
    │
    ▼
响应用户
```

**build 命令调用链路：**
```
vercel build / vercel deploy
    │
    ▼
Vercel Build System
    │
    ▼
builder.build()  ◄── @vercel/go
    │
    ├── Serverless: 生成 main.go → go build → 输出 bootstrap
    │
    └── Standalone: go build user-server → go build vc-init → 输出两个二进制
    │
    ▼
Lambda({ files, handler, runtime })
    │
    ▼
上传到 Vercel 平台
```

---

## 19. 完整对比总结表

### 19.1 功能对比

| 特性 | dev 命令 | build 命令 |
|------|---------|-----------|
| **主要用途** | 本地开发调试 | 生产部署 |
| **执行环境** | 本地机器 | Vercel 云平台 (AWS Lambda) |
| **目标平台** | 本地 OS | Linux |
| **编译模式** | 每次请求重新编译 | 一次性编译 |
| **热重载** | 支持（Serverless 模式） | 不适用 |
| **响应流** | 不支持 | 支持（Standalone 模式） |
| **并发处理** | 单进程 | Lambda 自动扩展 |

### 19.2 Serverless Function 模式对比

| 特性 | dev | build |
|------|-----|-------|
| **模板** | `dev-server.go` | `main.go` |
| **依赖库** | 无 | `go-bridge` |
| **端口通信** | FD 3 / 端口文件 | Lambda Runtime API |
| **输出** | 临时二进制 | `bootstrap` |
| **运行时** | 本地进程 | `provided.al2023` |

### 19.3 Standalone Server 模式对比

| 特性 | dev | build |
|------|-----|-------|
| **启动方式** | `go run .` | 预编译二进制 |
| **包装器** | 无 | `vc-init.go` |
| **端口配置** | `PORT` 环境变量 | `PORT` 环境变量 |
| **输出** | 无（源码直接运行） | `executable` + `user-server` |
| **IPC 协议** | 无 | Vercel IPC |

### 19.4 配置选项对比

| 配置 | dev 支持 | build 支持 |
|------|---------|-----------|
| `memory` | ❌ | ✅ |
| `maxDuration` | ❌ | ✅ |
| `regions` | ❌ | ✅ |
| `architecture` | ❌ | ✅ (x86_64/arm64) |
| `includeFiles` | ✅ | ✅ |
| `framework` | ✅ | ✅ |

### 19.5 使用场景建议

| 场景 | 推荐命令 | 说明 |
|------|---------|------|
| 本地开发调试 | `vercel dev` | 快速迭代，实时查看变更 |
| API 函数开发 | `vercel dev` | Serverless Function 模式 |
| 完整服务器开发 | `vercel dev` | Standalone Server 模式 |
| 预览部署 | `vercel` | 自动构建并部署到预览环境 |
| 生产部署 | `vercel --prod` | 自动构建并部署到生产环境 |
| 仅构建（不部署） | `vercel build` | 本地验证构建配置 |

---

## 附录：命令速查

### 本地开发

```bash
# 启动开发服务器
vercel dev

# 指定端口
vercel dev --listen 8080

# 跳过链接确认
vercel dev --yes
```

### 生产部署

```bash
# 部署到预览环境
vercel

# 部署到生产环境
vercel --prod

# 仅构建不部署
vercel build

# 查看构建日志
vercel logs <deployment-url>
```

### 项目配置示例

**vercel.json (Serverless Function):**

```json
{
  "functions": {
    "api/*.go": {
      "memory": 1024,
      "maxDuration": 30
    }
  }
}
```

**vercel.json (Standalone Server):**

```json
{
  "framework": "go",
  "functions": {
    "index.go": {
      "memory": 1024,
      "maxDuration": 60,
      "architecture": "arm64"
    }
  },
  "routes": [
    { "handle": "filesystem" },
    { "src": "/(.*)", "dest": "/" }
  ]
}
```
