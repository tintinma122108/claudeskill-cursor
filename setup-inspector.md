# setup-inspector

为当前前端项目配置**浏览器点击 → Cursor 自动跳转组件文件**的开发能力。

---

## 第一阶段：自动探测项目环境

在执行任何安装前，先读取以下文件来判断项目环境：

1. `package.json` → 判断框架（react/vue/svelte）和构建工具（vite/next/webpack/parcel）
2. `vite.config.*` / `next.config.*` / `webpack.config.*` → 确认构建工具版本和配置风格
3. 入口文件（`src/main.*` / `src/index.*` / `pages/_app.*`）→ 确认渲染方式

**探测失败的处理规则（优先判断）：**

- 如果当前目录**没有 `package.json`**：
  > 「当前目录不像一个前端项目（没有找到 package.json）。请在项目根目录下运行 /setup-inspector，或者等项目初始化完成后再来。」
  > 停止执行。

- 如果 `package.json` 存在，但**没有任何已知框架依赖**（无 react / vue / svelte / solid 等）：
  > 「我还没检测到前端框架。等你把框架搭好后（比如 npm install react），再运行 /setup-inspector 就可以了。」
  > 停止执行。

- 如果框架可识别，但**构建工具未知**：
  > 「我看到你用的是 [框架名]，但找不到构建工具配置（vite.config / next.config / webpack.config）。你用的是哪个构建工具？」
  > 等用户回答后继续。

---

**根据探测结果选择方案：**

| 框架 | 构建工具 | 方案 |
|------|----------|------|
| React | Vite（任意版本） | findComponentPlugin + react-dev-inspector（见下文） |
| React | Next.js | react-dev-inspector + next.config 中间件 |
| React | CRA / webpack | react-dev-inspector + errorOverlayMiddleware |
| Vue | Vite | vite-plugin-vue-inspector |
| 其他 | 任意 | 告知用户当前暂不支持，说明原因 |

如果无法判断，**主动问用户**：「我看到你用的是 [X]，请确认框架和构建工具是否正确？」

---

## 第二阶段：安装与配置（React + Vite 标准流程）

### 安装依赖

```bash
npm install react-dev-inspector
```

> ⚠️ 不要安装 `click-to-component`，该包已从 npm 移除。

### 在 vite.config 中添加 findComponentPlugin

**重要背景：Vite 8+ 已切换到 OXC 编译器，不再走 Babel**，因此所有依赖 Babel 的插件（包括 `@react-dev-inspector/babel-plugin` 和 `InspectorBabelPlugin`）在 Vite 8+ 中**静默失效**，`codeInfo` 永远为 undefined，点击无反应。

解决方案：绕过 Babel，在 Vite dev server 添加一个根据组件名搜索文件的中间件。

在 vite.config 导入区加入：

```js
import fs from 'node:fs'
import path from 'node:path'
```

在 `defineConfig` 之前定义插件：

```js
function getAllSourceFiles(dir, results = []) {
  for (const entry of fs.readdirSync(dir, { withFileTypes: true })) {
    const full = path.join(dir, entry.name)
    if (entry.isDirectory()) getAllSourceFiles(full, results)
    else if (/\.(jsx|tsx|js|ts)$/.test(entry.name)) results.push(full)
  }
  return results
}

function findComponentFile(name, dir) {
  const files = getAllSourceFiles(dir)
  const re = new RegExp(`(function|const)\\s+${name}[\\s(<]`)
  // 优先匹配同名文件（如 Doodle.jsx）
  const exact = files.find(f => path.basename(f, path.extname(f)) === name)
  if (exact) {
    const lines = fs.readFileSync(exact, 'utf8').split('\n')
    const lineNumber = lines.findIndex(l => re.test(l)) + 1 || 1
    return { file: exact, lineNumber }
  }
  // 回退：在文件内容里搜索组件定义
  for (const f of files) {
    const lines = fs.readFileSync(f, 'utf8').split('\n')
    const idx = lines.findIndex(l => re.test(l))
    if (idx !== -1) return { file: f, lineNumber: idx + 1 }
  }
  return null
}

const findComponentPlugin = {
  name: 'find-component',
  configureServer(server) {
    server.middlewares.use('/__find-component', (req, res) => {
      const name = new URL(req.url, 'http://localhost').searchParams.get('name')
      const srcDir = path.join(process.cwd(), 'src')
      const result = name ? findComponentFile(name, srcDir) : null
      const absolutePath = result?.file ?? null
      const lineNumber = result?.lineNumber ?? 1
      const relativePath = absolutePath ? path.relative(process.cwd(), absolutePath) : null
      res.setHeader('Content-Type', 'application/json')
      res.end(JSON.stringify({ absolutePath, lineNumber, relativePath }))
    })
  },
}
```

将 `findComponentPlugin` 加入 plugins 数组首位：

```js
export default defineConfig({
  plugins: [findComponentPlugin, react()],
})
```

> ⚠️ `@react-dev-inspector/vite-plugin` 的导出名是 `inspectorServer`，不是 `InspectorPlugin`。如无必要可不引入，本方案不依赖它。

### 在入口文件挂载 Inspector

找到项目入口文件（通常是 `src/main.jsx` 或 `src/main.tsx`），加入：

```jsx
import { Inspector } from 'react-dev-inspector'
```

在根组件渲染处（`<App />` 之前）加入：

```jsx
{import.meta.env.DEV && (
  <Inspector
    onClickElement={async ({ name }) => {
      const res = await fetch(`/__find-component?name=${name}`)
      const { absolutePath, lineNumber, relativePath } = await res.json()
      if (absolutePath) {
        window.open(`cursor://file/${absolutePath}:${lineNumber}`)
        navigator.clipboard.writeText(`${relativePath}:${lineNumber}`)
      } else {
        console.warn(`[inspector] 找不到组件文件: ${name}`)
      }
    }}
  />
)}
```

---

## 第三阶段：验证流（必须逐项确认）

配置完成后，按以下顺序验证，每一步失败都有对应排查方案：

### ✅ 验证 1：Cursor 命令行工具已安装

```bash
which cursor
```

**预期结果：** 输出路径如 `/usr/local/bin/cursor`

**失败处理：** 在 Cursor 中执行菜单 `Shell Command: Install 'cursor' command in PATH`，重启终端后重试。

---

### ✅ 验证 2：Dev server 正常启动

重启 dev server（**必须**在 vite.config 修改后重启，热更新不会生效）：

```bash
npm run dev
```

**预期结果：** 终端显示 `Local: http://localhost:517x/`，无报错。

**失败处理：** 检查 vite.config 语法是否正确，Node.js 版本是否兼容。

---

### ✅ 验证 3：Inspector 组件已挂载

浏览器打开 `http://localhost:517x/`，打开 DevTools → Components 标签（需安装 React DevTools 扩展）。

**预期结果：** 组件树顶部能看到 `Inspector` 节点。

**失败处理：**
- 如果 Components 标签不存在：安装 Chrome 扩展 [React Developer Tools](https://chrome.google.com/webstore/detail/react-developer-tools)
- 如果 Inspector 节点不存在：检查入口文件是否正确引入并渲染了 `<Inspector />`
- 按 `Command+Shift+R` 强制刷新（清除缓存）

---

### ✅ 验证 4：点击触发网络请求

打开 DevTools → 网络标签，按 `Ctrl+Shift+Command+C` 激活检查模式，点击页面任意元素。

**预期结果：** 网络列表出现 `__find-component?name=XXX` 请求，状态 200。

**失败处理：**
- 如果快捷键没有激活蓝色高亮：说明 Inspector 未挂载，回到验证 3
- 如果激活了高亮但无网络请求：在控制台执行 `document.querySelector('[data-inspector-host]')` 检查返回值；尝试删除 `import.meta.env.DEV &&` 条件，强制渲染 Inspector
- 如果有请求但状态非 200：检查 `findComponentPlugin` 是否正确加入 vite.config

---

### ✅ 验证 5：Cursor 成功打开文件

点击一个元素，观察 Cursor 是否自动跳转。

**预期结果：** Cursor 前台激活并打开对应组件文件。

**失败处理：**
- 如果网络请求返回 `absolutePath: null`：组件定义在某个文件内部（非独立文件），这是正常情况，`findComponentFile` 会用正则在文件内容中搜索定义；如仍找不到，检查源码目录是否不叫 `src`，需修改 `srcDir` 路径
- 如果有请求且有 `absolutePath` 但 Cursor 无反应：重新验证 `which cursor`，确认 Cursor 版本支持 `cursor://` URL schema（Cursor 1.0+ 均支持）

---

## 已知 Bug 案例库

每次遇到新问题，将其记录在此处，形成可复用的诊断经验。

---

### Bug #001：`codeInfo` 始终为 undefined，点击无响应

**症状：** Inspector 高亮正常，点击后 `onClickElement` 回调的 `codeInfo` 为 `undefined`，没有网络请求发出。

**根因：** Vite 8+ 使用 OXC 编译器替代 Babel，`@react-dev-inspector/babel-plugin` 和 `InspectorBabelPlugin` 不会被执行，导致 JSX 元素没有注入文件路径信息，Inspector 找不到 `codeInfo` 后静默退出。

**修复：** 使用本 skill 的 `findComponentPlugin` 方案，绕过 Babel 依赖，改为服务端文件搜索。

---

### Bug #002：`@react-dev-inspector/vite-plugin` 导出名错误

**症状：** `import { InspectorPlugin } from '@react-dev-inspector/vite-plugin'` 报错，找不到导出。

**根因：** v2 版本实际导出名为 `inspectorServer`，不是 `InspectorPlugin`。

**修复：** 改为 `import { inspectorServer } from '@react-dev-inspector/vite-plugin'`。本方案已不依赖此包，可忽略。

---

### Bug #003：`click-to-component` 包不存在

**症状：** `npm install click-to-component` 报 404。

**根因：** 该包已从 npm 移除。

**修复：** 改用 `react-dev-inspector`。

---

### Bug #004：终端 curl 测试时 zsh 报 glob 错误

**症状：** `curl http://localhost:5173/__open-in-editor?fileName=App.jsx` 报 `zsh: no matches found`。

**根因：** zsh 将 URL 中的 `?` 解释为 glob 通配符。

**修复：** 用单引号包裹 URL：`curl 'http://...'`；或转义问号：`curl http://.../__open-in-editor\?fileName=App.jsx`。

---

### Bug #005：切换 dev server 后浏览器仍显示 "server connection lost"

**症状：** 重启 dev server 后，浏览器控制台仍报 `[vite] server connection lost`，功能无响应。

**根因：** 浏览器仍在尝试连接旧的 server 实例，页面未刷新。

**修复：** 按 `Command+R`（或 `Command+Shift+R`）刷新浏览器页面。

---

### Bug #006：在错误目录运行 npm run dev

**症状：** `npm error enoent Could not read package.json`。

**根因：** 终端当前目录是项目根目录，但 `package.json` 在子目录（如 `app/`）里。

**修复：** 先 `cd app`（或对应子目录），再运行 `npm run dev`。

---

*如遇新问题，请将症状、根因、修复方式按以上格式追加到案例库。*
