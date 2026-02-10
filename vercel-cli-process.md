# Vercel CLI 脚手架执行流程完整解析

本文档详细梳理 Vercel CLI 从入口到执行的完整过程，主要针对 `dev` 命令和 Go 语言场景。

---

## 目录

1. [整体架构概览](#1-整体架构概览)
2. [CLI 入口与初始化](#2-cli-入口与初始化)
3. [命令解析与路由](#3-命令解析与路由)
4. [dev 命令执行流程](#4-dev-命令执行流程)
5. [DevServer 初始化与启动](#5-devserver-初始化与启动)
6. [配置文件加载](#6-配置文件加载)
7. [Builder 检测与加载](#7-builder-检测与加载)
8. [HTTP 请求处理流程](#8-http-请求处理流程)
9. [Go 语言 Builder 调用](#9-go-语言-builder-调用)
10. [完整调用链路图](#10-完整调用链路图)
11. [关键源文件索引](#11-关键源文件索引)

---

## 1. 整体架构概览

### 1.1 Vercel CLI 架构图

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                              用户终端                                            │
│                         $ vercel dev [dir]                                      │
└─────────────────────────────────────────┬───────────────────────────────────────┘
                                          │
                                          ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                           CLI 入口 (vc.js)                                       │
│                       packages/cli/src/index.ts                                  │
│  ┌────────────────────────────────────────────────────────────────────────────┐ │
│  │ • 解析命令行参数                                                            │ │
│  │ • 初始化日志系统 (output)                                                   │ │
│  │ • 读取全局配置 (~/.vercel/config.json)                                      │ │
│  │ • 创建 Client 实例                                                          │ │
│  │ • 命令路由分发                                                              │ │
│  └────────────────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────┬───────────────────────────────────────┘
                                          │
                                          ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                         dev 命令处理器                                           │
│                   packages/cli/src/commands/dev/                                 │
│  ┌────────────────────────────────────────────────────────────────────────────┐ │
│  │ • 检查递归调用保护                                                          │ │
│  │ • 项目链接检查 (getLinkedProject)                                           │ │
│  │ • 拉取环境变量 (pullEnvRecords)                                             │ │
│  │ • 创建 DevServer 实例                                                       │ │
│  └────────────────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────┬───────────────────────────────────────┘
                                          │
                                          ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                            DevServer                                             │
│                   packages/cli/src/util/dev/server.ts                            │
│  ┌────────────────────────────────────────────────────────────────────────────┐ │
│  │ • HTTP 服务器监听                                                           │ │
│  │ • 文件系统扫描                                                              │ │
│  │ • 配置加载 (vercel.json)                                                    │ │
│  │ • Builder 检测与匹配                                                        │ │
│  │ • 请求路由处理                                                              │ │
│  │ • 代理转发                                                                  │ │
│  └────────────────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────┬───────────────────────────────────────┘
                                          │
                    ┌─────────────────────┼─────────────────────┐
                    │                     │                     │
                    ▼                     ▼                     ▼
          ┌─────────────────┐   ┌─────────────────┐   ┌─────────────────┐
          │   @vercel/go    │   │  @vercel/node   │   │ @vercel/python  │
          │   (Go Builder)  │   │  (Node Builder) │   │ (Python Builder)│
          └────────┬────────┘   └────────┬────────┘   └────────┬────────┘
                   │                     │                     │
                   ▼                     ▼                     ▼
          ┌─────────────────┐   ┌─────────────────┐   ┌─────────────────┐
          │  Go 子进程       │   │  Node 子进程     │   │  Python 子进程   │
          │  (编译/运行)     │   │  (运行)          │   │  (运行)          │
          └─────────────────┘   └─────────────────┘   └─────────────────┘
```

### 1.2 核心模块职责

| 模块 | 路径 | 职责 |
|------|------|------|
| **CLI 入口** | `src/index.ts` | 参数解析、配置加载、命令分发 |
| **命令定义** | `src/commands/*/command.ts` | 定义命令参数、选项、描述 |
| **命令执行** | `src/commands/*/index.ts` | 命令入口逻辑 |
| **DevServer** | `src/util/dev/server.ts` | HTTP 服务器、路由、代理 |
| **Builder** | `src/util/dev/builder.ts` | 构建执行、进程管理 |
| **Router** | `src/util/dev/router.ts` | 请求路由匹配 |
| **项目链接** | `src/util/projects/link.ts` | .vercel 目录管理 |

---

## 2. CLI 入口与初始化

### 2.1 入口文件配置

**`packages/cli/package.json`:**

```json
{
  "name": "vercel",
  "bin": {
    "vc": "./dist/vc.js",
    "vercel": "./dist/vc.js"
  }
}
```

当用户执行 `vercel dev` 时，实际执行的是 `./dist/vc.js`，它是由 `src/index.ts` 编译生成的。

### 2.2 main() 函数执行流程

**`packages/cli/src/index.ts`:**

```typescript
// 主函数入口
async function main() {
  // 1. 解析命令行参数
  const { subcommand, parsedArgs } = parseArguments();
  
  // 2. 获取工作目录
  const cwd = parsedArgs.flags['--cwd'] || process.cwd();
  
  // 3. 初始化输出日志系统
  output.initialize({
    debug: parsedArgs.flags['--debug'],
  });
  
  // 4. 读取本地配置 (vercel.json)
  const localConfig = await getConfig(cwd);
  
  // 5. 确定目标命令
  const targetOrSubcommand = parsedArgs.args[0];
  const targetCommand = commands.get(targetOrSubcommand);
  
  // 6. 确保全局配置目录存在
  await mkdirp(VERCEL_DIR);  // ~/.vercel
  
  // 7. 读取全局配置文件
  const config = await configFiles.readConfigFile();      // ~/.vercel/config.json
  const authConfig = await configFiles.readAuthConfigFile();  // ~/.vercel/auth.json
  
  // 8. 初始化 Telemetry
  const telemetry = new TelemetryEventStore();
  
  // 9. 创建 HTTP Client 实例
  client = new Client({
    stdin: process.stdin,
    stdout: process.stdout,
    stderr: process.stderr,
    output,
    config,
    authConfig,
    localConfig,
  });
  
  // 10. 根据子命令分发到对应处理函数
  let exitCode: number;
  switch (targetCommand) {
    case 'dev':
      telemetry.trackCliCommandDev(userSuppliedSubCommand);
      func = require('./commands/dev').default;
      break;
    // ... 其他命令
  }
  
  // 11. 执行命令并返回退出码
  exitCode = await func(client);
  
  return exitCode;
}

// 启动
main().then(exitCode => process.exit(exitCode));
```

### 2.3 初始化流程图

```
vercel dev [dir]
      │
      ▼
┌─────────────────────────────────────────────────────────────┐
│                     main()                                   │
├─────────────────────────────────────────────────────────────┤
│  1. parseArguments()                                        │
│     ├── 解析 --debug, --cwd, --yes 等全局标志               │
│     └── 提取子命令 (dev) 和参数 ([dir])                     │
│                                                             │
│  2. 初始化 output 日志系统                                   │
│     └── 根据 --debug 设置日志级别                           │
│                                                             │
│  3. getConfig(cwd)                                          │
│     └── 读取本地 vercel.json (如果存在)                     │
│                                                             │
│  4. 确保全局配置目录存在                                     │
│     └── mkdir -p ~/.vercel                                  │
│                                                             │
│  5. 读取全局配置                                            │
│     ├── ~/.vercel/config.json  (设置配置)                   │
│     └── ~/.vercel/auth.json    (认证信息)                   │
│                                                             │
│  6. new Client()                                            │
│     └── 创建 API 客户端，用于与 Vercel API 通信             │
│                                                             │
│  7. 命令分发                                                │
│     └── switch(targetCommand) → require('./commands/dev')  │
│                                                             │
│  8. 执行命令                                                │
│     └── await func(client) → dev(client, opts, args)       │
└─────────────────────────────────────────────────────────────┘
```

---

## 3. 命令解析与路由

### 3.1 命令注册

**`packages/cli/src/commands/index.ts`:**

```typescript
import { devCommand } from './dev/command';
import { buildCommand } from './build/command';
// ... 其他命令

// 所有命令结构体
const commandsStructs = [
  aliasCommand,
  apiCommand,
  bisectCommand,
  buildCommand,
  devCommand,
  // ... 共 30+ 个命令
];

// 创建命令映射 Map
export const commands = new Map<string, string>();

for (const command of commandsStructs) {
  // 获取命令名称和别名
  const aliases = getCommandAliases(command);
  for (const alias of aliases) {
    commands.set(alias, command.name);
  }
}

// 结果示例:
// commands.get('dev') → 'dev'
// commands.get('develop') → 'dev'  (别名)
```

### 3.2 dev 命令定义

**`packages/cli/src/commands/dev/command.ts`:**

```typescript
import { yesOption } from '../../util/arg-common';

export const devCommand = {
  name: 'dev',
  aliases: ['develop'],  // 支持 vercel develop
  description: 'Starts the `vercel dev` server.',
  arguments: [
    {
      name: 'dir',
      required: false,
    },
  ],
  options: [
    {
      name: 'listen',
      shorthand: 'l',
      type: String,
      description: 'Specify a URI endpoint on which to listen',
    },
    yesOption,  // --yes, -y
    {
      name: 'port',
      shorthand: 'p',
      type: String,
      deprecated: true,  // 已废弃，使用 --listen
    },
  ],
  examples: [
    {
      name: 'Run the local development server',
      value: 'vercel dev',
    },
    {
      name: 'Run the local server on a specific port',
      value: 'vercel dev --listen 5005',
    },
  ],
};
```

### 3.3 命令路由分发

**`packages/cli/src/index.ts` (第 640-740 行):**

```typescript
// 根据命令名称分发到对应的处理函数
let func: any;
switch (targetCommand) {
  case 'alias':
    func = require('./commands/alias').default;
    break;
  case 'build':
    func = require('./commands/build').default;
    break;
  case 'dev':
    telemetry.trackCliCommandDev(userSuppliedSubCommand);
    func = require('./commands/dev').default;
    break;
  case 'deploy':
    func = require('./commands/deploy').default;
    break;
  // ... 其他命令
}

// 执行命令
exitCode = await func(client);
```

---

## 4. dev 命令执行流程

### 4.1 dev 命令入口

**`packages/cli/src/commands/dev/index.ts`:**

```typescript
import { devCommand } from './command';
import dev from './dev';

export default async function main(client: Client) {
  // 1. 检查递归调用保护
  if (process.env.__VERCEL_DEV_RUNNING) {
    output.error(
      `${cmd('vercel dev')} must not recursively invoke itself`
    );
    return 1;
  }

  // 2. 解析命令参数
  const { args, flags: opts } = parseArguments(client.argv.slice(2), {
    ...devCommand,
  });

  // 3. 读取本地配置
  const vercelConfig = await readConfig(cwd);
  
  // 4. 检查 package.json dev script 递归
  const pkg = await readJson<PackageJson>(join(cwd, 'package.json'));
  if (pkg?.scripts?.dev?.includes('vercel dev')) {
    output.warn('Your `dev` script may be calling `vercel dev` recursively');
  }

  // 5. 调用 dev() 函数
  return dev(client, opts, args, telemetry);
}
```

### 4.2 dev() 函数核心逻辑

**`packages/cli/src/commands/dev/dev.ts`:**

```typescript
export default async function dev(
  client: Client,
  opts: Partial<Options>,
  args: string[],
  telemetry: DevTelemetryClient
) {
  // 1. 解析目录和监听地址
  const [dir = '.'] = args;
  let cwd = resolve(dir);
  const listen = parseListen(opts['--listen'] || '3000');

  // 2. 获取项目链接状态
  let link = await getLinkedProject(client, cwd);

  // 3. 如果未链接，执行项目设置和链接
  if (link.status === 'not_linked' && !process.env.__VERCEL_SKIP_DEV_CMD) {
    link = await setupAndLink(client, cwd, {
      autoConfirm: opts['--yes'],
      link,
      successEmoji: 'link',
      setupMsg: 'Set up and develop',
    });

    if (link.status === 'not_linked') {
      return 0;  // 用户取消链接
    }
  }

  // 4. 处理错误状态
  if (link.status === 'error') {
    return link.exitCode;
  }

  // 5. 如果已链接，获取项目设置和环境变量
  let projectSettings: ProjectSettings | undefined;
  let envValues: Record<string, string> = {};
  let repoRoot: string | undefined;
  
  if (link.status === 'linked') {
    const { project, org } = link;
    
    // 更新 cwd 到仓库根目录
    if (link.repoRoot) {
      repoRoot = cwd = link.repoRoot;
    }
    
    projectSettings = project;
    
    // 如果项目有 rootDirectory 配置
    if (project.rootDirectory) {
      cwd = join(cwd, project.rootDirectory);
    }

    // 从 Vercel API 拉取环境变量
    envValues = (await pullEnvRecords(client, project.id, 'vercel-cli:dev')).env;
  }

  // 6. 检测多服务 (实验性功能)
  let services: ResolvedService[] | undefined;
  if (isExperimentalServicesEnabled()) {
    const result = await tryDetectServices(cwd);
    if (result && result.services.length > 0) {
      services = result.services;
    }
  }

  // 7. 创建 DevServer 实例
  const devServer = new DevServer(cwd, {
    projectSettings,
    envValues,
    repoRoot,
    services,
  });

  // 8. 设置 OIDC Token 刷新 (用于认证)
  const controller = new AbortController();
  const timeout = setTimeout(async () => {
    // 定期刷新 VERCEL_OIDC_TOKEN
    for await (const token of refreshOidcToken(...)) {
      envValues[VERCEL_OIDC_TOKEN] = token;
      await devServer.runDevCommand(true);
    }
  });

  // 9. 监听 SIGTERM 信号以优雅关闭
  process.on('SIGTERM', () => {
    clearTimeout(timeout);
    controller.abort();
    devServer.stop();
  });

  // 10. 启动 DevServer
  try {
    await devServer.start(...listen);
  } finally {
    clearTimeout(timeout);
    controller.abort();
  }
}
```

### 4.3 dev 命令执行流程图

```
vercel dev [dir]
      │
      ▼
┌─────────────────────────────────────────────────────────────┐
│               dev/index.ts (入口检查)                        │
├─────────────────────────────────────────────────────────────┤
│  检查 __VERCEL_DEV_RUNNING 防止递归调用                      │
│  解析命令参数 (--listen, --yes)                              │
│  读取 vercel.json 配置                                       │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│               dev/dev.ts (核心逻辑)                          │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │ 1. getLinkedProject(client, cwd)                     │   │
│  │    检查 .vercel/project.json 是否存在                 │   │
│  │    返回: { status: 'linked' | 'not_linked', ... }   │   │
│  └───────────────────────────┬─────────────────────────┘   │
│                              │                              │
│                              ▼                              │
│  ┌─────────────────────────────────────────────────────┐   │
│  │ 2. setupAndLink() (如果未链接)                        │   │
│  │    显示项目列表，让用户选择                           │   │
│  │    创建 .vercel/project.json                         │   │
│  └───────────────────────────┬─────────────────────────┘   │
│                              │                              │
│                              ▼                              │
│  ┌─────────────────────────────────────────────────────┐   │
│  │ 3. pullEnvRecords(client, projectId)                 │   │
│  │    从 Vercel API 拉取环境变量                         │   │
│  │    GET /v3/env/pull/{projectId}                      │   │
│  └───────────────────────────┬─────────────────────────┘   │
│                              │                              │
│                              ▼                              │
│  ┌─────────────────────────────────────────────────────┐   │
│  │ 4. new DevServer(cwd, options)                       │   │
│  │    创建开发服务器实例                                 │   │
│  │    传入 projectSettings, envValues                   │   │
│  └───────────────────────────┬─────────────────────────┘   │
│                              │                              │
│                              ▼                              │
│  ┌─────────────────────────────────────────────────────┐   │
│  │ 5. devServer.start(host, port)                       │   │
│  │    启动 HTTP 服务器，监听指定端口                     │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## 5. DevServer 初始化与启动

### 5.1 DevServer 类结构

**`packages/cli/src/util/dev/server.ts`:**

```typescript
export default class DevServer {
  // 公共属性
  public cwd: string;                        // 工作目录
  public repoRoot: string;                   // 仓库根目录
  public proxy: httpProxy;                   // HTTP 代理实例
  public envConfigs: EnvConfigs;             // 环境变量配置
  public files: BuilderInputs;               // 文件映射表
  public devCacheDir: string;                // 开发缓存目录

  // 私有属性
  private _address: URL | undefined;         // 服务器地址
  private server: http.Server;               // HTTP 服务器
  private buildMatches: Map<string, BuildMatch>;  // 构建匹配表
  private watcher?: FSWatcher;               // 文件监听器
  private devProcess?: ChildProcess;         // 框架 dev 进程
  private devProcessOrigin?: string;         // 框架 dev 服务器地址
  private projectSettings?: ProjectSettings; // 项目设置
  private shutdownCallbacks: Map<number, () => void>;  // 关闭回调
  
  constructor(cwd: string, options: DevServerOptions) {
    this.cwd = cwd;
    this.repoRoot = options.repoRoot || cwd;
    this.originalProjectSettings = options.projectSettings;
    this.envValues = options.envValues || {};
    this.services = options.services;
    
    // 初始化状态
    this.files = {};
    this.buildMatches = new Map();
    this.shutdownCallbacks = new Map();
    
    // 创建 HTTP 代理
    this.proxy = httpProxy.createProxyServer({
      ws: true,
      xfwd: true,
    });
  }
}
```

### 5.2 start() 方法详解

```typescript
async start(...listen: ListenSpec): Promise<void> {
  this.startPromise = this._start(...listen);
  await this.startPromise;
}

private async _start(host: string, port: number): Promise<void> {
  // 1. 验证目录存在
  if (!fs.existsSync(this.cwd)) {
    throw new Error(`Directory "${this.cwd}" does not exist`);
  }

  // 2. 加载 .vercelignore 过滤器
  const { ig } = await getVercelIgnore(this.cwd);
  this.filter = ig;

  // 3. 生成唯一的 Pod ID
  this.podId = randomBytes(2).toString('hex');

  // 4. 创建 HTTP 服务器
  this.server = http.createServer(this.devServerHandler);

  // 5. 监听端口 (自动递增如果占用)
  let portToTry = port;
  while (true) {
    try {
      const address = await listen(this.server, portToTry, host);
      this._address = new URL(address);
      break;
    } catch (err) {
      if (err.code === 'EADDRINUSE') {
        portToTry++;
        continue;
      }
      throw err;
    }
  }

  // 6. 获取并解析 vercel.json 配置
  const vercelConfig = await this.getVercelConfig();

  // 7. 扫描文件系统，构建文件映射
  const files = await getFiles(this.cwd, vercelConfig, this.filter);
  this.files = files;

  // 8. 更新构建匹配表
  await this.updateBuildMatches(vercelConfig, true);

  // 9. 执行初始的阻塞构建
  const blockingBuildsPromise = this.performInitialBuilds(vercelConfig);
  await blockingBuildsPromise;

  // 10. 启动文件监听器 (chokidar)
  this.watcher = watch(this.cwd, {
    ignored: this.filter,
    ignoreInitial: true,
    persistent: true,
    followSymlinks: false,
    depth: 100,
    atomic: 500,
  });
  
  this.watcher.on('add', (path) => this.handleFileEvent('add', path));
  this.watcher.on('change', (path) => this.handleFileEvent('change', path));
  this.watcher.on('unlink', (path) => this.handleFileEvent('unlink', path));

  // 11. 配置 WebSocket 升级处理
  this.server.on('upgrade', this.handleUpgrade);

  // 12. 输出 Ready 消息
  output.ready(`Available at ${link(this.address.href)}`);
}
```

### 5.3 DevServer 启动流程图

```
devServer.start(host, port)
      │
      ▼
┌─────────────────────────────────────────────────────────────┐
│                    _start() 方法                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │ 1. 验证目录存在                                       │   │
│  │    fs.existsSync(this.cwd)                           │   │
│  └───────────────────────────┬─────────────────────────┘   │
│                              │                              │
│                              ▼                              │
│  ┌─────────────────────────────────────────────────────┐   │
│  │ 2. 加载 .vercelignore                                │   │
│  │    getVercelIgnore(this.cwd)                         │   │
│  │    → 返回 ignore 实例用于过滤文件                     │   │
│  └───────────────────────────┬─────────────────────────┘   │
│                              │                              │
│                              ▼                              │
│  ┌─────────────────────────────────────────────────────┐   │
│  │ 3. 创建 HTTP 服务器并监听端口                         │   │
│  │    this.server = http.createServer(handler)          │   │
│  │    await listen(server, port, host)                  │   │
│  └───────────────────────────┬─────────────────────────┘   │
│                              │                              │
│                              ▼                              │
│  ┌─────────────────────────────────────────────────────┐   │
│  │ 4. 获取 vercel.json 配置                             │   │
│  │    await this.getVercelConfig()                      │   │
│  │    ├── compileVercelConfig() (编译 TS 配置)          │   │
│  │    ├── readJsonFile('vercel.json')                   │   │
│  │    ├── detectBuilders() (检测 builders)              │   │
│  │    ├── getLocalEnv('.env') (加载环境变量)            │   │
│  │    └── runDevCommand() (启动框架 dev 服务器)         │   │
│  └───────────────────────────┬─────────────────────────┘   │
│                              │                              │
│                              ▼                              │
│  ┌─────────────────────────────────────────────────────┐   │
│  │ 5. 扫描文件系统                                       │   │
│  │    await getFiles(cwd, vercelConfig, filter)         │   │
│  │    → 返回 { 'api/index.go': FileFsRef, ... }        │   │
│  └───────────────────────────┬─────────────────────────┘   │
│                              │                              │
│                              ▼                              │
│  ┌─────────────────────────────────────────────────────┐   │
│  │ 6. 更新构建匹配表                                     │   │
│  │    await this.updateBuildMatches(vercelConfig)       │   │
│  │    ├── getBuildMatches() 获取所有匹配                │   │
│  │    └── importBuilders() 导入对应的 builders          │   │
│  └───────────────────────────┬─────────────────────────┘   │
│                              │                              │
│                              ▼                              │
│  ┌─────────────────────────────────────────────────────┐   │
│  │ 7. 执行初始构建                                       │   │
│  │    await executeBuild(...) (阻塞构建)                │   │
│  └───────────────────────────┬─────────────────────────┘   │
│                              │                              │
│                              ▼                              │
│  ┌─────────────────────────────────────────────────────┐   │
│  │ 8. 启动文件监听器                                     │   │
│  │    this.watcher = chokidar.watch(cwd, options)       │   │
│  │    watcher.on('add' | 'change' | 'unlink', handler)  │   │
│  └───────────────────────────┬─────────────────────────┘   │
│                              │                              │
│                              ▼                              │
│  ┌─────────────────────────────────────────────────────┐   │
│  │ 9. 输出 Ready 消息                                    │   │
│  │    output.ready(`Available at http://localhost:3000`)│   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## 6. 配置文件加载

### 6.1 配置文件优先级

```
配置文件搜索顺序:
1. --local-config 命令行指定的路径
2. .vercel/vercel.json (编译后的 TS 配置)
3. vercel.json
4. now.json (已废弃，仅兼容)
```

### 6.2 _getVercelConfig() 方法

```typescript
async _getVercelConfig(): Promise<VercelConfig> {
  // 1. 编译 TypeScript 配置 (如果存在 vercel.config.ts)
  const { compileVercelConfig } = await import('../compile-vercel-config');
  await compileVercelConfig(this.cwd);
  
  // 2. 获取配置文件路径
  const configPath = getVercelConfigPath(this.cwd);

  // 3. 并行读取 package.json 和 vercel.json
  const [pkg, vercelConfig] = await Promise.all([
    this.readJsonFile<PackageJson>('package.json'),
    this.readJsonFile<VercelConfig>(configPath),
  ]);

  // 4. 验证配置格式
  await this.validateVercelConfig(vercelConfig);

  // 5. 转换路由配置
  const { error: routeError, routes } = getTransformedRoutes(vercelConfig);
  if (routeError) {
    output.prettyError(routeError);
  }

  // 6. Zero Config 检测 (如果没有 builds 配置)
  if (!vercelConfig.builds || vercelConfig.builds.length === 0) {
    const filesMap = this.resolveBuildFiles(this.files);
    
    // 使用 detectBuilders 自动检测
    const { builders, routes: defaultRoutes } = await detectBuilders(
      Object.keys(filesMap),
      pkg,
      {
        tag: 'latest',
        functions: vercelConfig.functions,
        projectSettings: this.projectSettings,
      }
    );
    
    vercelConfig.builds = builders;
    
    // 合并默认路由
    if (defaultRoutes) {
      vercelConfig.routes = appendRoutesToPhase({
        routes: vercelConfig.routes,
        newRoutes: defaultRoutes,
        phase: 'filesystem',
      });
    }
  }

  // 7. 加载本地环境变量
  const [runEnv, buildEnv] = await Promise.all([
    this.getLocalEnv('.env', vercelConfig.env),
    this.getLocalEnv('.env.build', vercelConfig.build?.env),
  ]);
  
  this.envConfigs = {
    runEnv,
    buildEnv,
    allEnv: { ...runEnv, ...buildEnv },
  };

  // 8. 启动框架 dev 命令 (如 next dev)
  await this.runDevCommand();

  return vercelConfig;
}
```

### 6.3 TypeScript 配置支持

**支持的配置文件扩展名:**

```typescript
// packages/cli/src/util/compile-vercel-config.ts
export const VERCEL_CONFIG_EXTENSIONS = ['ts', 'mts', 'js', 'mjs', 'cjs'];

// 搜索顺序: vercel.config.ts → vercel.config.mts → vercel.config.js → ...
```

**编译流程:**

```typescript
async function compileVercelConfig(workPath: string): Promise<void> {
  // 1. 查找配置文件
  const configFile = await findVercelConfigFile(workPath);
  if (!configFile) return;

  // 2. 使用 esbuild 编译
  const result = await esbuild.build({
    entryPoints: [configFile],
    bundle: true,
    platform: 'node',
    write: false,
    format: 'esm',
  });

  // 3. 执行编译后的代码获取配置对象
  const config = await executeCompiledConfig(result.outputFiles[0].text);

  // 4. 输出到 .vercel/vercel.json
  const outputPath = join(workPath, '.vercel', 'vercel.json');
  await writeFile(outputPath, JSON.stringify(config, null, 2));
}
```

---

## 7. Builder 检测与加载

### 7.1 Builder 检测逻辑

**`packages/fs-detectors/src/detect-builders.ts`:**

```typescript
// API 文件到 Builder 的映射规则
function getApiMatches(): Builder[] {
  return [
    { src: 'api/**/*.+(js|mjs|ts|tsx)', use: '@vercel/node' },
    { src: 'api/**/!(*_test).go',       use: '@vercel/go' },
    { src: 'api/**/*.py',               use: '@vercel/python' },
    { src: 'api/**/*.rb',               use: '@vercel/ruby' },
    { src: 'api/**/*.rs',               use: '@vercel/rust' },
  ];
}

export async function detectBuilders(
  files: string[],
  pkg: PackageJson | null,
  options: DetectOptions
): Promise<DetectBuildersResult> {
  const builders: Builder[] = [];
  const apiDirectory = detectApiDirectory(files);
  
  // 遍历文件，匹配 API builders
  for (const file of files) {
    const builder = maybeGetApiBuilder(file, apiDirectory);
    if (builder) {
      builders.push(builder);
    }
  }
  
  // 检测前端 builder (基于框架)
  const frontendBuilder = await detectFrontendBuilder(pkg, files, options);
  if (frontendBuilder) {
    builders.push(frontendBuilder);
  }
  
  // 生成默认路由
  const routes = generateDefaultRoutes(builders, options);
  
  return { builders, routes };
}
```

### 7.2 getBuildMatches() 获取构建匹配

**`packages/cli/src/util/dev/builder.ts`:**

```typescript
export async function getBuildMatches(
  vercelConfig: VercelConfig,
  cwd: string,
  devServer: DevServer,
  fileList: string[]
): Promise<BuildMatch[]> {
  // 1. 获取 builds 配置 (或默认为静态文件)
  const builds = vercelConfig.builds || [{ src: '**', use: '@vercel/static' }];
  
  // 2. 收集所有需要的 builder 规格
  const builderSpecs = new Set<string>();
  for (const build of builds) {
    builderSpecs.add(build.use);
  }
  
  // 3. 导入所有 builders
  const buildersWithPkgs = await importBuilders([...builderSpecs], cwd);
  
  // 4. 匹配文件到 builder
  const matches: BuildMatch[] = [];
  
  for (const buildConfig of builds) {
    const { src, use, config } = buildConfig;
    const builderWithPkg = buildersWithPkgs.get(use);
    
    // 使用 minimatch 匹配文件
    const matchedFiles = fileList.filter(name => minimatch(name, src));
    
    for (const file of matchedFiles) {
      matches.push({
        src,
        use,
        entrypoint: file,
        config,
        builderWithPkg,
        buildOutput: {},
        buildResults: new Map(),
      });
    }
  }
  
  return matches;
}
```

### 7.3 importBuilders() 导入 Builders

**`packages/cli/src/util/build/import-builders.ts`:**

```typescript
export async function importBuilders(
  builderSpecs: string[],
  cwd: string
): Promise<Map<string, BuilderWithPkg>> {
  const buildersDir = join(cwd, '.vercel', 'builders');
  const builders = new Map<string, BuilderWithPkg>();
  
  for (const spec of builderSpecs) {
    // 1. 解析 builder 规格
    const parsed = npa(spec);  // npm-package-arg
    
    // 2. 尝试解析已安装的 builder
    let requirePath: string;
    try {
      // 首先尝试从 .vercel/builders/node_modules 解析
      requirePath = require.resolve(spec, { paths: [buildersDir] });
    } catch (err) {
      // 然后尝试从 CLI 内置依赖解析
      try {
        requirePath = require.resolve(spec);
      } catch (err2) {
        // 需要安装
        await installBuilder(buildersDir, spec);
        requirePath = require.resolve(spec, { paths: [buildersDir] });
      }
    }
    
    // 3. 导入 builder 模块
    const builder = await import(requirePath);
    const pkg = await readPackageJson(dirname(requirePath));
    
    builders.set(spec, {
      path: requirePath,
      builder: builder.default || builder,
      pkg,
    });
  }
  
  return builders;
}
```

### 7.4 Builder 加载优先级

```
Builder 查找顺序:
1. .vercel/builders/node_modules/@vercel/go  (项目级)
2. CLI 内置依赖 (node_modules/@vercel/go)
3. 动态安装到 .vercel/builders/

示例:
当检测到 api/hello.go 文件时:
  → 匹配规则: 'api/**/!(*_test).go' → '@vercel/go'
  → 查找 @vercel/go builder
  → 调用 builder.startDevServer() 或 builder.build()
```

---

## 8. HTTP 请求处理流程

### 8.1 请求处理器入口

```typescript
// DevServer 的 HTTP 请求处理器
devServerHandler = async (
  req: http.IncomingMessage,
  res: http.ServerResponse
): Promise<void> => {
  // 等待 DevServer 完全启动
  await this.startPromise;
  
  // 生成请求 ID
  const requestId = generateRequestId(this.podId);
  
  // 获取配置
  const vercelConfig = await this.getVercelConfig();
  
  // 处理请求
  await this.serveProjectAsNowV2(req, res, requestId, vercelConfig);
};
```

### 8.2 serveProjectAsNowV2() 核心路由逻辑

```typescript
async serveProjectAsNowV2(
  req: http.IncomingMessage,
  res: http.ServerResponse,
  requestId: string,
  vercelConfig: VercelConfig,
  routes?: Route[],
  callLevel: number = 0
): Promise<void> {
  // 1. 清理 URL (移除双斜杠等)
  req.url = cleanUrl(req.url);

  // 2. 多服务模式路由 (实验性)
  if (this.orchestrator && callLevel === 0) {
    const serviceMatch = await this.orchestrator.matchService(req);
    if (serviceMatch) {
      return this.proxyToService(req, res, serviceMatch);
    }
  }

  // 3. 更新构建匹配表
  await this.updateBuildMatches(vercelConfig);

  // 4. 等待阻塞构建完成
  if (this.blockingBuildsPromise) {
    await this.blockingBuildsPromise;
  }

  // 5. 准备路由配置
  const { routes: configRoutes = [] } = vercelConfig;
  const allRoutes = routes || configRoutes;
  
  // 分离路由阶段
  const { filesystemRoutes, missRoutes, hitRoutes, errorRoutes } = 
    getRoutesTypes(allRoutes);

  // 6. 路由匹配循环 (两个阶段: null → filesystem)
  let match: BuildMatch | null = null;
  let routeResult: RouteResult | undefined;
  
  for (const phase of [null, 'filesystem'] as const) {
    // 确定当前阶段的路由
    const currentRoutes = phase === 'filesystem' 
      ? filesystemRoutes 
      : allRoutes;

    // 执行路由匹配
    routeResult = await devRouter(
      req.url,
      req.method,
      currentRoutes,
      this,
      vercelConfig
    );

    // 查找对应的 BuildMatch
    match = await findBuildMatch(
      this.buildMatches,
      this.files,
      routeResult.dest,
      this,
      vercelConfig
    );

    if (match) {
      break;  // 找到匹配，结束循环
    }
  }

  // 7. 处理未匹配的情况
  if (!match) {
    // 如果有框架 dev 服务器，代理到它
    if (this.devProcessOrigin) {
      return proxyPass(req, res, this.devProcessOrigin, this, requestId);
    }
    
    // 返回 404
    return this.send404(req, res, requestId);
  }

  // 8. 检查 builder 是否支持 startDevServer()
  const { builder, pkg: builderPkg } = match.builderWithPkg;
  
  if (builder.version === 3 && typeof builder.startDevServer === 'function') {
    // 调用 builder.startDevServer()
    const devServerResult = await builder.startDevServer({
      files: this.files,
      entrypoint: match.entrypoint,
      workPath: this.cwd,
      config: match.config || {},
      meta: {
        isDev: true,
        requestPath,
        env: this.envConfigs.runEnv,
      },
    });

    if (devServerResult) {
      const { port, pid, shutdown } = devServerResult;
      this.shutdownCallbacks.set(pid, shutdown);
      
      // 请求完成后关闭子进程
      res.once('close', () => this.killBuilderDevServer(pid));
      
      // 代理请求到 builder 的 dev 服务器
      return proxyPass(req, res, `http://127.0.0.1:${port}`, this, requestId);
    }
  }

  // 9. 没有 startDevServer，需要触发构建
  await this.triggerBuild(match, requestPath, req, vercelConfig);

  // 10. 获取构建输出并返回
  const foundAsset = findAsset(match, requestPath, vercelConfig);
  
  switch (foundAsset.asset.type) {
    case 'FileFsRef':
      return serveStaticFile(req, res, foundAsset.asset.fsPath);
    
    case 'Lambda':
      return this.invokeLambda(req, res, foundAsset.asset, requestId);
  }
}
```

### 8.3 请求处理流程图

```
HTTP 请求到达 DevServer
      │
      ▼
┌─────────────────────────────────────────────────────────────┐
│              devServerHandler()                              │
├─────────────────────────────────────────────────────────────┤
│  await this.startPromise   // 确保服务器已启动               │
│  requestId = generateRequestId()                            │
│  vercelConfig = await this.getVercelConfig()                │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│              serveProjectAsNowV2()                           │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌────────────────────────────────────────────────────┐    │
│  │ 1. devRouter() - 路由匹配                           │    │
│  │    使用 PCRE 正则匹配 vercel.json routes 配置       │    │
│  │    返回 { dest, headers, status, ... }             │    │
│  └──────────────────────────┬─────────────────────────┘    │
│                             │                               │
│                             ▼                               │
│  ┌────────────────────────────────────────────────────┐    │
│  │ 2. findBuildMatch() - 查找 Builder 匹配             │    │
│  │    在 this.buildMatches 中查找对应的 builder        │    │
│  │    例如: api/hello.go → @vercel/go                 │    │
│  └──────────────────────────┬─────────────────────────┘    │
│                             │                               │
│                             ▼                               │
│  ┌────────────────────────────────────────────────────┐    │
│  │ 3. 判断处理方式                                     │    │
│  │                                                     │    │
│  │  ├── 有 builder.startDevServer() ?                 │    │
│  │  │   │                                              │    │
│  │  │   ├── Yes → 调用 startDevServer()               │    │
│  │  │   │         代理请求到 builder dev server       │    │
│  │  │   │                                              │    │
│  │  │   └── No → 触发构建 (triggerBuild)              │    │
│  │  │            返回构建产物                          │    │
│  │  │                                                  │    │
│  │  └── 无匹配的 builder ?                            │    │
│  │      │                                              │    │
│  │      ├── 有 devProcessOrigin ?                     │    │
│  │      │   └── 代理到框架 dev server                 │    │
│  │      │                                              │    │
│  │      └── 返回 404                                  │    │
│  └────────────────────────────────────────────────────┘    │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## 9. Go 语言 Builder 调用

### 9.1 Go Builder startDevServer() 调用

当请求匹配到 `api/*.go` 文件时，DevServer 会调用 `@vercel/go` 的 `startDevServer()` 方法：

```typescript
// DevServer 调用 builder.startDevServer()
const devServerResult = await builder.startDevServer({
  files: this.files,           // 文件映射表
  entrypoint: match.entrypoint, // 如 'api/hello.go'
  workPath: this.cwd,          // 工作目录
  config: match.config || {},  // vercel.json 中的配置
  repoRootPath: this.repoRoot,
  meta: {
    isDev: true,
    requestPath,               // 请求路径
    devCacheDir,               // 缓存目录
    env: this.envConfigs.runEnv,     // 运行时环境变量
    buildEnv: this.envConfigs.buildEnv, // 构建时环境变量
  },
});

// devServerResult 结构:
// {
//   port: 54321,        // Go 进程监听的端口
//   pid: 12345,         // Go 进程 PID
//   shutdown: () => {}, // 关闭回调
// }
```

### 9.2 @vercel/go startDevServer() 实现

**`packages/go/src/index.ts`:**

```typescript
export async function startDevServer(options: StartDevServerOptions): Promise<StartDevServerResult> {
  const { entrypoint, workPath, config, meta = {} } = options;

  // 1. 判断运行模式
  if (config?.framework === 'go' || config?.framework === 'services') {
    // Standalone Server 模式
    return startStandaloneDevServer({ entrypoint, workPath, config, meta });
  }

  // 2. Serverless Function 模式
  // 分析入口文件
  const analyzed = await getAnalyzedEntrypoint({
    entrypoint,
    workPath,
  });

  // 3. 创建 Go 工具包装器
  const go = await createGo({ workPath, opts: { env: meta.env } });

  // 4. 生成 dev-server.go 包装器
  const devServerPath = join(workPath, '.vercel', 'cache', 'dev-server.go');
  await writeDevServerGo(devServerPath, analyzed);

  // 5. 编译并运行
  const child = spawn(go.binaryPath, ['run', devServerPath, entrypoint], {
    cwd: workPath,
    env: meta.env,
    stdio: ['ignore', 'inherit', 'inherit', 'pipe'],  // FD 3 用于通信
  });

  // 6. 等待端口号
  const port = await new Promise<number>((resolve, reject) => {
    child.stdio[3].on('data', (data) => {
      resolve(parseInt(data.toString().trim()));
    });
    child.on('error', reject);
    child.on('exit', (code) => {
      reject(new Error(`Go process exited with code ${code}`));
    });
  });

  return {
    port,
    pid: child.pid,
    shutdown: () => child.kill(),
  };
}
```

### 9.3 Go Dev 两种模式详解

#### Serverless Function 模式

```
请求: GET /api/hello
      │
      ▼
DevServer 检测到 api/hello.go
      │
      ▼
调用 @vercel/go startDevServer()
      │
      ├── 1. 分析 api/hello.go (AST)
      │      提取: packageName='handler', functionName='Handler'
      │
      ├── 2. 生成 dev-server.go 包装器
      │      - 导入用户包
      │      - 创建 HTTP 服务器
      │      - 监听随机端口
      │      - 通过 FD 3 返回端口号
      │
      ├── 3. 执行 go run dev-server.go api/hello.go
      │
      ├── 4. 等待端口号 (从 FD 3 读取)
      │
      └── 5. 返回 { port, pid, shutdown }
      │
      ▼
DevServer 代理请求到 http://127.0.0.1:{port}
      │
      ▼
Go 进程处理请求并返回响应
      │
      ▼
DevServer 返回响应给客户端
      │
      ▼
关闭 Go 进程 (调用 shutdown)
```

#### Standalone Server 模式

```
请求: GET /any/path
      │
      ▼
DevServer 检测到 main.go (framework: 'go')
      │
      ▼
调用 @vercel/go startDevServer()
      │
      ├── 1. 检测入口点 (main.go 或 cmd/app/main.go)
      │
      ├── 2. 执行 go run . 或 go run ./cmd/app
      │      设置环境变量 PORT=随机端口
      │
      ├── 3. 等待端口就绪 (轮询健康检查)
      │
      └── 4. 返回 { port, pid, shutdown }
      │
      ▼
DevServer 代理请求到 http://127.0.0.1:{port}
      │
      ▼
用户的 Go 服务器处理请求
      │
      ▼
DevServer 返回响应给客户端
      │
      ▼
(Standalone 模式的进程会持续运行，不会每次请求后关闭)
```

### 9.4 dev-server.go 模板解析

**`packages/go/dev-server.go`:**

```go
package main

import (
    "encoding/json"
    "fmt"
    "net"
    "net/http"
    "os"

    // 动态替换为用户包
    __VC_HANDLER_PACKAGE_NAME "example.com/myapp/api/hello"
)

type devServerInfo struct {
    Port int `json:"port"`
}

func main() {
    // 1. 监听随机端口
    listener, err := net.Listen("tcp", ":0")
    if err != nil {
        fmt.Fprintln(os.Stderr, "Failed to listen:", err)
        os.Exit(1)
    }

    // 2. 获取分配的端口号
    port := listener.Addr().(*net.TCPAddr).Port

    // 3. 通过 FD 3 将端口号发送给父进程
    info := devServerInfo{Port: port}
    fd3 := os.NewFile(3, "pipe")
    json.NewEncoder(fd3).Encode(info)
    fd3.Close()

    // 4. 创建 HTTP 服务器
    http.HandleFunc("/", __VC_HANDLER_FUNC_NAME)  // 如 handler.Handler

    // 5. 启动服务
    http.Serve(listener, nil)
}
```

### 9.5 Go 请求处理完整时序图

```
┌────────────┐      ┌────────────┐      ┌────────────┐      ┌────────────┐
│   Client   │      │ DevServer  │      │ @vercel/go │      │ Go Process │
└─────┬──────┘      └─────┬──────┘      └─────┬──────┘      └─────┬──────┘
      │                   │                   │                   │
      │  GET /api/hello   │                   │                   │
      │──────────────────►│                   │                   │
      │                   │                   │                   │
      │                   │  devRouter()      │                   │
      │                   │  找到匹配:        │                   │
      │                   │  api/hello.go     │                   │
      │                   │                   │                   │
      │                   │  startDevServer() │                   │
      │                   │──────────────────►│                   │
      │                   │                   │                   │
      │                   │                   │  分析 AST         │
      │                   │                   │  生成 dev-server.go
      │                   │                   │                   │
      │                   │                   │  spawn('go', ['run', ...])
      │                   │                   │──────────────────►│
      │                   │                   │                   │
      │                   │                   │                   │ 编译
      │                   │                   │                   │ 启动
      │                   │                   │                   │
      │                   │                   │  FD 3: port=54321│
      │                   │                   │◄──────────────────│
      │                   │                   │                   │
      │                   │  { port: 54321 }  │                   │
      │                   │◄──────────────────│                   │
      │                   │                   │                   │
      │                   │  proxyPass()      │                   │
      │                   │  GET http://127.0.0.1:54321/api/hello │
      │                   │───────────────────────────────────────►
      │                   │                   │                   │
      │                   │                   │                   │ Handler()
      │                   │                   │                   │ 处理请求
      │                   │                   │                   │
      │                   │  HTTP Response    │                   │
      │                   │◄───────────────────────────────────────
      │                   │                   │                   │
      │  HTTP Response    │                   │                   │
      │◄──────────────────│                   │                   │
      │                   │                   │                   │
      │                   │  shutdown()       │                   │
      │                   │───────────────────────────────────────►
      │                   │                   │                   │ SIGTERM
      │                   │                   │                   │ 进程退出
      │                   │                   │                   │
```

---

## 10. 完整调用链路图

### 10.1 从命令行到请求响应的完整链路

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                                                                                  │
│   $ vercel dev                                                                   │
│                                                                                  │
└─────────────────────────────────────────┬────────────────────────────────────────┘
                                          │
                                          ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                           packages/cli/src/index.ts                              │
│                                                                                  │
│   main()                                                                         │
│   ├── parseArguments()           // 解析 CLI 参数                                │
│   ├── getConfig()                // 读取 vercel.json                             │
│   ├── new Client()               // 创建 API 客户端                              │
│   └── switch('dev')              // 路由到 dev 命令                              │
│       └── require('./commands/dev').default(client)                              │
│                                                                                  │
└─────────────────────────────────────────┬────────────────────────────────────────┘
                                          │
                                          ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                      packages/cli/src/commands/dev/index.ts                      │
│                                                                                  │
│   default(client)                                                                │
│   ├── 检查 __VERCEL_DEV_RUNNING  // 防止递归                                     │
│   ├── parseArguments()           // 解析 dev 命令参数                            │
│   └── dev(client, opts, args)                                                    │
│                                                                                  │
└─────────────────────────────────────────┬────────────────────────────────────────┘
                                          │
                                          ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                        packages/cli/src/commands/dev/dev.ts                      │
│                                                                                  │
│   dev(client, opts, args)                                                        │
│   ├── getLinkedProject()         // 检查 .vercel/project.json                    │
│   ├── setupAndLink()             // 如果未链接，执行链接                          │
│   ├── pullEnvRecords()           // 从 API 拉取环境变量                          │
│   ├── new DevServer(cwd, options)                                                │
│   └── devServer.start(host, port)                                                │
│                                                                                  │
└─────────────────────────────────────────┬────────────────────────────────────────┘
                                          │
                                          ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                       packages/cli/src/util/dev/server.ts                        │
│                                                                                  │
│   DevServer._start(host, port)                                                   │
│   ├── getVercelIgnore()          // 加载忽略规则                                 │
│   ├── http.createServer()        // 创建 HTTP 服务器                             │
│   ├── listen()                   // 监听端口                                     │
│   ├── getVercelConfig()          // 加载配置                                     │
│   │   ├── compileVercelConfig()  // 编译 TS 配置                                 │
│   │   ├── detectBuilders()       // 检测 builders                                │
│   │   ├── getLocalEnv()          // 加载 .env                                    │
│   │   └── runDevCommand()        // 启动框架 dev                                 │
│   ├── getFiles()                 // 扫描文件系统                                 │
│   ├── updateBuildMatches()       // 更新构建匹配                                 │
│   │   ├── getBuildMatches()      // 获取匹配列表                                 │
│   │   └── importBuilders()       // 导入 builders                                │
│   ├── executeBuild()             // 初始构建                                     │
│   └── chokidar.watch()           // 启动文件监听                                 │
│                                                                                  │
│   output.ready('Available at http://localhost:3000')                             │
│                                                                                  │
└─────────────────────────────────────────┬────────────────────────────────────────┘
                                          │
                                          │  HTTP 请求到达
                                          │
                                          ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                              devServerHandler()                                  │
│                                                                                  │
│   serveProjectAsNowV2(req, res, requestId, vercelConfig)                         │
│   ├── devRouter()                // 路由匹配                                     │
│   ├── findBuildMatch()           // 查找 builder                                 │
│   │                                                                              │
│   │   如果找到 @vercel/go:                                                       │
│   │   ├── builder.startDevServer()                                              │
│   │   │   ├── 分析入口文件 (AST)                                                 │
│   │   │   ├── 生成 dev-server.go                                                │
│   │   │   ├── spawn('go', ['run', ...])                                         │
│   │   │   └── 返回 { port, pid }                                                │
│   │   │                                                                          │
│   │   └── proxyPass(req, res, `http://127.0.0.1:${port}`)                        │
│   │                                                                              │
│   │   如果没有匹配:                                                               │
│   │   └── proxyPass(req, res, devProcessOrigin) 或 send404()                     │
│   │                                                                              │
│   └── 返回响应                                                                    │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### 10.2 关键函数调用树

```
vercel dev
│
├── main() [src/index.ts]
│   ├── parseArguments()
│   ├── getConfig()
│   ├── new Client()
│   └── dev(client) [src/commands/dev/index.ts]
│
├── dev() [src/commands/dev/dev.ts]
│   ├── getLinkedProject()
│   ├── setupAndLink()
│   ├── pullEnvRecords()
│   └── devServer.start()
│
├── DevServer._start() [src/util/dev/server.ts]
│   ├── getVercelIgnore()
│   ├── http.createServer()
│   ├── listen()
│   ├── getVercelConfig()
│   │   ├── compileVercelConfig()
│   │   ├── readJsonFile()
│   │   ├── detectBuilders()
│   │   ├── getLocalEnv()
│   │   └── runDevCommand()
│   ├── getFiles()
│   ├── updateBuildMatches()
│   │   ├── getBuildMatches()
│   │   └── importBuilders()
│   ├── executeBuild()
│   └── chokidar.watch()
│
└── HTTP Request Handling
    │
    ├── devServerHandler()
    │   └── serveProjectAsNowV2()
    │       ├── devRouter()
    │       ├── findBuildMatch()
    │       ├── builder.startDevServer() [@vercel/go]
    │       │   ├── getAnalyzedEntrypoint()
    │       │   ├── createGo()
    │       │   ├── writeDevServerGo()
    │       │   └── spawn('go', ['run', ...])
    │       └── proxyPass()
    │
    └── Response sent to client
```

---

## 11. 关键源文件索引

### 11.1 CLI 核心文件

| 文件路径 | 说明 |
|---------|------|
| `packages/cli/src/index.ts` | CLI 主入口，命令分发 |
| `packages/cli/src/commands/index.ts` | 命令注册表 |
| `packages/cli/src/commands/dev/command.ts` | dev 命令定义 |
| `packages/cli/src/commands/dev/index.ts` | dev 命令入口 |
| `packages/cli/src/commands/dev/dev.ts` | dev 命令核心逻辑 |

### 11.2 DevServer 相关

| 文件路径 | 说明 |
|---------|------|
| `packages/cli/src/util/dev/server.ts` | DevServer 类 (2738 行) |
| `packages/cli/src/util/dev/builder.ts` | 构建执行器 |
| `packages/cli/src/util/dev/router.ts` | 请求路由器 |
| `packages/cli/src/util/dev/types.ts` | 类型定义 |

### 11.3 配置与项目链接

| 文件路径 | 说明 |
|---------|------|
| `packages/cli/src/util/config/read-config.ts` | 配置读取 |
| `packages/cli/src/util/compile-vercel-config.ts` | TS 配置编译 |
| `packages/cli/src/util/projects/link.ts` | 项目链接管理 |
| `packages/cli/src/util/get-config.ts` | 全局配置获取 |

### 11.4 Builder 相关

| 文件路径 | 说明 |
|---------|------|
| `packages/cli/src/util/build/import-builders.ts` | Builder 导入 |
| `packages/fs-detectors/src/detect-builders.ts` | Builder 检测 |
| `packages/go/src/index.ts` | Go Builder 实现 |
| `packages/go/dev-server.go` | Go dev 服务器模板 |

### 11.5 环境变量

| 文件路径 | 说明 |
|---------|------|
| `packages/cli/src/util/env/get-env-records.ts` | 环境变量拉取 |
| `packages/cli/src/util/env/refresh-oidc-token.ts` | OIDC Token 刷新 |

---

## 附录

### A. 环境变量

| 变量名 | 说明 |
|--------|------|
| `__VERCEL_DEV_RUNNING` | 递归调用保护标志 |
| `VERCEL` | 标识 Vercel 环境 (`1`) |
| `VERCEL_ENV` | 环境类型 (`development`) |
| `NOW_REGION` | 区域标识 (`dev1`) |
| `VERCEL_OIDC_TOKEN` | OIDC 认证 Token |
| `PORT` | Go Standalone 模式监听端口 |

### B. 目录结构

```
.vercel/
├── project.json           # 项目链接信息
├── repo.json              # Monorepo 链接信息
├── vercel.json            # 编译后的配置
├── builders/              # 安装的 builders
│   ├── package.json
│   └── node_modules/
│       └── @vercel/go/
└── cache/                 # 构建缓存
    ├── golang/            # Go SDK 和模块缓存
    └── dev-server.go      # 生成的 dev 服务器
```

### C. 命令速查

```bash
# 启动开发服务器
vercel dev

# 指定端口
vercel dev --listen 8080

# 指定目录
vercel dev ./my-project

# 跳过链接确认
vercel dev --yes

# 调试模式
vercel dev --debug
```

---

*本文档基于 Vercel CLI 源码分析生成*
