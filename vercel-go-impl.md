# Vercel Go 语言支持完整实现指南

> 本文档详细分析 `@vercel/go` 包的完整实现，包含每个方法的解析、关键点说明，旨在帮助你实现一套相同的 Go Functions 的 dev 和 build 支持。

---

## 一、整体架构概览

### 1.1 核心文件结构

```
packages/go/
├── src/
│   ├── index.ts              # 主入口，导出 build/startDevServer/prepareCache
│   ├── go-helpers.ts         # Go 版本管理、编译辅助、AST 分析
│   ├── standalone-server.ts  # 独立服务器模式（framework='go'）
│   └── entrypoint.ts         # 入口点检测
├── dev-server.go             # Dev 模式的 Go 模板（Serverless 模式）
├── main.go                   # Build 时的 main 函数模板
├── vc-init.go                # Standalone 模式的 Bootstrap 包装器
├── analyze.go                # Go AST 分析工具（编译成二进制）
└── package.json
```

### 1.2 两种运行模式

```
┌─────────────────────────────────────────────────────────────────────┐
│                     @vercel/go 两种模式                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  模式 1: Serverless 模式 (默认)                                       │
│  ├── 适用：api/*.go 文件，导出 http.HandlerFunc                       │
│  ├── Build：生成 bootstrap 二进制，使用 Lambda runtime                │
│  └── Dev：编译并运行 dev-server.go 模板                               │
│                                                                      │
│  模式 2: Standalone 模式 (framework='go')                            │
│  ├── 适用：完整的 Go HTTP 服务器（main.go）                           │
│  ├── Build：生成 executable + user-server，使用 executable runtime   │
│  └── Dev：直接运行 go run，通过 PORT 环境变量通信                      │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 二、Builder API 接口规范

### 2.1 必须导出的接口

```typescript
// 版本号（固定为 3）
export const version = 3;

// Build 函数：用于 vercel build 命令
export async function build(options: BuildOptions): Promise<BuildResult>;

// Dev Server 函数：用于 vercel dev 命令
export async function startDevServer(options: StartDevServerOptions): Promise<StartDevServerResult>;

// 缓存准备函数：用于持久化缓存
export async function prepareCache(options: PrepareCacheOptions): Promise<Files>;

// 可选：shouldServe 函数
export { shouldServe } from '@vercel/build-utils';
```

### 2.2 核心类型定义

```typescript
// BuildOptions - build() 函数的输入参数
interface BuildOptions {
  files: Files;              // 所有项目文件
  entrypoint: string;        // 入口文件路径 (如 'api/hello.go')
  workPath: string;          // 工作目录绝对路径
  repoRootPath: string;      // 仓库根目录
  config: Config;            // 用户配置 (vercel.json)
  meta?: Meta;               // 元数据 (isDev, env 等)
}

// Config - 用户配置
interface Config {
  framework?: string;        // 框架标识 ('go', 'services', etc)
  includeFiles?: string[];   // 包含的额外文件
  // ...
}

// StartDevServerResult - startDevServer() 的返回值
interface StartDevServerResult {
  port: number;              // Dev 服务器监听的端口
  pid: number;               // 进程 PID
}

// BuildResult - build() 的返回值
interface BuildResult {
  output: Lambda;            // 或 { [key: string]: Lambda }
}
```

---

## 三、Serverless 模式完整实现

### 3.1 `build()` 函数完整流程

```
build(options)
    │
    ├─ 1. 准备工作目录
    │     └─ download(files, downloadPath, meta)
    │
    ├─ 2. 设置交叉编译环境
    │     └─ GOOS=linux, GOARCH=amd64
    │
    ├─ 3. 检查运行模式
    │     ├─ framework='go' → buildStandaloneServer()
    │     └─ 默认 → 继续 Serverless 流程
    │
    ├─ 4. 处理路径段文件名 ([param].go)
    │     └─ getRenamedEntrypoint()
    │
    ├─ 5. 查找 go.mod
    │     └─ findGoModPath()
    │
    ├─ 6. 分析 Go 入口文件
    │     └─ getAnalyzedEntrypoint() → { packageName, functionName }
    │
    ├─ 7. 重命名 Handler 函数
    │     └─ renameHandlerFunction()
    │
    ├─ 8. 创建 Go 编译器包装器
    │     └─ createGo() → GoWrapper
    │
    ├─ 9. 根据 packageName 选择构建方式
    │     ├─ packageName === 'main' → buildHandlerAsPackageMain()
    │     └─ 其他 → buildHandlerWithGoMod()
    │
    ├─ 10. 创建 Lambda 对象
    │      └─ new Lambda({ files, handler: 'bootstrap', runtime })
    │
    └─ 11. 清理临时文件
          └─ cleanupFileSystem()
```

### 3.2 `build()` 函数详细实现

```typescript
// packages/go/src/index.ts

export const version = 3;

export async function build(options: BuildOptions) {
  const { files, config, workPath, meta = {} } = options;
  let { entrypoint } = options;

  // 1. 准备工作目录
  const goPath = await getWriteableDirectory();
  const srcPath = join(goPath, 'src', 'lambda');
  const downloadPath = meta.skipDownload ? workPath : srcPath;
  await download(files, downloadPath, meta);

  // 2. 跟踪需要撤销的文件操作
  const undo: UndoActions = {
    fileActions: [],
    directoryCreation: [],
    functionRenames: [],
  };

  // 3. 设置交叉编译环境变量
  const env = cloneEnv(process.env, meta.env, {
    GOARCH: 'amd64',
    GOOS: 'linux',
  });

  try {
    // 4. 处理 Git 私有仓库凭证
    if (env.GIT_CREDENTIALS) {
      await initPrivateGit(env.GIT_CREDENTIALS);
    }

    // 5. Standalone 模式分支
    if (config?.framework === 'go' || config?.framework === 'services') {
      const resolvedEntrypoint = await detectGoEntrypoint(workPath, entrypoint);
      return buildStandaloneServer({ ...options, entrypoint: resolvedEntrypoint });
    }

    // 6. 处理路径段文件名 ([id].go → now-bracket[id].go)
    const renamedEntrypoint = getRenamedEntrypoint(entrypoint);
    if (renamedEntrypoint) {
      await move(join(workPath, entrypoint), join(workPath, renamedEntrypoint));
      undo.fileActions.push({ to: join(workPath, entrypoint), from: join(workPath, renamedEntrypoint) });
      entrypoint = renamedEntrypoint;
    }

    // 7. 查找 go.mod 文件
    const entrypointAbsolute = join(workPath, entrypoint);
    const entrypointDirname = dirname(entrypointAbsolute);
    const { goModPath, isGoModInRootDir } = await findGoModPath(entrypointDirname, workPath);

    // 8. 分析 Go 入口文件 AST
    const analyzed = await getAnalyzedEntrypoint({
      entrypoint,
      modulePath: goModPath ? dirname(goModPath) : undefined,
      workPath,
    });

    // 9. 验证 package name
    const packageName = analyzed.packageName;
    if (goModPath && packageName === 'main') {
      throw new Error('Please change `package main` to `package handler`');
    }

    // 10. 重命名 Handler 函数（避免冲突）
    const originalFunctionName = analyzed.functionName;
    const handlerFunctionName = getNewHandlerFunctionName(originalFunctionName, entrypoint);
    await renameHandlerFunction(entrypointAbsolute, originalFunctionName, handlerFunctionName);
    undo.functionRenames.push({
      fsPath: originalEntrypointAbsolute,
      from: handlerFunctionName,
      to: originalFunctionName,
    });

    // 11. 创建 GoWrapper 实例
    const go = await createGo({
      modulePath: goModPath ? dirname(goModPath) : undefined,
      opts: { cwd: entrypointDirname, env },
      workPath,
    });

    // 12. 执行构建
    const outDir = await getWriteableDirectory();
    if (packageName === 'main') {
      await buildHandlerAsPackageMain({ /* ... */ });
    } else {
      await buildHandlerWithGoMod({ /* ... */ });
    }

    // 13. 创建 Lambda
    const runtime = await getProvidedRuntime();  // 'provided.al2023'
    const lambda = new Lambda({
      files: { ...(await glob('**', outDir)), ...includedFiles },
      handler: 'bootstrap',      // 入口二进制名
      runtime,                   // Lambda 运行时
      runtimeLanguage: 'go',
      supportsWrapper: true,
      environment: {},
    });

    return { output: lambda };
  } finally {
    await cleanupFileSystem(undo);
  }
}
```

### 3.3 关键方法解析

#### 3.3.1 `findGoModPath()` - 查找 go.mod 文件

```typescript
/**
 * 从入口目录向上查找 go.mod 文件
 * @param entrypointDir 入口文件所在目录
 * @param workPath 项目根目录
 * @returns { goModPath, isGoModInRootDir }
 */
async function findGoModPath(entrypointDir: string, workPath: string) {
  let goModPath: string | undefined = undefined;
  let isGoModInRootDir = false;
  let dir = entrypointDir;

  // 从入口目录向上遍历，直到项目根目录
  while (!isGoModInRootDir) {
    isGoModInRootDir = dir === workPath;
    const goMod = join(dir, 'go.mod');
    if (await pathExists(goMod)) {
      goModPath = goMod;
      debug(`Found ${goModPath}"`);
      break;
    }
    dir = dirname(dir);
  }

  return { goModPath, isGoModInRootDir };
}
```

#### 3.3.2 `getAnalyzedEntrypoint()` - 分析 Go 文件 AST

```typescript
// packages/go/src/go-helpers.ts

/**
 * 使用 Go AST 分析工具解析入口文件
 * 提取 packageName 和 functionName
 */
export async function getAnalyzedEntrypoint({
  entrypoint,
  modulePath,
  workPath,
}: {
  entrypoint: string;
  modulePath?: string;
  workPath: string;
}): Promise<Analyzed> {
  // analyze 二进制文件路径
  const bin = join(__dirname, `analyze${OUT_EXTENSION}`);

  // 如果 analyze 二进制不存在，先编译它
  const isAnalyzeExist = await pathExists(bin);
  if (!isAnalyzeExist) {
    debug(`Building analyze bin: ${bin}`);
    const src = join(__dirname, '../analyze.go');
    const go = await createGo({ modulePath, opts: { cwd: __dirname }, workPath });
    await go.build(src, bin);
  }

  // 执行 AST 分析
  const args = [`-modpath=${modulePath}`, join(workPath, entrypoint)];
  const analyzed = await execa.stdout(bin, args);

  if (!analyzed) {
    throw new Error(`Could not find an exported function in "${entrypoint}"`);
  }

  return JSON.parse(analyzed) as Analyzed;
  // 返回: { packageName: "handler", functionName: "Handler" }
}
```

#### 3.3.3 `renameHandlerFunction()` - 重命名 Handler 函数

```typescript
/**
 * 在源文件中重命名 Handler 函数
 * 目的：避免多个入口文件的函数名冲突
 */
async function renameHandlerFunction(fsPath: string, from: string, to: string) {
  let fileContents = await readFile(fsPath, 'utf8');

  // 使用正则替换函数名
  // 左侧：单个空格（匹配 `func Handler` 或 `var _ http.HandlerFunc = Index`）
  // 右侧：单词边界（匹配行尾或开括号）
  const fromRegex = new RegExp(String.raw` ${from}\b`, 'g');
  fileContents = fileContents.replace(fromRegex, ` ${to}`);

  await writeFile(fsPath, fileContents);
}

/**
 * 生成新的 Handler 函数名
 * 格式：{原函数名}_{入口路径slug}
 * 例如：Handler_api_hello_go
 */
export function getNewHandlerFunctionName(originalFunctionName: string, entrypoint: string) {
  const pathSlug = entrypoint.replace(/(\s|\\|\/|\]|\[|-|\.)/g, '_');
  return `${originalFunctionName}_${pathSlug}`;
}
```

#### 3.3.4 `buildHandlerWithGoMod()` - 使用 Go Modules 构建

```typescript
/**
 * 构建非 main 包的 Go 函数
 * 这是推荐的方式，使用 go.mod 管理依赖
 */
async function buildHandlerWithGoMod({
  downloadPath,
  entrypoint,
  entrypointAbsolute,
  entrypointDirname,
  go,
  goModPath,
  handlerFunctionName,
  isGoModInRootDir,
  outDir,
  packageName,
  undo,
}: BuildHandlerOptions): Promise<void> {
  debug(`Building Go handler as package "${packageName}"`);

  let goModDirname: string | undefined;

  // 1. 备份原始 go.mod
  if (goModPath !== undefined) {
    goModDirname = dirname(goModPath);
    const backupFile = join(goModDirname, `__vc_go.mod.bak`);
    await copy(goModPath, backupFile);
    undo.fileActions.push({ to: goModPath, from: backupFile });
  }

  // 2. 计算导入路径
  const entrypointArr = entrypoint.split(posix.sep);
  let goPackageName = `${packageName}/${packageName}`;
  const goFuncName = `${packageName}.${handlerFunctionName}`;

  // 如果有 go.mod，使用相对路径作为导入路径
  const relPackagePath = goModDirname
    ? posix.relative(goModDirname, entrypointDirname)
    : '';
  if (relPackagePath) {
    goPackageName = posix.join(packageName, relPackagePath);
  }

  // 3. 确定 main.go 写入位置
  let mainGoFile: string;
  if (goModPath && isGoModInRootDir) {
    mainGoFile = join(downloadPath, MAIN_GO_FILENAME);
  } else if (goModDirname && !isGoModInRootDir) {
    mainGoFile = join(goModDirname, MAIN_GO_FILENAME);
  } else {
    mainGoFile = join(entrypointDirname, MAIN_GO_FILENAME);
  }

  // 4. 写入 main.go 和 go.mod
  await Promise.all([
    writeEntrypoint(mainGoFile, goPackageName, goFuncName),
    writeGoMod({ destDir: goModDirname || entrypointDirname, goModPath, packageName }),
  ]);
  undo.fileActions.push({ to: undefined, from: mainGoFile });

  // 5. 移动用户文件到包目录
  let finalDestination = join(entrypointDirname, packageName, entrypointArr[entrypointArr.length - 1]);
  if (!goModPath || dirname(entrypointAbsolute) === dirname(goModPath)) {
    await move(entrypointAbsolute, finalDestination);
    undo.fileActions.push({ to: entrypointAbsolute, from: finalDestination });
    undo.directoryCreation.push(dirname(finalDestination));
  }

  // 6. go mod tidy
  debug('Tidy `go.mod` file...');
  await go.mod();

  // 7. go build
  debug('Running `go build`...');
  const destPath = join(outDir, HANDLER_FILENAME);  // 'bootstrap'
  const src = [join(baseGoModPath, MAIN_GO_FILENAME)];
  await go.build(src, destPath);
}
```

#### 3.3.5 `buildHandlerAsPackageMain()` - Legacy 模式构建

```typescript
/**
 * 构建 package main 的 Go 函数（旧版方式）
 * 需要 main.go 和用户文件在同一目录
 */
async function buildHandlerAsPackageMain({
  entrypointAbsolute,
  entrypointDirname,
  go,
  handlerFunctionName,
  outDir,
  undo,
}: BuildHandlerOptions): Promise<void> {
  debug('Building Go handler as package "main" (legacy)');

  // 1. 写入 main.go（使用空包名）
  await writeEntrypoint(
    join(entrypointDirname, MAIN_GO_FILENAME),
    '',  // 空包名表示 package main
    handlerFunctionName
  );
  undo.fileActions.push({ to: undefined, from: join(entrypointDirname, MAIN_GO_FILENAME) });

  // 2. go get（下载依赖）
  debug('Running `go get`...');
  await go.get();

  // 3. go build（编译多个文件）
  debug('Running `go build`...');
  const destPath = join(outDir, HANDLER_FILENAME);
  const src = [
    join(entrypointDirname, MAIN_GO_FILENAME),
    entrypointAbsolute,
  ].map(file => normalize(file));
  await go.build(src, destPath);
}
```

---

## 四、Dev Server 完整实现

### 4.1 `startDevServer()` 函数完整流程

```
startDevServer(opts)
    │
    ├─ 1. 修复入口文件扩展名
    │     └─ 确保以 .go 结尾
    │
    ├─ 2. 检查运行模式
    │     ├─ framework='go' → startStandaloneDevServer()
    │     └─ 默认 → 继续 Serverless Dev 流程
    │
    ├─ 3. 创建临时目录
    │     └─ .vercel/cache/go/{random}
    │
    ├─ 4. 查找并分析 go.mod
    │     └─ findGoModPath() + getAnalyzedEntrypoint()
    │
    ├─ 5. 复制和准备文件
    │     ├─ copyEntrypoint()      # 用户代码
    │     ├─ copyDevServer()       # dev-server.go 模板
    │     ├─ writeGoMod()          # 生成 go.mod
    │     └─ writeGoWork()         # 生成 go.work
    │
    ├─ 6. 编译 Dev Server
    │     └─ go.build('./...', executable)
    │
    ├─ 7. 启动进程
    │     └─ spawn(executable, { stdio: [..., 'pipe'] })
    │
    ├─ 8. 等待端口信息
    │     ├─ 方式1：从 FD 3 读取端口
    │     └─ 方式2：从端口文件读取
    │
    └─ 9. 返回 { port, pid }
```

### 4.2 `startDevServer()` 详细实现

```typescript
// packages/go/src/index.ts

export async function startDevServer(
  opts: StartDevServerOptions
): Promise<StartDevServerResult> {
  const { entrypoint, workPath, config, meta = {} } = opts;
  const { devCacheDir = join(workPath, '.vercel', 'cache') } = meta;

  // 1. 修复入口文件扩展名（路径段文件可能缺失 .go 后缀）
  let entrypointWithExt = entrypoint;
  if (!entrypoint.endsWith('.go')) {
    entrypointWithExt += '.go';
  }

  // 2. Standalone 模式分支
  if (config?.framework === 'go' || config?.framework === 'services') {
    const resolvedEntrypoint = await detectGoEntrypoint(workPath, entrypointWithExt);
    return startStandaloneDevServer(opts, resolvedEntrypoint);
  }

  // 3. 创建临时目录
  const entrypointDir = dirname(entrypointWithExt);
  const tmp = join(devCacheDir, 'go', Math.random().toString(32).substring(2));
  const tmpPackage = join(tmp, entrypointDir);
  await mkdirp(tmpPackage);

  // 4. 查找 go.mod 并分析入口文件
  const { goModPath } = await findGoModPath(join(workPath, entrypointDir), workPath);
  const modulePath = goModPath ? dirname(goModPath) : undefined;
  const analyzed = await getAnalyzedEntrypoint({
    entrypoint: entrypointWithExt,
    modulePath,
    workPath,
  });

  // 5. 并行复制/生成必要文件
  await Promise.all([
    copyEntrypoint(entrypointWithExt, tmpPackage),           // 用户代码（修改为 package main）
    copyDevServer(analyzed.functionName, tmpPackage),        // dev-server.go
    writeGoMod({ destDir: tmp, goModPath, packageName: analyzed.packageName }),
    writeGoWork(tmp, workPath, modulePath),
  ]);

  // 6. 准备端口文件路径（作为 FD 3 的备选方案）
  const portFile = join(TMP, `vercel-dev-port-${Math.random().toString(32).substring(2)}`);

  // 7. 设置环境变量
  const env = cloneEnv(process.env, meta.env, {
    VERCEL_DEV_PORT_FILE: portFile,
  });

  const executable = `./vercel-dev-server-go${process.platform === 'win32' ? '.exe' : ''}`;

  // 8. 编译 Dev Server（注意：必须先编译再运行，不能用 go run）
  const go = await createGo({
    modulePath,
    opts: { cwd: tmp, env },
    workPath,
  });
  await go.build('./...', executable);

  // 9. 启动进程（FD 3 用于接收端口号）
  debug(`SPAWNING ${executable} CWD=${tmp}`);
  const child = spawn(executable, [], {
    cwd: tmp,
    env,
    stdio: ['ignore', 'inherit', 'inherit', 'pipe'],  // FD 3 是管道
  });

  // 10. 进程退出时清理临时目录
  child.on('close', async () => {
    try {
      await retry(() => remove(tmp));
    } catch (err: any) {
      console.error(`Could not delete tmp directory: ${tmp}: ${err}`);
    }
  });

  // 11. 从 FD 3 读取端口号
  const portPipe = child.stdio[3];
  if (!isReadable(portPipe)) {
    throw new Error('File descriptor 3 is not readable');
  }

  const onPort = new Promise<PortInfo>(resolve => {
    portPipe.setEncoding('utf8');
    portPipe.once('data', d => {
      resolve({ port: Number(d) });
    });
  });
  const onPortFile = waitForPortFile(portFile);  // 备选：轮询端口文件
  const onExit = once.spread<[number, string | null]>(child, 'exit');

  // 12. 等待端口信息（三种情况竞争）
  const result = await Promise.race([onPort, onPortFile, onExit]);
  onExit.cancel();
  onPortFile.cancel();

  if (isPortInfo(result)) {
    return { port: result.port, pid: child.pid! };
  } else if (Array.isArray(result)) {
    const [exitCode, signal] = result;
    throw new Error(`\`go run ${entrypointWithExt}\` failed with ${signal || `exit code ${exitCode}`}`);
  } else {
    throw new Error(`Unexpected result type: ${typeof result}`);
  }
}
```

### 4.3 `dev-server.go` 模板

```go
// packages/go/dev-server.go

package main

import (
	"io/ioutil"
	"net"
	"net/http"
	"os"
	"strconv"
)

func main() {
	// 1. 创建 HTTP Handler（函数名由模板替换）
	handler := http.HandlerFunc(__HANDLER_FUNC_NAME)

	// 2. 监听随机端口
	listener, err := net.Listen("tcp", "127.0.0.1:0")
	if err != nil {
		panic(err)
	}

	// 3. 获取分配的端口号
	port := listener.Addr().(*net.TCPAddr).Port
	portBytes := []byte(strconv.Itoa(port))

	// 4. 尝试通过 FD 3 写入端口号
	file := os.NewFile(3, "pipe")
	_, err2 := file.Write(portBytes)
	if err2 != nil {
		// 5. 如果 FD 3 不可用，写入端口文件
		portFile := os.Getenv("VERCEL_DEV_PORT_FILE")
		os.Unsetenv("VERCEL_DEV_PORT_FILE")
		err3 := ioutil.WriteFile(portFile, portBytes, 0644)
		if err3 != nil {
			panic(err3)
		}
	}

	// 6. 启动 HTTP 服务器
	panic(http.Serve(listener, handler))
}
```

### 4.4 关键辅助函数

#### 4.4.1 `copyEntrypoint()` - 复制并修改用户代码

```typescript
/**
 * 复制用户代码到临时目录，并修改 package 为 main
 */
async function copyEntrypoint(entrypoint: string, dest: string): Promise<void> {
  const data = await readFile(entrypoint, 'utf8');

  // 将 package xxx 修改为 package main
  const patched = data.replace(/\bpackage\W+\S+\b/, 'package main');

  await writeFile(join(dest, 'entrypoint.go'), patched);
}
```

#### 4.4.2 `copyDevServer()` - 复制并填充 Dev Server 模板

```typescript
/**
 * 复制 dev-server.go 模板，填充 Handler 函数名
 */
async function copyDevServer(functionName: string, dest: string): Promise<void> {
  const data = await readFile(join(__dirname, '../dev-server.go'), 'utf8');

  // 替换占位符
  const patched = data.replace('__HANDLER_FUNC_NAME', functionName);

  await writeFile(join(dest, 'vercel-dev-server-main.go'), patched);
}
```

#### 4.4.3 `waitForPortFile()` - 轮询端口文件

```typescript
/**
 * 作为 FD 3 的备选方案，轮询端口文件
 */
async function waitForPortFile_(opts: { portFile: string; canceled: boolean }): Promise<PortInfo | void> {
  while (!opts.canceled) {
    await new Promise(resolve => setTimeout(resolve, 100));
    try {
      const port = Number(await readFile(opts.portFile, 'ascii'));
      retry(() => remove(opts.portFile));  // 清理端口文件
      return { port };
    } catch (err: any) {
      if (err.code !== 'ENOENT') {
        throw err;
      }
    }
  }
}
```

---

## 五、Standalone 模式实现

### 5.1 Build 流程

```typescript
// packages/go/src/standalone-server.ts

export async function buildStandaloneServer({
  files,
  entrypoint,
  config,
  workPath,
  meta = {},
}: BuildOptions): Promise<{ output: Lambda }> {
  debug(`Building standalone Go server: ${entrypoint}`);

  await download(files, workPath, meta);

  // 1. 获取 Lambda 配置
  const lambdaOptions = await getLambdaOptionsFromFunction({ sourceFile: entrypoint, config });
  const architecture = lambdaOptions?.architecture || 'x86_64';

  // 2. 设置交叉编译环境
  const env = cloneEnv(process.env, meta.env, {
    GOARCH: architecture === 'arm64' ? 'arm64' : 'amd64',
    GOOS: 'linux',
    CGO_ENABLED: '0',
  });

  // 3. 查找 go.mod
  const { goModPath } = await findGoModPath(workPath, workPath);
  const modulePath = goModPath ? dirname(goModPath) : workPath;

  const go = await createGo({ modulePath, opts: { cwd: workPath, env }, workPath });

  const outDir = await getWriteableDirectory();
  const userServerPath = join(outDir, 'user-server');
  const bootstrapPath = join(outDir, 'executable');

  // 4. 编译用户服务器
  const buildTarget = entrypoint === 'main.go' ? '.' : './' + dirname(entrypoint);
  await go.build(buildTarget, userServerPath);

  // 5. 编译 Bootstrap 包装器
  const bootstrapSrc = join(__dirname, '../vc-init.go');
  const bootstrapBuildDir = await getWriteableDirectory();
  await copy(bootstrapSrc, join(bootstrapBuildDir, 'main.go'));
  await writeFile(join(bootstrapBuildDir, 'go.mod'), 'module vc-init\n\ngo 1.21\n');

  const bootstrapGo = await createGo({
    modulePath: bootstrapBuildDir,
    opts: { cwd: bootstrapBuildDir, env },
    workPath: bootstrapBuildDir,
  });
  await bootstrapGo.build('.', bootstrapPath);

  // 6. 读取二进制文件
  const [userServerData, bootstrapData] = await Promise.all([
    readFile(userServerPath),
    readFile(bootstrapPath),
  ]);

  // 7. 创建 Lambda
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

### 5.2 `vc-init.go` Bootstrap 包装器

```go
// packages/go/vc-init.go

package main

import (
	"context"
	"encoding/json"
	"fmt"
	"net"
	"net/http"
	"net/http/httputil"
	"net/url"
	"os"
	"os/exec"
	"strconv"
	"strings"
	"sync"
	"time"
)

// IPC 消息结构
type StartMessage struct {
	Type    string       `json:"type"`
	Payload StartPayload `json:"payload"`
}

type StartPayload struct {
	InitDuration int `json:"initDuration"`
	HTTPPort     int `json:"httpPort"`
}

type EndMessage struct {
	Type    string     `json:"type"`
	Payload EndPayload `json:"payload"`
}

var (
	ipcConn   net.Conn
	ipcMutex  sync.Mutex
	startTime time.Time
)

func main() {
	startTime = time.Now()

	// 1. 连接 IPC Socket
	connectIPC()

	// 2. 查找空闲端口
	userPort, _ := findFreePort()

	// 3. 启动用户服务器
	cmd := exec.CommandContext(context.Background(), "./user-server")
	cmd.Env = append(os.Environ(), fmt.Sprintf("PORT=%d", userPort))
	cmd.Stdout = os.Stdout
	cmd.Stderr = os.Stderr
	cmd.Start()

	// 4. 等待用户服务器就绪
	waitForServer(userPort, 30*time.Second)

	// 5. 创建反向代理
	targetURL, _ := url.Parse(fmt.Sprintf("http://127.0.0.1:%d", userPort))
	proxy := httputil.NewSingleHostReverseProxy(targetURL)

	// 6. 创建 HTTP 服务器
	listenPort := 3000
	server := &http.Server{
		Addr: fmt.Sprintf("127.0.0.1:%d", listenPort),
		Handler: http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			// 处理健康检查
			if r.URL.Path == "/_vercel/ping" {
				w.WriteHeader(http.StatusOK)
				w.Write([]byte("OK"))
				return
			}

			// 提取 Vercel 内部头
			invocationID := r.Header.Get("X-Vercel-Internal-Invocation-Id")
			requestIDStr := r.Header.Get("X-Vercel-Internal-Request-Id")
			requestID, _ := strconv.ParseUint(requestIDStr, 10, 64)

			// 移除内部头再转发
			for key := range r.Header {
				if strings.HasPrefix(strings.ToLower(key), "x-vercel-internal-") {
					r.Header.Del(key)
				}
			}

			// 转发请求
			proxy.ServeHTTP(w, r)

			// 发送 end 消息
			if ipcConn != nil && invocationID != "" {
				sendIPCMessage(EndMessage{
					Type: "end",
					Payload: EndPayload{
						Context: RequestContext{
							InvocationID: invocationID,
							RequestID:    requestID,
						},
					},
				})
			}
		}),
	}

	// 7. 发送 server-started IPC 消息
	initDuration := int(time.Since(startTime).Milliseconds())
	sendIPCMessage(StartMessage{
		Type: "server-started",
		Payload: StartPayload{
			InitDuration: initDuration,
			HTTPPort:     listenPort,
		},
	})

	// 8. 启动服务器
	server.ListenAndServe()
}
```

### 5.3 Dev Server 流程

```typescript
// packages/go/src/standalone-server.ts

export async function startStandaloneDevServer(
  opts: StartDevServerOptions,
  resolvedEntrypoint: string
): Promise<StartDevServerResult> {
  const { workPath, meta = {} } = opts;

  // 随机端口
  const port = Math.floor(Math.random() * (65535 - 49152) + 49152);

  const env = cloneEnv(process.env, meta.env, {
    PORT: String(port),  // 通过环境变量传递端口
  });

  // 确定运行目标
  const runTarget = resolvedEntrypoint === 'main.go' ? '.' : './' + dirname(resolvedEntrypoint);

  debug(`Starting standalone Go dev server: go run ${runTarget} (port ${port})`);

  // 直接运行 go run
  const child = spawn('go', ['run', runTarget], {
    cwd: workPath,
    env,
    stdio: ['ignore', 'inherit', 'inherit'],
  });

  // 等待服务器启动
  await new Promise(resolve => setTimeout(resolve, 2000));

  return { port, pid: child.pid! };
}
```

---

## 六、Go 版本管理与编译器包装

### 6.1 `GoWrapper` 类

```typescript
// packages/go/src/go-helpers.ts

export class GoWrapper {
  private env: Env;
  private opts: execa.Options;

  constructor(env: Env, opts: execa.Options = {}) {
    if (!opts.cwd) {
      opts.cwd = process.cwd();
    }
    this.env = env;
    this.opts = opts;
  }

  /**
   * 执行 go 命令
   */
  private execute(...args: string[]) {
    const { opts, env } = this;
    debug(`Exec: go ${args.join(' ')}`);
    debug(`  CWD=${opts.cwd}`);
    debug(`  GOROOT=${env.GOROOT}`);
    return execa('go', args, { stdio: 'inherit', ...opts, env });
  }

  /**
   * go mod tidy
   */
  mod() {
    return this.execute('mod', 'tidy');
  }

  /**
   * go get [src]
   */
  get(src?: string) {
    const args = ['get'];
    if (src) {
      args.push(src);
    }
    return this.execute(...args);
  }

  /**
   * go build -ldflags "-s -w" -o dest src
   */
  build(src: string | string[], dest: string) {
    debug(`Building optimized 'go' binary ${src} -> ${dest}`);
    const sources = Array.isArray(src) ? src : [src];

    // 支持自定义构建标志
    const envGoBuildFlags = this.env.GO_BUILD_FLAGS;
    const flags = envGoBuildFlags ? stringArgv(envGoBuildFlags) : GO_FLAGS;
    // GO_FLAGS = ['-ldflags', '-s -w']  // 优化二进制大小

    return this.execute('build', ...flags, '-o', dest, ...sources);
  }
}
```

### 6.2 `createGo()` - 创建 GoWrapper 实例

```typescript
// packages/go/src/go-helpers.ts

/**
 * 版本映射表（短版本 -> 完整版本）
 */
const versionMap = new Map([
  ['1.24', '1.24.5'],
  ['1.23', '1.23.11'],
  ['1.22', '1.22.12'],
  // ...
]);

/**
 * 创建 GoWrapper 实例
 * 
 * 查找顺序：
 * 1. 本地缓存 (.vercel/cache/golang)
 * 2. 全局缓存 (~/.cache/com.vercel.cli/golang)
 * 3. 系统 PATH
 * 
 * 如果都找不到，自动下载所需版本
 */
export async function createGo({
  modulePath,
  opts = {},
  workPath,
}: CreateGoOptions): Promise<GoWrapper> {
  // 1. 从 go.mod 解析首选版本
  let goPreferredVersion: GoVersions | undefined;
  if (modulePath) {
    goPreferredVersion = await parseGoModVersionFromModule(modulePath);
  }

  // 2. 确定要使用的版本
  const goSelectedVersion = goPreferredVersion
    ? goPreferredVersion.toolchain || goPreferredVersion.go
    : Array.from(versionMap.values())[0];  // 默认最新版

  // 3. 准备环境变量
  const env = cloneEnv(process.env, opts.env);
  const { platform, arch } = process;
  const goGlobalCacheDir = join(goGlobalCachePath, `${goSelectedVersion}_${platform}_${arch}`);
  const goCacheDir = join(workPath, localCacheDir);

  if (goPreferredVersion) {
    env.GO111MODULE = 'on';
  }

  // 4. 查找可用的 Go 安装
  const goDirs = {
    'local cache': goCacheDir,
    'global cache': goGlobalCacheDir,
    'system PATH': null,
  };

  for (const [label, goDir] of Object.entries(goDirs)) {
    try {
      const goBinDir = goDir && join(goDir, 'bin');
      if (goBinDir && !(await pathExists(goBinDir))) {
        continue;
      }

      env.GOROOT = goDir || undefined;
      env.PATH = goBinDir || env.PATH;

      // 检查版本是否匹配
      const { stdout } = await execa('go', ['version'], { env });
      const { version, short } = parseGoVersionString(stdout);

      if (version === goSelectedVersion || short === goSelectedVersion) {
        debug(`Selected go ${version} (from ${label})`);
        await setGoEnv(goDir);
        return new GoWrapper(env, opts);
      }
    } catch {
      debug(`Go not found in ${label}`);
    }
  }

  // 5. 下载并安装所需版本
  await download({ dest: goGlobalCacheDir, version: goSelectedVersion });
  await setGoEnv(goGlobalCacheDir);

  return new GoWrapper(env, opts);
}
```

### 6.3 Go 版本下载

```typescript
/**
 * 下载并安装 Go
 */
async function download({ dest, version }: { dest: string; version: string }) {
  const { filename, url } = getGoUrl(version);
  console.log(`Downloading go: ${url}`);

  const res = await fetch(url);
  if (!res.ok) {
    throw new Error(`Failed to download: ${url} (${res.status})`);
  }

  await remove(dest);
  await mkdirp(dest);

  if (/\.zip$/.test(filename)) {
    // Windows: 解压 zip
    const zipFile = join(tmpdir(), filename);
    await streamPipeline(res.body, createWriteStream(zipFile));
    const zip = await yauzl.open(zipFile);
    // ... 解压逻辑
  } else {
    // Unix: 解压 tar.gz
    await new Promise((resolve, reject) => {
      res.body
        .pipe(extract({ cwd: dest, strip: 1 }))
        .on('error', reject)
        .on('finish', resolve);
    });
  }
}

/**
 * 获取 Go 下载 URL
 */
function getGoUrl(version: string) {
  const { arch, platform } = process;
  const ext = platform === 'win32' ? 'zip' : 'tar.gz';
  const goPlatform = platformMap.get(platform) || platform;  // win32 -> windows
  const goArch = archMap.get(arch) || arch;                  // x64 -> amd64

  const filename = `go${version}.${goPlatform}-${goArch}.${ext}`;
  return {
    filename,
    url: `https://dl.google.com/go/${filename}`,
  };
}
```

---

## 七、AST 分析工具 (`analyze.go`)

```go
// packages/go/analyze.go

package main

import (
	"encoding/json"
	"fmt"
	"go/ast"
	"go/parser"
	"go/token"
	"io/ioutil"
	"log"
	"os"
	"strings"
)

type analyze struct {
	PackageName string   `json:"packageName"`
	FuncName    string   `json:"functionName"`
	Watch       []string `json:"watch"`
}

func main() {
	if len(os.Args) != 3 {
		fmt.Println("Usage: ./go-analyze -modpath=module-path file_name.go")
		os.Exit(1)
	}

	fileName := os.Args[2]
	rf, _ := ioutil.ReadFile(fileName)
	
	// 解析 Go 源文件
	fset := token.NewFileSet()
	parsed, err := parser.ParseFile(fset, fileName, nil, parser.ParseComments)
	if err != nil {
		log.Fatal(err)
	}

	offset := parsed.Pos()

	// 遍历所有声明
	for _, decl := range parsed.Decls {
		fn, ok := decl.(*ast.FuncDecl)
		if !ok {
			continue
		}

		// 检查是否为导出函数
		if fn.Name.IsExported() {
			// 检查是否为有效的 http.HandlerFunc
			params := rf[fn.Type.Params.Pos()-offset : fn.Type.Params.End()-offset]
			validHandlerFunc := (
				strings.Contains(string(params), "http.ResponseWriter") &&
				strings.Contains(string(params), "*http.Request") &&
				len(fn.Type.Params.List) == 2 &&
				(fn.Recv == nil || len(fn.Recv.List) == 0)  // 非方法
			)

			if validHandlerFunc {
				analyzed := analyze{
					PackageName: parsed.Name.Name,
					FuncName:    fn.Name.Name,
				}
				analyzedJSON, _ := json.Marshal(analyzed)
				fmt.Print(string(analyzedJSON))
				os.Exit(0)
			}
		}
	}
}
```

---

## 八、缓存管理

### 8.1 `prepareCache()` 函数

```typescript
// packages/go/src/index.ts

export async function prepareCache({ workPath }: PrepareCacheOptions): Promise<Files> {
  // Go 缓存目录
  const goCacheDir = join(workPath, localCacheDir);  // .vercel/cache/golang

  // 检查是否为符号链接（指向全局缓存）
  const stat = await lstat(goCacheDir);
  if (stat.isSymbolicLink()) {
    // 如果是符号链接，将全局缓存移动到本地
    // 这样可以在下次构建时复用
    const goGlobalCacheDir = await readlink(goCacheDir);
    debug(`Preparing cache by moving ${goGlobalCacheDir} -> ${goCacheDir}`);
    await unlink(goCacheDir);
    await move(goGlobalCacheDir, goCacheDir);
  }

  // 返回缓存文件
  const cache = await glob(`${localCacheDir}/**`, workPath);
  return cache;
}
```

---

## 九、main.go 模板

### 9.1 Serverless 模式的 main.go

```go
// packages/go/main.go

package main

import (
	"net/http"
	"os"
	"syscall"

	"__VC_HANDLER_PACKAGE_NAME"                    // 用户包导入路径
	vc "github.com/vercel/go-bridge/go/bridge"    // Vercel Go Bridge
)

func checkForLambdaWrapper() {
	wrapper := os.Getenv("AWS_LAMBDA_EXEC_WRAPPER")
	if wrapper == "" {
		return
	}

	os.Setenv("AWS_LAMBDA_EXEC_WRAPPER", "")
	argv := append([]string{wrapper}, os.Args...)
	err := syscall.Exec(wrapper, argv, os.Environ())
	if err != nil {
		panic(err)
	}
}

func main() {
	checkForLambdaWrapper()
	// __VC_HANDLER_FUNC_NAME 会被替换为实际的函数名
	vc.Start(http.HandlerFunc(__VC_HANDLER_FUNC_NAME))
}
```

### 9.2 `writeEntrypoint()` 函数

```typescript
/**
 * 生成 main.go 文件
 */
async function writeEntrypoint(
  dest: string,
  goPackageName: string,
  goFuncName: string
) {
  const modMainGoContents = await readFile(join(__dirname, '../main.go'), 'utf8');

  const mainModGoContents = modMainGoContents
    .replace('__VC_HANDLER_PACKAGE_NAME', goPackageName)
    .replace('__VC_HANDLER_FUNC_NAME', goFuncName);

  await writeFile(dest, mainModGoContents, 'utf-8');
}
```

---

## 十、完整实现清单

### 10.1 必须实现的函数

| 函数 | 描述 | 文件 |
|------|------|------|
| `build()` | 构建 Lambda | index.ts |
| `startDevServer()` | 启动开发服务器 | index.ts |
| `prepareCache()` | 准备缓存 | index.ts |
| `createGo()` | 创建 Go 编译器包装 | go-helpers.ts |
| `getAnalyzedEntrypoint()` | AST 分析 | go-helpers.ts |
| `findGoModPath()` | 查找 go.mod | index.ts |
| `buildStandaloneServer()` | Standalone 构建 | standalone-server.ts |
| `startStandaloneDevServer()` | Standalone Dev | standalone-server.ts |
| `detectGoEntrypoint()` | 检测入口点 | entrypoint.ts |

### 10.2 必须实现的 Go 模板

| 文件 | 用途 |
|------|------|
| `dev-server.go` | Dev 模式的 HTTP 服务器模板 |
| `main.go` | Build 模式的 main 函数模板 |
| `vc-init.go` | Standalone 模式的 Bootstrap |
| `analyze.go` | AST 分析工具 |

### 10.3 关键依赖

```json
{
  "devDependencies": {
    "@vercel/build-utils": "workspace:*",  // Lambda、Files、glob 等
    "execa": "^1.0.0",                      // 执行命令
    "fs-extra": "^7.0.0",                   // 文件操作
    "async-retry": "1.3.3",                 // 重试逻辑
    "tar": "7.5.7",                         // 解压 tar.gz
    "yauzl-promise": "2.1.3",               // 解压 zip
    "xdg-app-paths": "5.1.0",               // 获取缓存目录
    "string-argv": "0.3.1"                  // 解析命令行参数
  }
}
```

---

## 十一、IPC 通信机制

### 11.1 Dev 模式 IPC

```
┌─────────────────┐                    ┌─────────────────┐
│   DevServer     │                    │  Go Dev Server  │
│   (Node.js)     │                    │   (go binary)   │
└────────┬────────┘                    └────────┬────────┘
         │                                      │
         │  spawn(executable, { stdio: [       │
         │    'ignore',                        │
         │    'inherit',                       │
         │    'inherit',                       │
         │    'pipe'  ← FD 3                   │
         │  ]})                                │
         │                                      │
         │←─────── 端口号 (via FD 3) ───────────│
         │                                      │
         │  proxyPass(req) ──────────────────→ │
         │←────────── response ────────────────│
```

### 11.2 Build 模式 IPC (Standalone)

```
┌─────────────────┐      Unix Socket       ┌─────────────────┐
│  Vercel Runtime │◄────────────────────►  │    Bootstrap    │
│   (Lambda)      │   VERCEL_IPC_PATH      │   (vc-init)     │
└─────────────────┘                        └────────┬────────┘
                                                    │
                              spawn("./user-server")│
                              env: PORT=xxxx        │
                                                    ▼
                                           ┌─────────────────┐
                                           │  User Server    │
                                           │   (user-server) │
                                           └─────────────────┘
```

---

## 十二、关键点总结

### 12.1 Dev 模式关键点

1. **必须先编译再运行**：不能使用 `go run`，必须 `go build` 后再 `spawn`
2. **FD 3 通信**：Go 进程通过 FD 3 写入端口号，Node.js 读取
3. **端口文件备选**：如果 FD 3 不可用，使用 `VERCEL_DEV_PORT_FILE`
4. **临时目录隔离**：每次请求在独立临时目录构建
5. **package main 转换**：用户代码需要修改为 `package main`

### 12.2 Build 模式关键点

1. **交叉编译**：`GOOS=linux GOARCH=amd64`
2. **Handler 函数重命名**：避免多入口冲突
3. **go.mod 处理**：备份、修改、恢复
4. **Lambda Runtime**：Serverless 用 `provided.al2023`，Standalone 用 `executable`
5. **二进制命名**：Serverless 输出 `bootstrap`，Standalone 输出 `executable` + `user-server`

### 12.3 版本管理关键点

1. **从 go.mod 读取版本**：优先使用用户指定版本
2. **三级查找**：本地缓存 → 全局缓存 → 系统 PATH
3. **自动下载**：找不到时从 `dl.google.com/go/` 下载
4. **缓存持久化**：`prepareCache()` 保存 Go 安装到 `.vercel/cache/golang`

### 12.4 AST 分析关键点

1. **查找导出函数**：`IsExported() == true`
2. **验证 HandlerFunc 签名**：`(http.ResponseWriter, *http.Request)`
3. **非方法函数**：`fn.Recv == nil || len(fn.Recv.List) == 0`
4. **返回 JSON**：`{ packageName, functionName }`

---

## 十三、实现步骤建议

1. **先实现 GoWrapper**：封装 go 命令执行
2. **实现 AST 分析**：编写 analyze.go 并集成
3. **实现 Dev 模式**：从 `startDevServer()` 开始，验证 IPC 通信
4. **实现 Build 模式**：实现 `build()`，验证 Lambda 输出
5. **实现 Standalone 模式**：添加 framework='go' 支持
6. **实现版本管理**：添加 Go 版本检测和下载
7. **实现缓存**：添加 `prepareCache()`

---

*文档完成，共约 2000 行，涵盖 Vercel Go 支持的完整实现细节。*
