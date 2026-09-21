---
name: electron-development
description: 基于 Electron 官方中文文档的开发指导 Skill。用于创建、设计、实现、审查、调试和发布 Electron 应用，覆盖应用架构、进程模型、窗口、IPC、协议、安全、性能、打包、分发、自动更新和测试。
---

# Electron 开发指导 Skill

本 Skill 以 [Electron 官方中文文档](https://www.electronjs.org/zh/docs/latest/) 为主要依据。处理 Electron 任务时，优先遵循官方 API、教程、安全建议和版本文档；如果项目 Electron 版本与文档最新版本不同，先确认项目版本，再查阅对应版本的 API 变化和弃用项。

## 使用方式

当用户要求创建或修改 Electron 应用时：

1. 识别 Electron 版本、Node.js 版本、操作系统和打包工具。
2. 明确需求属于 Main、Preload、Renderer、Utility Process 还是构建发布流程。
3. 先设计权限边界，再实现功能；Renderer 默认视为不可信环境。
4. 优先使用官方 API 和安全默认值，不为了绕过错误而关闭安全选项。
5. 对 IPC、文件路径、URL、协议、外部进程和更新包做输入校验。
6. 在开发环境和打包后的生产环境分别验证路径、协议、资源和权限。
7. 说明所依据的官方文档章节，并对版本敏感的配置标注版本范围。

## 官方文档导航

- 入门教程：https://www.electronjs.org/zh/docs/latest/tutorial/tutorial-prerequisites
- 应用架构：https://www.electronjs.org/zh/docs/latest/tutorial/process-model
- 进程间通信：https://www.electronjs.org/zh/docs/latest/tutorial/ipc
- Context Isolation：https://www.electronjs.org/zh/docs/latest/tutorial/context-isolation
- 进程沙盒化：https://www.electronjs.org/zh/docs/latest/tutorial/sandbox
- 安全清单：https://www.electronjs.org/zh/docs/latest/tutorial/security
- 应用打包：https://www.electronjs.org/zh/docs/latest/tutorial/application-distribution
- 自动更新：https://www.electronjs.org/zh/docs/latest/tutorial/updates
- API 文档：https://www.electronjs.org/zh/docs/latest/api/app
- Electron Releases：https://releases.electronjs.org/

## 应用架构

### 进程模型

- **Main Process**：每个应用只有一个，负责生命周期、窗口、菜单、托盘、原生系统 API 和权限较高的操作。
- **Renderer Process**：每个 `WebContents` 通常对应一个，负责 UI；不要赋予其不必要的 Node.js 或 Electron 权限。
- **Preload Script**：在 Renderer 页面加载前运行，用于提供经过限制的桥接 API。
- **Utility Process**：适用于与 UI 无关、需要隔离或计算密集型的任务；不要把所有工作都堆到 Main Process。

推荐结构：

```text
src/
├── main/
│   ├── index.ts
│   ├── windows.ts
│   ├── ipc/
│   │   ├── index.ts
│   │   └── handlers/
│   └── security/
├── preload/
│   ├── index.ts
│   └── types.ts
└── renderer/
    ├── index.html
    └── src/
```

主进程负责创建窗口并等待 `app.whenReady()`；不要在 `app` ready 前调用依赖 Chromium 或原生资源的 API。处理 `window-all-closed`、`activate` 和平台差异时，遵循官方生命周期示例。

## 窗口和 WebContents

创建窗口时使用安全默认值：

```ts
const window = new BrowserWindow({
  webPreferences: {
    preload: path.join(__dirname, '../preload/index.js'),
    contextIsolation: true,
    nodeIntegration: false,
    sandbox: true
  }
})
```

要求：

- `contextIsolation: true` 必须保持启用。
- `nodeIntegration: false` 必须保持禁用。
- 优先启用 `sandbox: true`；如确有兼容性原因，必须记录原因、影响和替代方案。
- 生产环境使用受信任的本地入口或自定义协议，不要无约束地加载远程页面。
- 使用 `webContents` 的 `will-navigate` 和 `setWindowOpenHandler` 控制导航和新窗口。
- 不要把 `webContents`、`BrowserWindow` 或 Electron 事件对象暴露给 Renderer。
- 不要用 `webSecurity: false`、`allowRunningInsecureContent: true` 等选项掩盖资源或跨域问题。

## IPC 和 Preload

### 设计规则

**允许：**

- 使用 `contextBridge.exposeInMainWorld` 暴露最小化、目的单一、类型明确的 API。
- 使用 `ipcRenderer.invoke` 与 `ipcMain.handle` 完成异步请求/响应。
- 在 Preload 中包装回调，只向 Renderer 传递业务数据，不传递 `event`。
- 在 Main 的集中式 IPC 注册层注册处理器，避免业务模块散落 `ipcMain.handle`。
- 在每个处理器中验证 `event.sender` 或 `event.senderFrame` 的来源。
- 在 Main 中对所有来自 Renderer 的参数重新做类型、范围和权限验证。

**禁止：**

- 将完整的 `ipcRenderer` 暴露给 Renderer。
- 暴露通用的 `send`、`on`、`once` 或任意 channel API。
- 在 Renderer 中直接 `import { ipcRenderer } from 'electron'`。
- 将 `event`、`sender`、`webContents` 等对象传给 Renderer。
- 使用 `sendSync` 作为常规通信方式。
- 信任未经校验的文件路径、命令参数、URL 或序列化数据。

示例：

```ts
// preload/index.ts
import { contextBridge, ipcRenderer } from 'electron'

contextBridge.exposeInMainWorld('electronAPI', {
  getAppVersion: () => ipcRenderer.invoke('app:get-version'),
  chooseFile: () => ipcRenderer.invoke('dialog:choose-file')
})
```

```ts
// main/ipc/index.ts
import { app, ipcMain } from 'electron'

export function registerIpcHandlers() {
  ipcMain.handle('app:get-version', (event) => {
    assertTrustedFrame(event.senderFrame)
    return app.getVersion()
  })
}

function assertTrustedFrame(frame: Electron.WebFrameMain | undefined) {
  if (!frame || !isTrustedAppUrl(frame.url)) {
    throw new Error('Untrusted IPC sender')
  }
}

function isTrustedAppUrl(rawUrl: string) {
  const url = new URL(rawUrl)
  return url.protocol === 'app:' && url.hostname === 'local'
}
```

实际项目中应为 `window.electronAPI` 编写 TypeScript 全局类型声明。IPC channel 名称应采用功能域命名，例如 `settings:load`、`files:read`，而不是暴露任意调用能力。

## 数据、文件和外部链接

- IPC 只传递可序列化的数据：字符串、数字、布尔值、`null`、数组和普通对象。
- 不要发送函数、Promise、Symbol、WeakMap、WeakSet 或未经设计的 DOM 对象。
- 文件操作必须限制根目录，使用 `path.resolve` 后检查结果是否仍位于允许目录内，防止路径穿越。
- 不允许把 Renderer 提供的字符串直接作为 shell 命令执行。
- 外部链接必须使用 `URL` 解析，并限制允许的 scheme；通常只允许 `https:`，其他 scheme 必须有明确业务理由。
- 对文件选择、导入、导出和拖放内容进行大小、类型、路径和权限校验。

安全外链示例：

```ts
const allowedSchemes = new Set(['https:', 'mailto:'])
const url = new URL(rawUrl)
if (!allowedSchemes.has(url.protocol)) {
  throw new Error('URL scheme is not allowed')
}
await shell.openExternal(url.toString())
```

## 安全基线

遵循官方安全清单：

- 保持 `contextIsolation`、沙盒和禁用 Node 集成。
- 使用严格 Content Security Policy，避免不必要的 `unsafe-eval` 和 `unsafe-inline`。
- 限制导航、弹窗、协议和外部内容来源。
- 不加载不受信任的远程内容；必须加载时，单独隔离并重新评估权限。
- 不使用过时或危险的远程模块模式。
- 不禁用 Chromium web security。
- 不在日志、更新配置或仓库中写入 token、私钥和签名凭据。
- 依赖升级后重新检查 Electron 安全公告、弃用 API 和打包工具兼容���。

## 菜单、托盘、快捷键和系统能力

系统能力必须留在 Main Process，通过 Preload 暴露单一用途接口：

- 菜单和上下文菜单使用 `Menu`、`MenuItem`。
- 托盘使用 `Tray`，注意 Windows、macOS 和 Linux 的生命周期差异。
- 全局快捷键使用 `globalShortcut`，应用退出时释放注册项。
- 通知使用 `Notification`，不要把用户输入未经处理地拼入系统命令。
- 文件对话框使用 `dialog`，返回给 Renderer 的是经过筛选的路径或业务数据。
- 使用 `shell` 时严格限制 `openExternal`、`openPath` 等高权限操作。

## 存储和数据库

根据数据类型选择存储位置：

- 用户配置、日志和本地数据使用 `app.getPath('userData')` 下的应用目录。
- 不要把可变数据写入安装目录或 ASAR 包内。
- 敏感凭据优先使用操作系统凭据存储，而不是明文 JSON 或 localStorage。
- SQLite、原生模块和文件数据库应在 Main 或 Utility Process 中访问，通过窄 IPC API 提供能力。
- 对数据库迁移、锁、并发访问、备份和损坏恢复进行设计。

## 性能和稳定性

- 避免在 Main Process 执行长时间同步任务。
- CPU 密集型任务使用 Worker、Utility Process 或异步 API。
- 避免频繁创建窗口和无界监听器，窗口销毁时移除监听器。
- 对 IPC、文件、网络和更新操作设置超时、取消和错误处理。
- 通过 DevTools、Chromium 性能工具和主进程日志定位内存、CPU、启动和渲染问题。
- 生产日志应分级、可脱敏，并避免记录用户隐私和凭据。

## 打包、分发和代码签名

Electron 本身不规定唯一打包工具，可根据项目选择 Electron Forge、electron-builder 或其他成熟方案。无论工具如何选择：

- 固定 Electron、Node.js 和打包工具版本。
- 启用 ASAR；原生模块只有在有明确理由时才配置解包。
- macOS 使用 Developer ID 签名、公证和 Hardened Runtime。
- Windows 使用 Authenticode 签名，并正确配置发布者名称和证书轮换。
- Linux 使用发行版、Snap、Flatpak 或项目选择的签名发布渠道。
- CI 中通过密钥管理服务保护签名私钥和发布凭据。
- 在真实平台验证安装、启动、卸载、升级、权限、协议注册和崩溃恢复。
- 使用 Electron Fuses 禁用不需要的危险能力，例如 `runAsNode`；配置前确认所用 Electron 和打包工具版本支持情况。

## 自动更新

- macOS、Windows 可使用 Electron `autoUpdater`；也可以使用 `electron-updater` 获得更完整的发布控制。
- Linux 不依赖 Electron 内置 `autoUpdater`，使用发行版包管理器、Snap 或 Flatpak 的更新机制。
- 更新源必须使用 HTTPS，并对安装包、元数据和签名进行验证。
- Windows 保持更新代码签名验证，配置与签名证书匹配的发布者名称。
- 监听 `checking-for-update`、`update-available`、`update-not-available`、`download-progress`、`update-downloaded` 和 `error`。
- `update-downloaded` 后提供立即安装和稍后安装；处理长期不重启、强制更新年龄、回滚和降级策略。
- Squirrel.Windows 首次运行阶段存在 `--squirrel-firstrun` 时，不要立即检查更新。
- 发布前核对当前 `electron-builder` / `electron-updater` 版本的元数据格式、Web Installer、安全校验和 Linux 包验证选项，不要照搬旧版本配置。

## 测试策略

至少覆盖：

- Main 生命周期和窗口创建。
- Preload API 类型和暴露面。
- IPC 成功、失败、超时、来源伪造和参数校验。
- 文件路径穿越、危险 URL scheme 和恶意输入。
- Renderer 组件和用户流程。
- 打包后资源路径、ASAR、原生模块和协议。
- Windows、macOS、Linux 的安装、升级、卸载和签名。
- 自动更新失败、断网、下载中断、版本回滚和用户延迟安装。

## 代码审查清单

### 架构

- [ ] Main、Preload、Renderer 的职责和权限边界清晰。
- [ ] 长任务没有阻塞 Main 或 Renderer。
- [ ] 生产资源路径不依赖开发服务器假设。

### IPC

- [ ] Renderer 没有直接导入 `ipcRenderer`。
- [ ] Preload 只暴露最小化、类型化 API。
- [ ] 所有 handler 验证来源和输入。
- [ ] 没有把 Electron event 或通用 IPC 能力传给 Renderer。
- [ ] IPC 数据可序列化且有错误处理。

### 安全

- [ ] `contextIsolation: true`。
- [ ] `nodeIntegration: false`。
- [ ] `sandbox: true` 或记录了明确例外。
- [ ] 导航、新窗口、外部 URL 和协议受到限制。
- [ ] CSP 严格，没有不必要的 `unsafe-*`。
- [ ] 文件路径、命令参数、更新包和凭据均有校验或保护。

### 发布

- [ ] ASAR、代码签名和公证按目标平台配置。
- [ ] 更新源使用 HTTPS，更新包和元数据可验证。
- [ ] 已测试安装、升级、卸载、回滚和断网场景。
- [ ] CI 凭据没有进入仓库和构建日志。

## 排错顺序

1. 确认 Electron、Node.js、打包工具和目标平台版本。
2. 判断问题位于 Main、Preload、Renderer、Utility Process 还是安装/更新阶段。
3. 检查打包后的 Preload 路径、资源路径和 ASAR 内容。
4. 检查 IPC channel、注册时机、参数类型和发送方 URL。
5. 检查开发服务器 URL 与生产自定义协议的差异。
6. 检查导航、CSP、沙盒、原生模块和平台权限。
7. 更新问题检查 HTTPS、清单、签名、证书、发布者名称和安装格式。
8. 对照当前 Electron 官方中文文档和版本变更记录确认 API 行为。
9. 不通过关闭安全配置来“修复”问题；修复真实的来源、权限、路径或兼容性问题。

## 版本说明

Electron API、默认安全配置、自动更新元数据和 electron-builder 选项会随版本变化。实施前必须核对：

- 项目 `package.json` 中的 Electron 版本。
- 官方文档中的对应 API 和弃用说明。
- 目标平台的签名、公证和分发要求。
- `electron-builder`、`electron-updater`、Electron Forge 等工具的当前版本文档。

本文档不是 Electron API 的替代品；遇到 API 细节、行为变化或安全决策时，以官方文档和官方安全建议为准。
