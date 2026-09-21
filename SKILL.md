---
name: electron-development
description: Electron 应用开发指导，重点覆盖安全 IPC、进程隔离、导航控制、软件更新、代码签名、打包发布与测试。用于设计、实现、审查和排查 Electron 项目。
---

# Electron 开发指导 Skill

## 目标

本 Skill 用于指导 Electron 应用的设计、开发、安全审查、更新机制、打包发布和故障排查。优先选择安全默认值，避免为了快速实现功能而扩大渲染进程权限。

## 开发原则

1. 明确区分 Main、Preload 和 Renderer 三类代码。
2. Renderer 默认视为不可信环境，不能直接访问 Node.js 或 Electron 高权限 API。
3. 只通过类型化、最小权限的 Preload API 跨进程通信。
4. 所有来自 Renderer 的数据都必须在 Main 进程重新校验。
5. 生产环境必须考虑导航控制、CSP、代码签名、更新完整性和回滚策略。
6. 不因为测试方便而关闭 `contextIsolation`、`sandbox` 或安全校验。

## 推荐架构

```text
src/
├── main/
│   ├── index.ts          # 应用生命周期和窗口创建
│   ├── ipc/              # 集中式 IPC 注册层
│   └── security/         # URL、来源和输入校验
├── preload/
│   ├── index.ts          # 最小化 contextBridge API
│   └── types.ts           # Window API 类型声明
└── renderer/
    └── ...                # UI 和业务展示逻辑
```

## 安全

### 1. IPC 通信机制

**允许：**

- 使用 `contextBridge.exposeInMainWorld` 暴露最小化、目的单一的 API 方法。
- 通过 `ipcRenderer.invoke` 进行异步请求/响应通信。
- 在 Preload 中包装回调，避免将 `event` 参数暴露给 Renderer。
- 在 `ipcMain.handle` / `ipcMain.on` 中验证 `event.sender` 或 `event.senderFrame` 的来源。
- 使用集中式 IPC 注册层，禁止在业务代码中散落 `ipcMain.handle` 调用。
- 所有 Renderer 代码通过类型化的 `window.electronAPI` 访问 IPC。

**禁止：**

- 将完整的 `ipcRenderer` 对象或通用的 `send` / `on` 方法暴露给 Renderer。
- 直接将 `event` 参数传递给 Renderer 回调。
- 暴露无限制的 `shell.openExternal`，而不校验 scheme 和目标地址。
- 在 IPC 处理器中信任未经校验的文件路径、URL 或其他输入。
- 在 `src/**` 的 Renderer 代码中直接 `import { ipcRenderer } from 'electron'`。

### 2. 推荐 IPC 模式

```ts
// preload/index.ts
import { contextBridge, ipcRenderer } from 'electron'

contextBridge.exposeInMainWorld('electronAPI', {
  getAppVersion: () => ipcRenderer.invoke('app:get-version'),
  openExternal: (url: string) => ipcRenderer.invoke('shell:open-external', url)
})
```

```ts
// main/ipc/index.ts
import { app, ipcMain, shell } from 'electron'

const ALLOWED_EXTERNAL_SCHEMES = new Set(['https:', 'mailto:'])

export function registerIpcHandlers() {
  ipcMain.handle('app:get-version', (event) => {
    assertTrustedFrame(event.senderFrame)
    return app.getVersion()
  })

  ipcMain.handle('shell:open-external', async (event, rawUrl: unknown) => {
    assertTrustedFrame(event.senderFrame)
    if (typeof rawUrl !== 'string') throw new TypeError('Invalid URL')

    const url = new URL(rawUrl)
    if (!ALLOWED_EXTERNAL_SCHEMES.has(url.protocol)) {
      throw new Error('External URL scheme is not allowed')
    }

    await shell.openExternal(url.toString())
  })
}

function assertTrustedFrame(frame: Electron.WebFrameMain | undefined) {
  if (!frame || !frame.url.startsWith('app://')) {
    throw new Error('Untrusted IPC sender')
  }
}
```

实际项目中应将 `app://` 替换为应用真实的受信任来源，并在应用启动时只调用一次 `registerIpcHandlers()`。

### 3. WebPreferences

创建 `BrowserWindow` 时保持以下配置：

```ts
webPreferences: {
  preload: path.join(__dirname, '../preload/index.js'),
  contextIsolation: true,
  nodeIntegration: false,
  sandbox: true
}
```

**禁止：**

- 开启 `nodeIntegration: true`。
- 关闭 `contextIsolation`。
- 生产环境加载远程内容却没有严格 CSP 和导航限制。

对 `will-navigate`、`setWindowOpenHandler` 和必要时的 `webContents` 新窗口事件进行拦截，只允许受信任的应用来源。外部链接必须经过 scheme、域名和目标地址校验。

### 4. IPC 数据序列化

**允许：** JSON 可序列化的对象、数组、字符串、数字、布尔值和 `null`。

**禁止：**

- DOM 对象，例如 `ImageBitmap`、`File`、`DOMMatrix`。
- 函数、Promise、Symbol、WeakMap、WeakSet。
- 将 `sendSync` 作为常规通信方式，因为它会阻塞 Renderer。

### 5. 内容安全策略

生产环境设置严格 CSP，至少限制脚本来源，不使用不必要的 `unsafe-eval`。不要把 CSP 当作唯一防线，仍需保持上下文隔离、来源校验和输入校验。

## 软件更新机制

### 1. 平台策略

- macOS 和 Windows 使用 Electron `autoUpdater` 或 `electron-updater`。
- Linux 使用发行版包管理器、Snap 或 Flatpak；不要依赖 Electron 内置 `autoUpdater`。
- 使用 `electron-builder` 时，优先采用其签名、发布和更新生态。

### 2. 更新源和签名

**允许：**

- 使用 HTTPS 分发更新。
- 对更新包进行代码签名，并在安装前验证签名。
- Windows 保持 `win.verifyUpdateCodeSignature: true`，并正确配置 `publisherName`。
- 对更新清单（如 `latest.yml`）做独立签名，防止降级攻击。
- 使用 Ed25519 或同等强度的数字签名保护发布元数据。
- macOS 更新前完成代码签名和公证。

**禁止：**

- 通过 HTTP 分发更新而不做签名验证。
- 只验证二进制签名而忽略更新清单完整性。
- 在生产环境发布未签名应用。
- 禁用 ASAR 完整性校验。

### 3. 更新生命周期

监听并记录以下事件：

- `checking-for-update`
- `update-available`
- `update-not-available`
- `download-progress`
- `update-downloaded`
- `error`

下载完成后向用户提供“立即重启安装”和“稍后安装”选项，并处理长期不重启的情况：

- 支持空闲安装，例如用户 15 分钟无操作后安装。
- 设置最大更新年龄，例如超过 30 天后强制提示或重启。
- 支持经过密码学验证的降级和回滚。
- 在 Squirrel.Windows 的 `--squirrel-firstrun` 阶段不要立即检查更新。

### 4. electron-builder 更新配置

保持安全配置，不使用未经验证的 Web Installer 载荷：

```yaml
build:
  asar: true
  asarUnpack: []
  electronUpdaterCompatibility: '>=2.16'
  nsis:
    oneClick: false
    perMachine: false
    # 按当前 electron-builder 版本确认 disableWebInstaller 的配置位置和默认值
  win:
    verifyUpdateCodeSignature: true
  linux:
    # 发布渠道必须提供可验证的签名包
    target:
      - AppImage
```

对于 electron-builder v27/v28 及以上版本，发布前必须核对当前版本的 `disableWebInstaller`、`allowUnverifiedLinuxPackages` 和更新元数据要求；不要复制旧版本配置而忽略版本兼容性。若配置项在当前版本已默认安全，也不要为了绕过错误而显式关闭安全校验。

## 代码签名与完整性

- macOS：配置 Developer ID、签名、Hardened Runtime 和公证。
- Windows：配置 Authenticode 证书、`publisherName`，并验证证书轮换策略。
- 打包时启用 ASAR；除非有明确且审查过的原生模块需求，不要随意解包资源。
- 使用 Electron Fuses 禁用不需要的危险能力，例如 `runAsNode`。
- CI 中保护签名私钥和发布凭据，禁止写入仓库或构建日志。
- 在发布前验证安装包、更新包、清单和签名，而不仅是构建是否成功。

## 实施工作流

1. 确认 Electron、Node.js、electron-builder/electron-updater 的版本。
2. 画出 Main、Preload、Renderer 的权限边界。
3. 先配置安全的 `webPreferences` 和导航拦截。
4. 为每个功能设计单一用途、类型化的 IPC API。
5. 在集中式 IPC 层注册处理器，并验证发送方和参数。
6. 为文件路径、URL、命令参数设置允许范围，拒绝路径穿越和危险 scheme。
7. 配置 CSP、ASAR、代码签名和更新源。
8. 测试开发环境、打包环境、安装、升级、回滚和卸载流程。
9. 在 Windows、macOS、Linux 上验证平台差异。
10. 审查日志中是否泄露令牌、路径、签名私钥或用户隐私。

## 审查清单

### IPC

- [ ] Renderer 没有直接导入 `ipcRenderer`。
- [ ] Preload 只暴露最小 API，没有通用 `send` / `on`。
- [ ] 每个 handler 都验证发送方来源。
- [ ] 每个外部输入都做类型、范围和权限校验。
- [ ] 没有把 Electron `event` 对象传给 Renderer。

### 窗口和导航

- [ ] `contextIsolation: true`。
- [ ] `nodeIntegration: false`。
- [ ] `sandbox: true`，除非有记录充分的兼容性理由。
- [ ] 已拦截未知导航和新窗口。
- [ ] 外部 URL 使用 HTTPS 或明确允许的 scheme。
- [ ] 生产环境 CSP 严格且没有不必要的 `unsafe-*`。

### 更新和发布

- [ ] 更新通过 HTTPS 获取。
- [ ] 更新包和清单都经过完整性验证。
- [ ] Windows 验证 `publisherName` 和证书。
- [ ] macOS 已签名并公证。
- [ ] Linux 使用发行版支持的签名包或更新机制。
- [ ] 处理 `update-downloaded` 后长期不重启。
- [ ] 已验证回滚、降级和版本年龄策略。
- [ ] 未禁用 ASAR 完整性校验或更新安全开关。

## 排错顺序

1. 确认问题发生在 Main、Preload 还是 Renderer。
2. 检查打包后 Preload 路径和实际文件是否存在。
3. 检查 IPC channel 名称、参数类型和注册时机。
4. 检查 `event.senderFrame.url` 是否为预期来源。
5. 检查开发环境 URL 与生产环境自定义协议是否不同。
6. 检查签名证书、发布清单、HTTPS 和更新渠道。
7. 检查系统平台、安装格式和 Electron/electron-builder 版本差异。
8. 不要通过关闭安全配置来“修复”问题；应定位真正的来源、权限或兼容性原因。
