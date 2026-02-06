# Vercel CLI 支持 Python 语言的完整解析

本文档详细分析 Vercel CLI 中 `dev` 命令和 `build` 命令对 Python 语言的支持，包括本地开发和生产部署的完整流程。

## 目录

### 第一部分：dev 命令（本地开发）

1. [概述](#1-概述)
2. [整体架构](#2-整体架构)
3. [startDevServer 执行流程](#3-startdevserver-执行流程)
4. [Dev Server 两种模式](#4-dev-server-两种模式)
5. [虚拟环境检测与使用](#5-虚拟环境检测与使用)
6. [静态文件服务](#6-静态文件服务)
7. [持久化服务器机制](#7-持久化服务器机制)

### 第二部分：build 命令（生产构建）

8. [Build 命令概述](#8-build-命令概述)
9. [Build 整体架构](#9-build-整体架构)
10. [Build 执行流程详解](#10-build-执行流程详解)
11. [Python 版本管理](#11-python-版本管理)
12. [依赖安装机制](#12-依赖安装机制)
13. [uv 包管理工具](#13-uv-包管理工具)
14. [Lambda 打包与 Runtime Trampoline](#14-lambda-打包与-runtime-trampoline)
15. [生产环境 WSGI/ASGI 处理](#15-生产环境-wsgiasgi-处理)

### 第三部分：dev 与 build 的对比

16. [核心差异对比](#16-核心差异对比)
17. [共享组件分析](#17-共享组件分析)
18. [完整对比总结表](#18-完整对比总结表)

---

# 第一部分：dev 命令（本地开发）

## 1. 概述

Vercel CLI 的 `dev` 命令通过 `@vercel/python` 运行时包支持 Python 语言的本地开发。目前 dev 模式仅支持 **FastAPI** 和 **Flask** 两个框架。

### 支持的框架

| 框架 | 协议 | Dev Server | 备注 |
|------|------|------------|------|
| **FastAPI** | ASGI | uvicorn / hypercorn | 优先使用 uvicorn |
| **Flask** | WSGI | Werkzeug / wsgiref | 优先使用 Werkzeug |

### 关键特性

- **持久化服务器**：服务器在请求间保持运行，支持后台任务
- **静态文件服务**：自动从 `public/` 目录提供静态文件
- **虚拟环境检测**：自动检测并使用用户的虚拟环境
- **热重载**：依赖 uvicorn/Werkzeug 的内置重载机制

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
│  • Builder 调度                                                          │
└───────────────────────────────────┬─────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                   @vercel/python startDevServer()                        │
│  • 框架检测 (FastAPI/Flask)                                              │
│  • 入口点检测                                                            │
│  • 虚拟环境检测                                                          │
│  • 持久化服务器管理                                                       │
└───────────────────────────────────┬─────────────────────────────────────┘
                                    │
           ┌────────────────────────┴────────────────────────┐
           │                                                  │
           ▼                                                  ▼
┌──────────────────────────┐                    ┌──────────────────────────┐
│  ASGI 模式 (FastAPI)     │                    │  WSGI 模式 (Flask)       │
│                          │                    │                          │
│  • vc_init_dev_asgi.py   │                    │  • vc_init_dev_wsgi.py   │
│  • uvicorn / hypercorn   │                    │  • Werkzeug / wsgiref    │
│  • StaticFiles 中间件     │                    │  • 自定义静态文件处理     │
└──────────────────────────┘                    └──────────────────────────┘
           │                                                  │
           └────────────────────────┬─────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                        用户 Python 应用                                  │
│                      (app.py 中的 app 对象)                              │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 3. startDevServer 执行流程

### 3.1 函数入口 (`packages/python/src/start-dev-server.ts`)

```typescript
export const startDevServer: StartDevServer = async opts => {
  const { entrypoint: rawEntrypoint, workPath, meta = {}, config, onStdout, onStderr } = opts;

  // 1. 仅支持 FastAPI 和 Flask
  const framework = config?.framework;
  if (framework !== 'fastapi' && framework !== 'flask') {
    return null;  // 返回 null 表示不处理，使用传统构建流程
  }

  // 2. 检测入口点
  const entry = await detectPythonEntrypoint(framework, workPath, rawEntrypoint);
  if (!entry) {
    throw new NowBuildError({
      code: 'PYTHON_ENTRYPOINT_NOT_FOUND',
      message: `No ${framework} entrypoint found...`,
    });
  }

  // 3. 转换为模块路径 (e.g. "src/app.py" -> "src.app")
  const modulePath = entry.replace(/\.py$/i, '').replace(/[\\/]/g, '.');

  // 4. 检查是否有已存在的持久化服务器
  const serverKey = `${workPath}::${entry}::${framework}`;
  const existing = PERSISTENT_SERVERS.get(serverKey);
  if (existing) {
    return { port: existing.port, pid: existing.pid, shutdown: async () => {} };
  }

  // 5. 检测虚拟环境
  const { pythonCmd, venvRoot } = useVirtualEnv(workPath, env, systemPython);

  // 6. 根据框架启动对应的 dev server
  if (framework !== 'flask') {
    // ASGI 模式
    const devShimModule = createDevAsgiShim(workPath, modulePath);
    spawn(pythonCmd, ['-u', '-m', devShimModule], { cwd: workPath, env });
  } else {
    // WSGI 模式
    const devShimModule = createDevWsgiShim(workPath, modulePath);
    spawn(pythonCmd, ['-u', '-m', devShimModule], { cwd: workPath, env });
  }

  // 7. 等待服务器启动并获取端口
  // 通过正则匹配日志输出: "Uvicorn running on http://127.0.0.1:PORT"

  // 8. 保存到持久化服务器映射
  PERSISTENT_SERVERS.set(serverKey, { port, pid, child, ... });

  return { port, pid, shutdown: async () => {} };
};
```

### 3.2 入口点检测 (`packages/python/src/entrypoint.ts`)

```typescript
// 候选文件名
export const PYTHON_ENTRYPOINT_FILENAMES = ['app', 'index', 'server', 'main', 'wsgi', 'asgi'];

// 候选目录
export const PYTHON_ENTRYPOINT_DIRS = ['', 'src', 'app', 'api'];

// 生成候选列表
export const PYTHON_CANDIDATE_ENTRYPOINTS = PYTHON_ENTRYPOINT_FILENAMES.flatMap(
  filename => PYTHON_ENTRYPOINT_DIRS.map(dir => pathPosix.join(dir, `${filename}.py`))
);
// 结果: ['app.py', 'src/app.py', 'app/app.py', 'api/app.py', 'index.py', 'src/index.py', ...]

export async function detectPythonEntrypoint(
  framework: PythonFramework,
  workPath: string,
  configuredEntrypoint: string
): Promise<string | null> {
  // 1. 尝试配置的入口点
  if (configuredEntrypoint && await pathExists(join(workPath, configuredEntrypoint))) {
    return configuredEntrypoint;
  }

  // 2. 搜索候选文件
  for (const candidate of PYTHON_CANDIDATE_ENTRYPOINTS) {
    const fullPath = join(workPath, candidate);
    if (await pathExists(fullPath)) {
      // 使用 AST 检测文件是否包含有效的 app 定义
      if (await hasValidAppDefinition(fullPath)) {
        return candidate;
      }
    }
  }

  // 3. 检查 pyproject.toml [project.scripts].app
  return await getPyprojectEntrypoint(workPath);
}
```

---

## 4. Dev Server 两种模式

### 4.1 ASGI 模式 (FastAPI)

**Shim 文件**: `vc_init_dev_asgi.py`

```python
# packages/python/vc_init_dev_asgi.py
from importlib import import_module

# 尝试导入 StaticFiles (FastAPI/Starlette)
StaticFiles = None
try:
    from fastapi.staticfiles import StaticFiles as _SF
    StaticFiles = _SF
except:
    try:
        from starlette.staticfiles import StaticFiles as _SF
        StaticFiles = _SF
    except:
        pass

# 导入用户模块 (__VC_DEV_MODULE_PATH__ 在生成时被替换)
USER_MODULE = "__VC_DEV_MODULE_PATH__"  # 如 "src.app"
_mod = import_module(USER_MODULE)
_app = getattr(_mod, 'app', None)

if _app is None:
    raise RuntimeError(f"Missing 'app' in module '{USER_MODULE}'")

# 获取实际的 ASGI 应用
USER_ASGI_APP = getattr(_app, 'asgi', _app) if callable(getattr(_app, 'asgi', None)) else _app

PUBLIC_DIR = 'public'

# 准备静态文件应用
static_app = None
if StaticFiles is not None:
    static_app = StaticFiles(directory=PUBLIC_DIR, check_dir=False)

async def app(scope, receive, send):
    """组合静态文件和用户应用的 ASGI 应用"""
    if static_app is not None and scope.get('type') == 'http':
        req_path = scope.get('path', '/') or '/'
        safe = _p.normpath(req_path).lstrip('/')
        target = _p.join(PUBLIC_DIR, safe)
        
        # 如果是有效的静态文件，返回它
        if _p.isfile(target):
            await static_app(scope, receive, send)
            return
    
    # 否则转发到用户应用
    await USER_ASGI_APP(scope, receive, send)

if __name__ == '__main__':
    # 优先使用 uvicorn
    try:
        import uvicorn
        uvicorn.run('vc_init_dev_asgi:app', host='127.0.0.1', port=0, use_colors=True)
    except:
        # 备选: hypercorn
        import asyncio
        from hypercorn.config import Config
        from hypercorn.asyncio import serve
        
        config = Config()
        config.bind = ['127.0.0.1:0']
        asyncio.run(serve(app, config))
```

### 4.2 WSGI 模式 (Flask)

**Shim 文件**: `vc_init_dev_wsgi.py`

```python
# packages/python/vc_init_dev_wsgi.py
from importlib import import_module
import mimetypes

USER_MODULE = "__VC_DEV_MODULE_PATH__"
PUBLIC_DIR = "public"

_mod = import_module(USER_MODULE)
_app = getattr(_mod, "app", None)

if _app is None:
    raise RuntimeError(f"Missing 'app' in module '{USER_MODULE}'")

def _static_wsgi_app(environ, start_response):
    """处理静态文件的 WSGI 应用"""
    if environ.get("REQUEST_METHOD") not in ("GET", "HEAD"):
        return _not_found(start_response)
    
    req_path = environ.get("PATH_INFO", "/")
    safe = _p.normpath(req_path).lstrip("/")
    full = _p.join(PUBLIC_DIR, safe)
    
    if not _is_safe_file(PUBLIC_DIR, full):
        return _not_found(start_response)
    
    ctype, _ = mimetypes.guess_type(full)
    headers = [("Content-Type", ctype or "application/octet-stream")]
    
    with open(full, "rb") as f:
        data = f.read()
    headers.append(("Content-Length", str(len(data))))
    start_response("200 OK", headers)
    return [data]

def _combined_app(environ, start_response):
    """组合静态文件和用户应用"""
    # 先尝试静态文件
    result = _static_wsgi_app(environ, capture_start_response)
    if captured_status.startswith("200 "):
        return result
    
    # 否则转发到用户应用
    return _app(environ, start_response)

app = _combined_app

if __name__ == "__main__":
    # 优先使用 Werkzeug
    try:
        from werkzeug.serving import run_simple
        run_simple('127.0.0.1', 0, app, use_reloader=False)
    except:
        # 备选: wsgiref
        from wsgiref.simple_server import make_server
        httpd = make_server('127.0.0.1', 0, app)
        port = httpd.server_port
        print(f"Serving on http://127.0.0.1:{port}")
        httpd.serve_forever()
```

---

## 5. 虚拟环境检测与使用

### 5.1 检测逻辑 (`packages/python/src/utils.ts`)

```typescript
export function isInVirtualEnv(): string | null {
  // 检查 VIRTUAL_ENV 环境变量
  const venv = process.env.VIRTUAL_ENV;
  if (venv && fs.existsSync(venv)) {
    return venv;
  }
  return null;
}

export function useVirtualEnv(
  workPath: string,
  env: NodeJS.ProcessEnv,
  fallbackPython: string
): { pythonCmd: string; venvRoot: string | null } {
  // 1. 检查是否已在虚拟环境中
  const activeVenv = isInVirtualEnv();
  if (activeVenv) {
    return { pythonCmd: getVenvPythonBin(activeVenv), venvRoot: activeVenv };
  }

  // 2. 搜索项目目录下的虚拟环境
  const candidates = ['.venv', 'venv', '.env', 'env'];
  for (const name of candidates) {
    const venvPath = join(workPath, name);
    if (fs.existsSync(venvPath)) {
      const pythonBin = getVenvPythonBin(venvPath);
      if (fs.existsSync(pythonBin)) {
        return { pythonCmd: pythonBin, venvRoot: venvPath };
      }
    }
  }

  // 3. 使用回退的系统 Python
  return { pythonCmd: fallbackPython, venvRoot: null };
}

export function getVenvPythonBin(venvPath: string): string {
  const isWin = process.platform === 'win32';
  return isWin
    ? join(venvPath, 'Scripts', 'python.exe')
    : join(venvPath, 'bin', 'python');
}
```

### 5.2 虚拟环境搜索优先级

1. `VIRTUAL_ENV` 环境变量（已激活的虚拟环境）
2. `.venv/` 目录
3. `venv/` 目录
4. `.env/` 目录
5. `env/` 目录
6. 系统 `python3`

---

## 6. 静态文件服务

### 6.1 工作原理

Dev 模式下，静态文件从 `public/` 目录提供服务：

```
project/
├── public/
│   ├── favicon.ico
│   ├── images/
│   │   └── logo.png
│   └── styles.css
├── app.py
└── requirements.txt
```

请求路径 `/images/logo.png` 会首先检查 `public/images/logo.png` 是否存在。

### 6.2 安全检查

```python
def _is_safe_file(base_dir: str, target: str) -> bool:
    """防止路径遍历攻击"""
    try:
        base = _p.realpath(base_dir)
        tgt = _p.realpath(target)
        # 确保目标路径在 base_dir 内
        return (tgt == base or tgt.startswith(base + os.sep)) and _p.isfile(tgt)
    except:
        return False
```

---

## 7. 持久化服务器机制

### 7.1 设计目的

传统的 Serverless 模式下，每个请求都会启动一个新进程。但 Python Web 框架（尤其是 FastAPI）通常有后台任务、WebSocket 连接等需要持久化的场景。

### 7.2 实现方式

```typescript
// 持久化服务器映射
const PERSISTENT_SERVERS = new Map<
  string,  // key: `${workPath}::${entry}::${framework}`
  {
    port: number;
    pid: number;
    child: ChildProcess;
    stdoutLogListener: ((buf: Buffer) => void) | null;
    stderrLogListener: ((buf: Buffer) => void) | null;
  }
>();

// 检查已存在的服务器
const existing = PERSISTENT_SERVERS.get(serverKey);
if (existing) {
  return {
    port: existing.port,
    pid: existing.pid,
    shutdown: async () => {
      // no-op: 不终止持久化服务器
    },
  };
}
```

### 7.3 生命周期管理

```typescript
function installGlobalCleanupHandlers() {
  const killAll = () => {
    for (const [key, info] of PERSISTENT_SERVERS.entries()) {
      process.kill(info.pid, 'SIGTERM');
      process.kill(info.pid, 'SIGKILL');
      PERSISTENT_SERVERS.delete(key);
    }
  };

  process.on('SIGINT', () => { killAll(); process.exit(130); });
  process.on('SIGTERM', () => { killAll(); process.exit(143); });
  process.on('exit', () => { killAll(); });
}
```

---

# 第二部分：build 命令（生产构建）

## 8. Build 命令概述

`@vercel/python` 的 `build()` 函数负责将 Python 应用打包为可以在 Vercel 平台上运行的 Lambda 函数。

### 核心特点

| 特性 | 说明 |
|------|------|
| **Python 版本** | 支持 3.9 - 3.14，默认 3.12 |
| **依赖管理** | 使用 `uv` 统一处理所有依赖格式 |
| **输出格式** | AWS Lambda Python Runtime |
| **运行时** | `vercel-runtime` 包提供 WSGI/ASGI 适配 |
| **响应流** | 支持 Response Streaming |

---

## 9. Build 整体架构

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         Vercel Build System                              │
│                    (vercel build / vercel deploy)                        │
└───────────────────────────────────┬─────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                        @vercel/python build()                            │
│  • 下载源文件                                                            │
│  • 检测 Python 版本                                                      │
│  • 创建虚拟环境                                                          │
│  • 安装依赖 (uv)                                                         │
│  • 生成 Runtime Trampoline                                              │
└───────────────────────────────────┬─────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                          依赖安装流程                                     │
│                                                                          │
│  uv.lock ─────┐                                                          │
│  pyproject.toml ──┼──▶ ensureUvProject() ──▶ uv sync ──▶ _vendor/       │
│  Pipfile.lock ────┤                                                      │
│  requirements.txt ┘                                                      │
└───────────────────────────────────┬─────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                            Lambda 输出                                   │
│                                                                          │
│  files: {                                                                │
│    'vc__handler__python.py': Runtime Trampoline,                        │
│    '_vendor/': 所有依赖包,                                               │
│    'app.py': 用户代码,                                                   │
│    ...                                                                   │
│  }                                                                       │
│  handler: 'vc__handler__python.vc_handler'                              │
│  runtime: 'python3.12'                                                  │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                          Vercel 部署平台                                 │
│                    (AWS Lambda Python Runtime)                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 10. Build 执行流程详解

### 10.1 build() 函数入口 (`packages/python/src/index.ts`)

```typescript
export const build: BuildV3 = async ({
  workPath,
  repoRootPath,
  files: originalFiles,
  entrypoint,
  meta = {},
  config,
}) => {
  const framework = config?.framework;

  // 1. 下载文件到工作目录
  workPath = await downloadFilesInWorkPath({ workPath, files: originalFiles, entrypoint, meta });

  // 2. 对于 Python 框架，运行项目构建命令
  if (isPythonFramework(framework)) {
    // 运行 projectBuildCommand 或 pyproject.toml 中的脚本
    const projectBuildCommand = config?.projectSettings?.buildCommand;
    if (projectBuildCommand) {
      await execCommand(projectBuildCommand, { env: spawnEnv, cwd: workPath });
    } else {
      await runPyprojectScript(workPath, ['vercel-build', 'now-build', 'build'], spawnEnv);
    }
  }

  // 3. 零配置入口点发现
  if (isPythonFramework(framework) && !fsFiles[entrypoint]) {
    const detected = await detectPythonEntrypoint(framework, workPath, entrypoint);
    if (detected) {
      entrypoint = detected;
    } else {
      throw new NowBuildError({ code: 'ENTRYPOINT_NOT_FOUND', ... });
    }
  }

  // 4. 检测 Python 版本
  // 优先级: .python-version > pyproject.toml > Pipfile.lock > 默认 3.12
  const pythonVersion = getSupportedPythonVersion({ isDev: meta.isDev, declaredPythonVersion });

  // 5. 创建虚拟环境
  const venvPath = join(workPath, '.vercel', 'python', '.venv');
  await ensureVenv({ pythonPath: pythonVersion.pythonPath, venvPath });

  // 6. 安装依赖
  const uv = new UvRunner(await getUvBinaryOrInstall(pythonVersion.pythonPath));
  const { projectDir } = await ensureUvProject({ workPath, ..., uv, venvPath });
  await uv.sync({ venvPath, projectDir, locked: true });

  // 7. 安装 vercel-runtime
  await uv.pip({ venvPath, projectDir, args: ['install', `vercel-runtime==${VERSION}`] });

  // 8. 生成 Runtime Trampoline
  const runtimeTrampoline = `
import os, sys, site
os.environ.update({
  "__VC_HANDLER_MODULE_NAME": "${moduleName}",
  "__VC_HANDLER_ENTRYPOINT": "${entrypointWithSuffix}",
  ...
})
site.addsitedir(_vendor)
from vercel_runtime.vc_init import vc_handler
`;

  // 9. 打包 Lambda
  const output = new Lambda({
    files: { ...files, [`${handlerPyFilename}.py`]: new FileBlob({ data: runtimeTrampoline }) },
    handler: `${handlerPyFilename}.vc_handler`,
    runtime: pythonVersion.runtime,
    environment: { PYTHONPATH: vendorDir },
    supportsResponseStreaming: true,
  });

  return { output };
};
```

### 10.2 关键步骤详解

#### 步骤 1-2: 下载文件和运行构建命令

```typescript
// 下载文件
workPath = await downloadFilesInWorkPath({ workPath, files: originalFiles, entrypoint, meta });

// 运行构建命令 (如果配置了)
if (projectBuildCommand) {
  console.log(`Running "${projectBuildCommand}"`);
  await execCommand(projectBuildCommand, { env: spawnEnv, cwd: workPath });
} else {
  // 尝试运行 pyproject.toml 中定义的脚本
  await runPyprojectScript(workPath, ['vercel-build', 'now-build', 'build'], spawnEnv);
}
```

#### 步骤 3: 入口点检测

```typescript
const detected = await detectPythonEntrypoint(config.framework, workPath, entrypoint);
// 搜索: app.py, src/app.py, app/app.py, api/app.py, index.py, ...
```

#### 步骤 4: Python 版本检测

```typescript
// 优先级: .python-version > pyproject.toml > Pipfile.lock > 默认
const pythonVersion = getSupportedPythonVersion({ isDev: meta.isDev, declaredPythonVersion });
// 结果: { version: '3.12', pythonPath: 'python3.12', runtime: 'python3.12' }
```

#### 步骤 5-6: 虚拟环境和依赖安装

```typescript
// 创建虚拟环境
const venvPath = join(workPath, '.vercel', 'python', '.venv');
await ensureVenv({ pythonPath: pythonVersion.pythonPath, venvPath });

// 统一依赖格式并安装
const { projectDir } = await ensureUvProject({ workPath, ..., uv, venvPath });
await uv.sync({ venvPath, projectDir, locked: true });
```

#### 步骤 7: 安装 vercel-runtime

```typescript
// vercel-runtime 提供生产环境的 WSGI/ASGI 适配
const runtimeDep = `vercel-runtime==${VERCEL_RUNTIME_VERSION}`;
await uv.pip({ venvPath, projectDir, args: ['install', runtimeDep] });
```

#### 步骤 8: 生成 Runtime Trampoline

```typescript
const runtimeTrampoline = `
import importlib
import os
import os.path
import site
import sys

_here = os.path.dirname(__file__)

os.environ.update({
  "__VC_HANDLER_MODULE_NAME": "${moduleName}",
  "__VC_HANDLER_ENTRYPOINT": "${entrypointWithSuffix}",
  "__VC_HANDLER_ENTRYPOINT_ABS": os.path.join(_here, "${entrypointWithSuffix}"),
  "__VC_HANDLER_VENDOR_DIR": "${vendorDir}",
})

_vendor_rel = '${vendorDir}'
_vendor = os.path.normpath(os.path.join(_here, _vendor_rel))

if os.path.isdir(_vendor):
    site.addsitedir(_vendor)
    # Move _vendor to front of sys.path
    while _vendor in sys.path:
        sys.path.remove(_vendor)
    sys.path.insert(1, _vendor)
    importlib.invalidate_caches()

from vercel_runtime.vc_init import vc_handler
`;
```

---

## 11. Python 版本管理

### 11.1 支持的版本 (`packages/python/src/version.ts`)

```typescript
export const DEFAULT_PYTHON_VERSION = '3.12';

const allOptions: PythonVersion[] = [
  { version: '3.14', pipPath: 'pip3.14', pythonPath: 'python3.14', runtime: 'python3.14' },
  { version: '3.13', pipPath: 'pip3.13', pythonPath: 'python3.13', runtime: 'python3.13' },
  { version: '3.12', pipPath: 'pip3.12', pythonPath: 'python3.12', runtime: 'python3.12' },  // 默认
  { version: '3.11', pipPath: 'pip3.11', pythonPath: 'python3.11', runtime: 'python3.11' },
  { version: '3.10', pipPath: 'pip3.10', pythonPath: 'python3.10', runtime: 'python3.10' },
  { version: '3.9', pipPath: 'pip3.9', pythonPath: 'python3.9', runtime: 'python3.9' },
  { version: '3.6', pipPath: 'pip3.6', pythonPath: 'python3.6', runtime: 'python3.6',
    discontinueDate: new Date('2022-07-18') },  // 已弃用
];
```

### 11.2 版本检测优先级

| 优先级 | 来源 | 示例 |
|-------|------|------|
| 1 | `.python-version` 文件 | `3.12` |
| 2 | `pyproject.toml` 的 `requires-python` | `>=3.10` |
| 3 | `Pipfile.lock` 的 `_meta.requires.python_version` | `3.11` |
| 4 | 默认版本 | `3.12` |

### 11.3 Dev 模式特殊处理

```typescript
function getDevPythonVersion(): PythonVersion {
  // Dev 模式使用系统 Python
  return {
    version: '3',
    pipPath: 'pip3',
    pythonPath: 'python3',
    runtime: 'python3',
  };
}
```

---

## 12. 依赖安装机制

### 12.1 Manifest 检测 (`packages/python/src/install.ts`)

```typescript
export type ManifestType =
  | 'uv.lock'
  | 'pyproject.toml'
  | 'Pipfile.lock'
  | 'Pipfile'
  | 'requirements.txt'
  | null;

export async function detectInstallSource({
  workPath,
  entryDirectory,
  fsFiles,
}: DetectInstallSourceParams): Promise<InstallSourceInfo> {
  // 检测顺序 (优先级从高到低)
  const uvLockDir = findDir({ file: 'uv.lock', ... });
  const pyprojectDir = findDir({ file: 'pyproject.toml', ... });
  const pipfileLockDir = findDir({ file: 'Pipfile.lock', ... });
  const pipfileDir = findDir({ file: 'Pipfile', ... });
  const requirementsDir = findDir({ file: 'requirements.txt', ... });

  // 返回最高优先级的 manifest
  if (uvLockDir && pyprojectDir) {
    return { manifestType: 'uv.lock', manifestPath: join(uvLockDir, 'uv.lock') };
  } else if (pyprojectDir) {
    return { manifestType: 'pyproject.toml', manifestPath: join(pyprojectDir, 'pyproject.toml') };
  } else if (pipfileLockDir) {
    return { manifestType: 'Pipfile.lock', ... };
  }
  // ...
}
```

### 12.2 ensureUvProject() 统一处理

所有依赖格式都会转换为 `pyproject.toml` + `uv.lock`：

| 原始格式 | 处理方式 |
|---------|---------|
| `uv.lock` + `pyproject.toml` | 直接使用 |
| `pyproject.toml` | 运行 `uv lock` 生成 `uv.lock` |
| `Pipfile.lock` | 使用 `pipfile2req` 导出后 `uv add -r` |
| `Pipfile` | 使用 `pipfile2req` 导出后 `uv add -r` |
| `requirements.txt` | 直接 `uv add -r requirements.txt` |
| 无 manifest | 创建空 `pyproject.toml` |

### 12.3 Vendor 目录

依赖包被安装到 `_vendor/` 目录，然后在运行时通过 `site.addsitedir()` 添加到 `sys.path`：

```typescript
const vendorFiles = await mirrorSitePackagesIntoVendor({
  venvPath,
  vendorDirName: vendorDir,  // '_vendor'
});
for (const [p, f] of Object.entries(vendorFiles)) {
  files[p] = f;
}
```

---

## 13. uv 包管理工具

### 13.1 UvRunner 类 (`packages/python/src/uv.ts`)

```typescript
export const UV_VERSION = '0.9.22';

export class UvRunner {
  private uvPath: string;

  constructor(uvPath: string) {
    this.uvPath = uvPath;
  }

  // 列出已安装的 Python 版本
  listInstalledPythons(): Set<string> {
    const result = execSync(`${this.uvPath} python list --only-installed`);
    // 解析输出
  }

  // 同步依赖 (uv sync)
  async sync(options: { venvPath: string; projectDir: string; locked?: boolean }) {
    const args = ['sync', '--active', '--no-dev', '--link-mode', 'copy'];
    if (options.locked) args.push('--locked');
    await execa(this.uvPath, args, { cwd: options.projectDir, env: this.getEnv(options.venvPath) });
  }

  // 锁定依赖 (uv lock)
  async lock(projectDir: string) {
    await execa(this.uvPath, ['lock'], { cwd: projectDir });
  }

  // 添加依赖 (uv add)
  async addDependencies(options: { venvPath: string; projectDir: string; dependencies: string[] }) {
    await execa(this.uvPath, ['add', ...options.dependencies], { ... });
  }

  // pip 命令 (uv pip)
  async pip(options: { venvPath: string; projectDir: string; args: string[] }) {
    await execa(this.uvPath, ['pip', ...options.args], { ... });
  }
}
```

### 13.2 uv 的优势

| 特性 | 说明 |
|------|------|
| **速度** | 比 pip 快 10-100x |
| **统一管理** | 可以处理所有 Python 依赖格式 |
| **确定性** | 通过 lock 文件确保可重复构建 |
| **空间效率** | 使用硬链接减少磁盘占用 |

---

## 14. Lambda 打包与 Runtime Trampoline

### 14.1 Runtime Trampoline

Runtime Trampoline 是一个小型 Python 脚本，负责：

1. 设置环境变量
2. 配置 `sys.path` 以包含 `_vendor/` 目录
3. 导入 `vercel_runtime.vc_init` 模块
4. 暴露 `vc_handler` 函数给 Lambda

```python
# vc__handler__python.py (生成的)
import importlib
import os
import os.path
import site
import sys

_here = os.path.dirname(__file__)

# 设置环境变量
os.environ.update({
  "__VC_HANDLER_MODULE_NAME": "app",           # 用户模块名
  "__VC_HANDLER_ENTRYPOINT": "app.py",         # 入口文件
  "__VC_HANDLER_ENTRYPOINT_ABS": os.path.join(_here, "app.py"),
  "__VC_HANDLER_VENDOR_DIR": "_vendor",
})

# 配置 vendor 目录
_vendor = os.path.normpath(os.path.join(_here, '_vendor'))
if os.path.isdir(_vendor):
    site.addsitedir(_vendor)
    # 将 _vendor 移到 sys.path 前面
    while _vendor in sys.path:
        sys.path.remove(_vendor)
    sys.path.insert(1, _vendor)
    importlib.invalidate_caches()

# 导入处理器
from vercel_runtime.vc_init import vc_handler
```

### 14.2 Lambda 输出结构

```typescript
const output = new Lambda({
  files: {
    'vc__handler__python.py': FileBlob,  // Runtime Trampoline
    'app.py': FileFsRef,                  // 用户入口
    '_vendor/': { ... },                  // 所有依赖包
    // ... 其他用户文件
  },
  handler: 'vc__handler__python.vc_handler',
  runtime: 'python3.12',
  environment: { PYTHONPATH: '_vendor' },
  supportsResponseStreaming: true,
});
```

### 14.3 Lambda 目录结构

```
/var/task/
├── vc__handler__python.py    # Runtime Trampoline
├── app.py                    # 用户入口
├── _vendor/                  # 依赖包
│   ├── fastapi/
│   ├── starlette/
│   ├── pydantic/
│   ├── vercel_runtime/
│   │   └── vc_init.py       # WSGI/ASGI 适配器
│   └── ...
└── [其他用户文件]
```

---

## 15. 生产环境 WSGI/ASGI 处理

### 15.1 vc_init.py 概述

`vercel_runtime.vc_init` 是生产环境的核心模块，负责：

1. **IPC 通信**：与 Vercel 平台通信
2. **日志处理**：捕获并转发日志到平台
3. **请求上下文**：管理请求级别的上下文变量
4. **WSGI/ASGI 适配**：将 HTTP 请求转换为 WSGI/ASGI 调用

### 15.2 ASGI 中间件

```python
# python/vercel-runtime/src/vercel_runtime/vc_init.py
class ASGIMiddleware:
    """
    ASGI 中间件:
    - 处理 /_vercel/ping 健康检查
    - 提取 x-vercel-internal-* headers
    - 设置请求上下文 (用于日志/指标)
    - 发送 handler-started / end IPC 消息
    """
    def __init__(self, app):
        self.app = app

    async def __call__(self, scope, receive, send):
        if scope.get('type') != 'http':
            await self.app(scope, receive, send)
            return

        # 健康检查
        if scope.get('path') == '/_vercel/ping':
            await send({'type': 'http.response.start', 'status': 200, 'headers': []})
            await send({'type': 'http.response.body', 'body': b'', 'more_body': False})
            return

        # 提取内部 headers
        headers_list = scope.get('headers', [])
        new_headers = []
        invocation_id = "0"
        request_id = 0

        for k, v in headers_list:
            key = k.decode().lower()
            if key == 'x-vercel-internal-invocation-id':
                invocation_id = v.decode()
                continue
            if key == 'x-vercel-internal-request-id':
                request_id = int(v.decode())
                continue
            if key.startswith('x-vercel-internal-'):
                continue
            new_headers.append((k, v))

        # 发送 handler-started 消息
        send_message({
            "type": "handler-started",
            "payload": {
                "handlerStartedAt": int(time.time() * 1000),
                "context": { "invocationId": invocation_id, "requestId": request_id }
            }
        })

        # 设置请求上下文
        token = storage.set({ "invocationId": invocation_id, "requestId": request_id })

        try:
            new_scope = dict(scope)
            new_scope['headers'] = new_headers
            await self.app(new_scope, receive, send)
        finally:
            storage.reset(token)
            # 发送 end 消息
            send_message({
                "type": "end",
                "payload": { "context": { "invocationId": invocation_id, "requestId": request_id } }
            })
```

### 15.3 WSGI Handler

```python
class Handler(BaseHandler):
    def handle_request(self):
        # 构建 WSGI environ
        if '?' in self.path:
            path, query = self.path.split('?', 1)
        else:
            path, query = self.path, ''
        
        content_length = int(self.headers.get('Content-Length', 0))
        body = self.rfile.read(content_length) if content_length else b''
        
        environ = {
            'REQUEST_METHOD': self.command,
            'SCRIPT_NAME': '',
            'PATH_INFO': path,
            'QUERY_STRING': query,
            'CONTENT_TYPE': self.headers.get('content-type', ''),
            'CONTENT_LENGTH': str(content_length),
            'SERVER_NAME': 'localhost',
            'SERVER_PORT': '80',
            'SERVER_PROTOCOL': self.request_version,
            'wsgi.version': (1, 0),
            'wsgi.url_scheme': 'https',
            'wsgi.input': BytesIO(body),
            'wsgi.errors': sys.stderr,
            'wsgi.multithread': False,
            'wsgi.multiprocess': True,
            'wsgi.run_once': False,
        }
        
        # 添加 HTTP headers
        for key, value in self.headers.items():
            key = key.upper().replace('-', '_')
            if key not in ('CONTENT_TYPE', 'CONTENT_LENGTH'):
                environ['HTTP_' + key] = value
        
        # 调用用户应用
        response = app(environ, self.start_response)
        
        # 发送响应
        for chunk in response:
            self.wfile.write(chunk)
```

### 15.4 日志处理

```python
def setup_logging(send_message, storage):
    class VCLogHandler(logging.Handler):
        def emit(self, record):
            message = record.getMessage()
            level = "info"
            if record.levelno >= logging.ERROR:
                level = "error"
            elif record.levelno >= logging.WARNING:
                level = "warn"
            
            context = storage.get()
            if context:
                send_message({
                    "type": "log",
                    "payload": {
                        "context": {
                            "invocationId": context['invocationId'],
                            "requestId": context['requestId'],
                        },
                        "message": base64.b64encode(message.encode()).decode(),
                        "level": level,
                    }
                })

    # 重写 stdout/stderr
    class StreamWrapper:
        def write(self, message):
            context = storage.get()
            if context:
                send_message({
                    "type": "log",
                    "payload": {
                        "context": { ... },
                        "message": base64.b64encode(message.encode()).decode(),
                        "stream": self.stream_name,
                    }
                })

    sys.stdout = StreamWrapper(sys.stdout, "stdout")
    sys.stderr = StreamWrapper(sys.stderr, "stderr")
```

---

# 第三部分：dev 与 build 的对比

## 16. 核心差异对比

### 16.1 架构对比

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
│  startDevServer()            │        │  build()                     │
│  • 仅 FastAPI/Flask          │        │  • 所有 Python 框架          │
│  • 使用用户 venv              │        │  • 创建新 venv               │
│  • uvicorn/werkzeug          │        │  • uv 安装依赖               │
└──────────────┬───────────────┘        └──────────────┬───────────────┘
               │                                       │
               ▼                                       ▼
┌──────────────────────────────┐        ┌──────────────────────────────┐
│  vc_init_dev_*.py            │        │  vercel_runtime.vc_init      │
│  • 简单的静态文件中间件       │        │  • 完整的 IPC 协议           │
│  • 直接调用用户 app           │        │  • 日志/指标收集             │
└──────────────────────────────┘        └──────────────────────────────┘
```

### 16.2 Shim/Handler 对比

| 特性 | Dev Shim | Production Handler |
|------|---------|-------------------|
| **文件** | `vc_init_dev_asgi.py` / `vc_init_dev_wsgi.py` | `vercel_runtime.vc_init` |
| **静态文件** | 内置中间件 | 无 |
| **IPC 通信** | 无 | 完整支持 |
| **日志处理** | 直接输出 | 通过 IPC 转发 |
| **健康检查** | 无 | `/_vercel/ping` |
| **请求上下文** | 无 | 完整支持 |

### 16.3 Python 版本

| 模式 | 版本选择 | 说明 |
|------|---------|------|
| **Dev** | 系统 `python3` | 使用用户已安装的版本 |
| **Build** | 指定版本 (如 `python3.12`) | 从配置文件检测或使用默认 |

### 16.4 虚拟环境

| 模式 | 虚拟环境 | 说明 |
|------|---------|------|
| **Dev** | 用户的 `.venv` / `venv` | 假设用户已安装依赖 |
| **Build** | `.vercel/python/.venv` | 自动创建并安装依赖 |

### 16.5 依赖安装

| 模式 | 安装方式 | 说明 |
|------|---------|------|
| **Dev** | 不安装 | 假设用户已在虚拟环境中安装 |
| **Build** | 使用 `uv` 自动安装 | 统一处理所有依赖格式 |

---

## 17. 共享组件分析

### 17.1 共享的组件

| 组件 | 文件 | 用途 |
|------|------|------|
| **入口点检测** | `entrypoint.ts` | 检测 `app.py`、`index.py` 等 |
| **版本管理** | `version.ts` | Python 版本检测和选择 |
| **虚拟环境工具** | `utils.ts` | 虚拟环境检测和配置 |

### 17.2 入口点检测

```typescript
// 两种模式共用
export const PYTHON_CANDIDATE_ENTRYPOINTS = [
  'app.py', 'src/app.py', 'app/app.py', 'api/app.py',
  'index.py', 'src/index.py', 'app/index.py', 'api/index.py',
  'server.py', 'src/server.py', 'app/server.py', 'api/server.py',
  'main.py', 'src/main.py', 'app/main.py', 'api/main.py',
  'wsgi.py', 'src/wsgi.py', 'app/wsgi.py', 'api/wsgi.py',
  'asgi.py', 'src/asgi.py', 'app/asgi.py', 'api/asgi.py',
];

export async function detectPythonEntrypoint(
  framework: PythonFramework,
  workPath: string,
  configuredEntrypoint: string
): Promise<string | null> {
  // 1. 尝试配置的入口点
  // 2. 搜索候选文件
  // 3. 检查 pyproject.toml
}
```

### 17.3 框架检测

```typescript
// packages/build-utils/src/framework-helpers.ts
export const PYTHON_FRAMEWORKS = ['fastapi', 'flask', 'python'] as const;

export function isPythonFramework(framework): framework is PythonFramework {
  return PYTHON_FRAMEWORKS.includes(framework);
}
```

---

## 18. 完整对比总结表

### 18.1 功能对比

| 特性 | dev 命令 | build 命令 |
|------|---------|-----------|
| **主要用途** | 本地开发调试 | 生产部署 |
| **执行环境** | 本地机器 | Vercel 云平台 |
| **支持框架** | FastAPI, Flask | 所有 Python 框架 |
| **Python 版本** | 系统 `python3` | 指定版本 (3.9-3.14) |
| **依赖安装** | 不安装（假设已安装） | 使用 uv 自动安装 |
| **虚拟环境** | 用户的 `.venv` | `.vercel/python/.venv` |
| **静态文件** | 内置中间件 | 无特殊处理 |
| **热重载** | 支持（依赖框架） | 不适用 |
| **响应流** | 不支持 | 支持 |
| **IPC 通信** | 无 | 完整支持 |

### 18.2 文件对比

| 用途 | Dev 文件 | Build 文件 |
|------|---------|-----------|
| **ASGI 适配** | `vc_init_dev_asgi.py` | `vercel_runtime.vc_init` |
| **WSGI 适配** | `vc_init_dev_wsgi.py` | `vercel_runtime.vc_init` |
| **入口脚本** | shim (运行时生成) | `vc__handler__python.py` |
| **依赖目录** | 用户的 site-packages | `_vendor/` |

### 18.3 服务器对比

| 框架 | Dev Server | 生产运行时 |
|------|-----------|-----------|
| **FastAPI** | uvicorn / hypercorn | Lambda Python Runtime |
| **Flask** | Werkzeug / wsgiref | Lambda Python Runtime |
| **其他** | 不支持 | Lambda Python Runtime |

### 18.4 配置选项

| 配置 | dev 支持 | build 支持 |
|------|---------|-----------|
| `framework` | ✅ (仅 fastapi/flask) | ✅ |
| `python_version` | ❌ (使用系统版本) | ✅ |
| `installCommand` | ❌ | ✅ |
| `buildCommand` | ❌ | ✅ |
| `excludeFiles` | ❌ | ✅ |

### 18.5 使用场景建议

| 场景 | 推荐命令 | 说明 |
|------|---------|------|
| FastAPI 本地开发 | `vercel dev` | 支持热重载 |
| Flask 本地开发 | `vercel dev` | 支持热重载 |
| Django 开发 | `python manage.py runserver` | dev 命令不支持 |
| 预览部署 | `vercel` | 自动构建并部署 |
| 生产部署 | `vercel --prod` | 自动构建并部署 |

---

## 附录：项目配置示例

### FastAPI 项目

**项目结构:**
```
my-fastapi-app/
├── app.py              # 入口文件
├── requirements.txt    # 或 pyproject.toml
├── public/             # 静态文件 (可选)
│   └── favicon.ico
└── vercel.json
```

**app.py:**
```python
from fastapi import FastAPI

app = FastAPI()

@app.get("/")
def read_root():
    return {"Hello": "World"}

@app.get("/api/hello")
def hello():
    return {"message": "Hello from FastAPI"}
```

**vercel.json:**
```json
{
  "framework": "fastapi"
}
```

### Flask 项目

**项目结构:**
```
my-flask-app/
├── app.py
├── requirements.txt
└── vercel.json
```

**app.py:**
```python
from flask import Flask

app = Flask(__name__)

@app.route("/")
def home():
    return "Hello from Flask!"

@app.route("/api/hello")
def hello():
    return {"message": "Hello from Flask"}
```

**vercel.json:**
```json
{
  "framework": "flask"
}
```

### 指定 Python 版本

**.python-version:**
```
3.12
```

**或 pyproject.toml:**
```toml
[project]
name = "my-app"
requires-python = ">=3.11"
dependencies = [
    "fastapi>=0.100.0",
    "uvicorn>=0.22.0",
]
```

### 命令速查

```bash
# 本地开发
vercel dev

# 指定端口
vercel dev --listen 8080

# 部署到预览环境
vercel

# 部署到生产环境
vercel --prod

# 仅构建不部署
vercel build
```
