---
name: dsh-plugin-dev
description: DSH (DeepSeek Harness) 插件从零开发到发布的完整指南。适用于：创建新插件骨架、注册 Tool/Command/WebAPI、构建 Web UI 面板、CLI 工具封装、npm 发布与 GitHub Actions CI/CD、插件市场上架。基于 Cordis 框架，DSH 0.2.x。
---

# DSH 插件开发 Skill

基于 dsh-ops（SSH/K8s/Helm/ArgoCD/Argo Workflows 一体化运维插件）的完整开发实战经验。

---

## 1. 项目骨架

```
my-dsh-plugin/
├── package.json          # 入口 + dsh 配置（必须）
├── cordis.patch.yml      # bundle patch，注册到加载器（必须）
├── index.js              # Host 端入口（必须，export { apply, inject, name }）
├── lib/
│   ├── client.js         # Client 端入口（可选，Web UI 面板）
│   ├── mod-xxx.mjs       # 业务模块（按功能拆分）
│   └── mod-yyy.mjs
├── web/                  # SPA 前端源码（可选，复杂 UI 时用）
│   ├── index.html
│   ├── app.js
│   └── style.css
├── scripts/
│   ├── build-ui.mjs      # 构建脚本（将 web/ 打包为 lib/client.js）
│   └── doctor.mjs        # 依赖检查脚本
├── .github/
│   └── workflows/
│       └── publish.yml   # GitHub Actions 自动发布到 npm
├── .gitignore
├── LICENSE
└── README.md
```

## 2. package.json 关键字段

```json
{
  "name": "@scope/dsh-my-plugin",
  "version": "0.1.0",
  "private": false,
  "description": "插件描述",
  "keywords": ["dsh", "dsh-plugin"],
  "type": "module",
  "main": "index.js",
  "exports": {
    ".": "./index.js",
    "./client": "./lib/client.js",
    "./cordis.patch.yml": "./cordis.patch.yml",
    "./package.json": "./package.json"
  },
  "scripts": {
    "build": "node scripts/build-ui.mjs",
    "prepublishOnly": "node scripts/build-ui.mjs",
    "doctor": "node scripts/doctor.mjs"
  },
  "dsh": {
    "bundle": {
      "patch": "./cordis.patch.yml"
    },
    "client": {
      "platform": "web",
      "immediately": true,
      "inject": [
        "@deepseek-ai/dsh-client-runtime",
        "@deepseek-ai/dsh-client-ui-renderer",
        "@deepseek-ai/dsh-client-ui-layout",
        "@deepseek-ai/dsh-client-ui-sidebar"
      ]
    }
  },
  "files": [
    "index.js",
    "lib/",
    "cordis.patch.yml",
    "scripts/doctor.mjs"
  ],
  "repository": {
    "type": "git",
    "url": "https://github.com/OWNER/REPO.git"
  },
  "license": "MIT",
  "engines": { "node": ">=22" },
  "peerDependencies": {
    "@deepseek-ai/dsh": ">=0.2.0-rc.2 <0.3.0-0",
    "@deepseek-ai/dsh-tools": ">=0.2.0-rc.2 <0.3.0-0"
  },
  "peerDependenciesMeta": {
    "@deepseek-ai/dsh": { "optional": true },
    "@deepseek-ai/dsh-tools": { "optional": true }
  }
}
```

**要点：**
- `"type": "module"` 必须 ESM
- `dsh.bundle.patch` 指向 cordis.patch.yml
- `dsh.client` 声明浏览器端面板
- `dsh.client.immediately: true` 页面加载时立即执行 client.js（侧边栏插件需要）
- `dsh.client.inject` 列出依赖的运行时模块
- peer deps 标 optional — 缺一个 peer 只丢一个能力，不许整插件崩
- `files` 只包含发布到 npm 的文件
- scoped package 发布用 `npm publish --access public`

## 3. cordis.patch.yml

```yaml
- insert:
    - id: dsh-my-plugin
      name: "@scope/dsh-my-plugin"
```

这是 Cordis 加载器的 patch 指令，把自己注册进插件列表。

**关键踩坑：**
- cordis.patch.yml 是**整体替换**，不是 deep merge
- **不要**加 `disabled: true`，否则插件页显示已启用但实际不加载
- `name` 必须和 package.json 的 `name` 一致
- profile 的 `dsh.profile.bundles` 列出包名即整层生效

## 4. index.js — Cordis 插件核心

```js
import { fileURLToPath } from 'node:url'
import { dirname, join } from 'node:path'

const HERE = dirname(fileURLToPath(import.meta.url))

const inject = ['webServer']
const name = '@scope/dsh-my-plugin'

function apply(ctx) {
  // 注册各种能力
  registerTools(ctx)
  registerRoutes(ctx)
}

export { apply, inject, name }
```

**必须 export `{ apply, inject, name }`**：
- `apply(ctx)` — 插件入口，接收 Cordis context
- `inject` — 声明依赖的服务（如 `'commands'`, `'webServer'`, `'agents'`）
- `name` — 插件标识，和 package.json name 一致

## 5. Tool 注册（AI 可调用）

```js
let defineTool = null
try { ({ defineTool } = await import('@deepseek-ai/dsh-tools')) } catch { defineTool = null }

function registerTools(ctx) {
  const toolsSvc = (() => { try { return ctx.get?.('tools') } catch { return null } })()
  if (!defineTool || !toolsSvc?.register) return

  toolsSvc.register(defineTool({
    name: 'my_tool_name',
    description: '详细说明什么时候应该使用这个工具、它能做什么',
    parameters: {
      host: { type: 'string', required: true, description: '目标主机名' },
      command: { type: 'string', required: true, description: '要执行的命令' },
      timeout: { type: 'number', description: '超时秒数，默认 60' },
    },
    output: { type: 'object', properties: {}, additionalProperties: true },
    execute: async (params) => {
      const result = await doSomething(params)
      return textOutput(result)
    },
  }))
}
```

**defineTool 格式规则（必须严格遵守）：**
- parameters 是**扁平的** `{ key: { type, description, required? } }`，不要用 JSON Schema 的 `type:'object', properties` 包裹
- 用 `execute` 而非 `handler`
- **必须用 `defineTool` 包一层**，直接传 options 给 register 会让 parameters 变 null
- output 的 `additionalProperties` 必须显式写 `true` 或 `false`
- output 的 `required` 不允许出现
- 最安全的 output 写法：`{ type: 'object', properties: {}, additionalProperties: true }`
- **parameters 中 `type: 'object'` 同样要求** `additionalProperties` 为布尔值，不能用 schema 对象如 `{ type: 'string' }`。写法：`{ type: 'object', properties: {}, additionalProperties: true }`

**textOutput 辅助函数（统一文本输出格式）：**

```js
function textOutput(text) {
  return { type: 'text', text: typeof text === 'string' ? text : JSON.stringify(text, null, 2) }
}
```

## 6. 斜杠命令（人敲，确定性执行）

```js
ctx.commands.register({
  definitionId: 'my-plugin',
  name: 'mycmd',
  description: '做某件事',
  input: { hint: '<sub> [args] [--flag]' },
  handler: async (invocation) => {
    const result = await doSomething(invocation.rawInput)
    return { kind: result.ok ? 'success' : 'error', text: result.text }
  },
})
```

需要在 `inject` 中声明 `'commands'`。

## 7. Web API 端点（面板后端）

```js
function registerRoutes(ctx) {
  const routes = {
    '/api/my-plugin/status': handleStatus,
    '/api/my-plugin/items': handleItems,
  }

  for (const [path, handler] of Object.entries(routes)) {
    if (!claimRoute(path)) continue
    ctx.effect(() => {
      ctx.webServer.register({ path, handler })
      return () => { releaseRoute(path) }
    }, `my-plugin: ${path}`)
  }
}

function sendJson(res, code, data) {
  res.writeHead(code, { 'content-type': 'application/json' })
  res.end(JSON.stringify(data))
}
```

**路由守卫（防 Cordis fiber 重放）：**

```js
const CLAIMED = new Set()
function claimRoute(path) {
  if (CLAIMED.has(path)) return false
  CLAIMED.add(path); return true
}
function releaseRoute(path) { CLAIMED.delete(path) }
```

Cordis fiber 重放时同一路径不能重复注册 webServer handler，否则路由永久 404。

## 8. Agent 钩子（拦截消息流）

```js
const { createUserMessage } = await import('@deepseek-ai/dsh-llm')

ctx.on('agent/pre-step', async (payload, next) => {
  const decision = await next()
  if (decision.kind !== 'enter' || payload.step !== 1) return decision
  const extra = createUserMessage({
    content: [{ type: 'text', text: '[my-plugin] 注入的上下文信息...' }],
    source: { kind: name, form: 'notice', summary: '...' },
  })
  return { ...decision, messages: [...decision.messages, extra] }
})
```

## 9. Client 端 UI（简单面板）

```js
const MODULE_LOADER = typeof window !== 'undefined' && window.__ModuleLoader__
  && typeof window.__ModuleLoader__.load === 'function'
  ? window.__ModuleLoader__
  : (() => { throw new Error('需要 DSH >= 0.2.0-rc.2') })()
```

设置页面板用 `slots.inject('settings.section', ...)` 注册。
侧边栏面板用 `slots.inject('sidebar.panel', ...)` 注册。

**client.js UI 注册模式：**

```js
window.__ModuleLoader__.load({
  id: '@scope/dsh-my-plugin',
  factory: () => ({
    inject: ['slots'],
    apply(ctx) {
      ctx.slots.register('sidebar.panel', {
        id: 'my-plugin-panel',
        order: 100,
        label: '我的插件',
        icon: 'icon-name',
      }, () => {
        // 返回 React 元素
        return React.createElement('div', null, 'Hello')
      })
    },
  }),
})
```

## 10. SPA Web UI 架构（复杂面板）

对于复杂 UI（如运维控制台），采用 iframe + srcdoc 方案：

**web/ 目录结构：**
```
web/
├── index.html      # 主页面模板
├── app.js          # 应用逻辑（路由、状态管理、API 调用）
├── style.css       # 样式
└── components/     # UI 组件（可选）
```

**构建脚本 scripts/build-ui.mjs（将 SPA 内联到 client.js）：**

```js
import { readFileSync, writeFileSync } from 'node:fs'
import { join, dirname } from 'node:path'
import { fileURLToPath } from 'node:url'

const ROOT = dirname(dirname(fileURLToPath(import.meta.url)))
const html = readFileSync(join(ROOT, 'web/index.html'), 'utf8')
const css = readFileSync(join(ROOT, 'web/style.css'), 'utf8')
const js = readFileSync(join(ROOT, 'web/app.js'), 'utf8')

const fullHtml = html
  .replace('/* __CSS__ */', css)
  .replace('/* __JS__ */', js)

const escaped = JSON.stringify(fullHtml)

const clientJs = `
(function () {
  if (typeof window === 'undefined' || !window.__ModuleLoader__) return
  const HTML = ${escaped}
  window.__ModuleLoader__.load({
    id: '@scope/dsh-my-plugin',
    factory: () => ({
      inject: ['slots'],
      apply(ctx) {
        ctx.slots.register('sidebar.panel', {
          id: 'my-plugin-panel',
          order: 100,
          label: '我的插件',
        }, () => {
          const el = document.createElement('div')
          el.style.cssText = 'width:100%;height:100%'
          const iframe = document.createElement('iframe')
          iframe.srcdoc = HTML
          iframe.style.cssText = 'width:100%;height:100%;border:none'
          el.appendChild(iframe)
          return { element: el }
        })
      },
    }),
  })
})()
`

writeFileSync(join(ROOT, 'lib/client.js'), clientJs)
console.log('Built lib/client.js')
```

**iframe 与 Host 通信（postMessage 桥）：**

```js
// 在 iframe 内的 app.js：
async function api(path, options = {}) {
  const resp = await fetch(path, options)
  return resp.json()
}

// 直接用 fetch 调用 Host 端的 /api/my-plugin/* 路由
// iframe 和 Host 同源，不需要 CORS
```

## 11. CLI 工具封装（exec 模式）

运维类插件通过子进程调用外部 CLI（kubectl、helm、ssh 等）：

```js
import { execFile } from 'node:child_process'

function exec(bin, args, options = {}) {
  return new Promise((resolve) => {
    execFile(bin, args, {
      timeout: options.timeout || 30000,
      maxBuffer: 10 * 1024 * 1024,
      env: { ...process.env, ...options.env },
    }, (err, stdout, stderr) => {
      resolve({
        ok: !err,
        code: err?.code ?? 0,
        stdout: stdout.trim(),
        stderr: stderr.trim(),
      })
    })
  })
}
```

**使用示例：**

```js
// kubectl
const result = await exec('kubectl', ['get', 'pods', '-n', ns, '-o', 'json'])
if (result.ok) return textOutput(result.stdout)

// ssh
const result = await exec('ssh', ['-o', 'BatchMode=yes', host, command])

// helm
const result = await exec('helm', ['list', '-n', ns, '-o', 'json'])
```

## 12. 模块化架构

按功能拆分为独立模块文件：

```js
// lib/mod-ssh.mjs — SSH 模块
export function registerSSH(ctx, { toolsSvc, defineTool, dataDir }) {
  // 注册 SSH 相关 tools
  toolsSvc.register(defineTool({ name: 'ssh_exec', ... }))
  toolsSvc.register(defineTool({ name: 'ssh_hosts_list', ... }))
}

// lib/mod-k8s.mjs — K8s 模块
export function registerK8s(ctx, { toolsSvc, defineTool, dataDir }) {
  toolsSvc.register(defineTool({ name: 'k8s_get', ... }))
}
```

**在 index.js 中按配置加载：**

```js
const config = loadConfig()

if (config.ssh?.enabled !== false) {
  const { registerSSH } = await import('./lib/mod-ssh.mjs')
  registerSSH(ctx, shared)
}

if (config.k8s?.enabled !== false) {
  const { registerK8s } = await import('./lib/mod-k8s.mjs')
  registerK8s(ctx, shared)
}
```

## 13. 数据存储

```
~/.dsh/<plugin-name>/          # 插件私有数据（不在包内！）
  ├── config.json              # 用户配置
  ├── data/                    # 持久化数据
  └── cache/                   # 缓存
```

**必须在 `~/.dsh/` 下建目录**，不要存在 `node_modules` 包内——npm update/uninstall 会删掉。

```js
import { homedir } from 'node:os'
import { join } from 'node:path'
import { mkdirSync, readFileSync, writeFileSync, existsSync } from 'node:fs'

function dataDir(sub = '') {
  const d = join(homedir(), '.dsh', 'my-plugin', sub)
  mkdirSync(d, { recursive: true })
  return d
}

function loadConfig() {
  const f = join(dataDir(), 'config.json')
  try { return JSON.parse(readFileSync(f, 'utf8')) } catch { return {} }
}
```

## 14. 依赖检查脚本 doctor.mjs

```js
#!/usr/bin/env node
import { execFileSync } from 'node:child_process'

const deps = [
  { bin: 'ssh',     module: 'ssh',    install: 'macOS 自带',           required: true },
  { bin: 'kubectl', module: 'k8s',    install: 'brew install kubectl', required: true },
  { bin: 'helm',    module: 'helm',   install: 'brew install helm',    required: false },
]

let ok = true
for (const d of deps) {
  try {
    const v = execFileSync(d.bin, ['version', '--short'], { timeout: 5000 }).toString().trim()
    console.log(`  ${d.bin}: ${v}`)
  } catch {
    const tag = d.required ? 'MISSING' : 'optional'
    console.log(`  ${d.bin}: ${tag} — ${d.install}`)
    if (d.required) ok = false
  }
}
process.exit(ok ? 0 : 1)
```

## 15. npm 发布

### 15.1 Scoped Package

```bash
# 首次发布
npm publish --access public

# 后续版本
npm version patch
npm publish --access public
```

Scoped package（`@scope/name`）默认 private，必须加 `--access public`。

### 15.2 GitHub Actions 自动发布

`.github/workflows/publish.yml`：

```yaml
name: Publish to npm
on:
  push:
    tags:
      - 'v*'
jobs:
  publish:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      id-token: write
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 22
          registry-url: https://registry.npmjs.org
      - run: npm publish --access public --provenance
        env:
          NODE_AUTH_TOKEN: ${{ secrets.NPM_TOKEN }}
```

**配置步骤：**
1. 在 npm 创建 Granular Access Token（勾选 "Bypass two-factor authentication"）
2. 在 GitHub repo Settings → Secrets → Actions 添加 `NPM_TOKEN`
3. 推送 tag 触发：`git tag v0.1.0 && git push origin v0.1.0`

### 15.3 npm Token 注意事项

- npm 现在只提供 Security Key（WebAuthn）作为 2FA 方式，不再支持 TOTP
- 自动发布必须用 Granular Access Token + "Bypass 2FA" 选项
- 首次发布可能进入 staged 状态（0.0.0-stage），需在 npm 网页确认

## 16. 插件市场上架

DSH 插件市场使用 curated catalog（`awesome-dsh-plugin` 仓库），不是 npm 关键字搜索。

**提交步骤：**

1. Fork `awesome-dsh-plugin/awesome-dsh-plugin`
2. 在 `data/plugins/` 下创建 `<owner>__<repo>.yml`：

```yaml
url: https://github.com/OWNER/REPO
name: OWNER/REPO
category: remote
npm: "@scope/dsh-my-plugin"
description:
  en: One-line English description.
  zh: "一行中文描述。"
```

3. 提交 PR 到上游 `main` 分支

**category 可选值：** `remote`, `entertainment`, `tool`, `social`, `productivity`, `developer`, `education` 等。

## 17. 开发 & 调试流程

```bash
# 1. 创建插件目录
mkdir -p ~/.dsh/my-dev/dsh-my-plugin && cd $_

# 2. 初始化文件（package.json, index.js, cordis.patch.yml）

# 3. 链接到 profile
ln -s $(pwd) ~/.dsh/profiles/desktop/node_modules/@scope/dsh-my-plugin

# 4. 在 profile 的 package.json 中添加依赖和 bundles

# 5. 重启 DSH

# 6. 验证
dsh --profile desktop --dump-config | grep my-plugin

# 7. 开发迭代（修改代码 → 重启 DSH → 验证）
# 重要：bundles 列表只在启动时读，代码修改后必须重启

# 8. 构建
node scripts/build-ui.mjs

# 9. 发布
git init && git add . && git commit -m "v0.1.0: initial release"
gh repo create owner/repo --public --source . --push
npm publish --access public
```

## 18. 关键踩坑清单

1. **cordis.patch.yml 是整体替换** — 不是 deep merge，写 patch 要包含所有配置
2. **副作用放 apply() 里** — 模块级代码在 import 时执行，dump-config 也会触发
3. **ctx.effect() 的 dispose 必须释放资源** — 路由、锁、定时器都要清理
4. **ESM 模块单例** — 进程级状态用模块级变量（一个进程只有一份）
5. **defineTool parameters 格式** — 扁平 `{ key: { type, description } }`，不是 JSON Schema
6. **defineTool output schema** — `additionalProperties` 必须显式写布尔值，`required` 不允许
7. **defineTool parameters 中的 object** — `additionalProperties` 也必须是布尔值 `true`/`false`，不能用 `{ type: 'string' }` 等 schema 对象
7. **必须用 defineTool 包裹** — 直接传 options 给 register 会让 parameters 变 null
8. **profile 安装后必须重启 DSH** — bundles 列表只在启动时读
9. **不要在 cordis.patch.yml 加 disabled: true** — 会导致插件页显示已启用但实际不加载
10. **路由重复注册** — Cordis fiber 重放时必须用 claimRoute 守卫，否则路由永久 404
11. **Config 导出** — 不导出 Config 时用 `apply(ctx)` 一参；导出 Config schema 时用 `apply(ctx, config)` 二参。不要用 `ctx.config`
12. **npm scoped 包发布** — 默认 private，必须 `--access public`
13. **npm 2FA** — 只有 Security Key 选项，自动发布用 Granular Access Token + Bypass 2FA
14. **插件市场是 curated** — 必须向 awesome-dsh-plugin 提 PR，不是自动发现
15. **peer deps 防御导入** — 每个可选依赖都用 try/catch 包裹，缺了只降级不崩
16. **client.js 模块注册 ID 必须用完整包名** — `__ModuleLoader__.load({ id: "@scope/pkg-name" })` 而非裸名 `"pkg-name"`。DSH 按 npm 包名匹配 client 模块，ID 不一致会报 `duplicate factory` / `import failed`
17. **发版验收必须覆盖前端** — 仅检查"运行中"会漏掉 client 加载失败。完整验收：后端启动 → 工具注册数量 → 侧栏入口出现 → 面板能打开并连接

## 19. 长文本 token 保护

```js
function trimForModel(text, maxLen = 2000) {
  if (text.length <= maxLen) return text
  return text.slice(0, maxLen * 0.4) + `\n\n...(truncated, ${text.length} chars total)`
}
```

工具返回超长文本时截断，避免消耗模型 token 预算。

## 20. 完整 dsh-ops 参考

实战产出：`@hua-ri/dsh-ops`（https://github.com/hua-ri/dsh-ops）
- 5 个功能模块：SSH / K8s / Helm / ArgoCD / Argo Workflows
- 49 个 AI tools + 完整 Web UI 控制台
- iframe + srcdoc SPA 架构
- GitHub Actions 自动发布到 npm
- 已上架 DSH 插件市场
