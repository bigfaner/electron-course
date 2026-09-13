---
name: terminology
description: 项目文档统一术语表——进程/世界、两道边界、IPC、函数代理等核心概念的规范叫法与禁用叫法
domains: [electron, ipc, preload, contextbridge, docs]
---

# 统一术语表

> 目的：`docs/ARCHITECTURE.md`、`docs/ELECTRON-GUIDE.md` 及后续所有交流使用同一套词汇，避免"同一个概念三个名字"。
> 规则：正文首次出现用**规范名（英文）**；图/泳道标签用本表"标准标签"列；"禁用"列的叫法不要再出现。

## 1. 进程与世界（层级不能错，最重要）

| 规范术语 | 英文 | 定义 | 标准标签 | 禁用/易错叫法 |
|---|---|---|---|---|
| **主进程** | Main Process | 独立 OS 进程，Node.js 环境，全局唯一 | `主进程（Node）` | ❌ 单独说"Node 进程"；✅ 可写"主进程（Node.js 环境）" |
| **渲染进程** | Renderer Process | 每个 `BrowserWindow` 一个独立 OS 进程 | `渲染进程（OS 进程）` | ❌ 与"页面"混称 |
| **preload 世界** | preload world | 渲染进程**内部**的隔离 V8 Context，运行 preload 脚本 | `preload 世界` | ❌ "preload 进程"（它不是进程！）；❌ 不带前缀的"隔离世界" |
| **主世界** | main world | 渲染进程内部页面 JS 所在的 V8 Context，React 代码运行于此 | `主世界（React）`/`主世界（页面）` | ❌ "页面进程" |

关键层级（一句话）：**主世界与 preload 世界同属一个渲染进程；主进程才是另一个 OS 进程。**

## 2. 两道边界

| 规范术语 | 定义 | 俗称（仅辅助修辞） | 禁用 |
|---|---|---|---|
| **边界 A（contextBridge 边界）** | 进程内、跨 V8 世界的边界；只有 C++ 能跨 | "C++ 海关" | ❌ 把它说成"跨进程" |
| **边界 B（IPC 边界）** | 跨 OS 进程的边界；经 Mojo 管道传序列化消息 | — | — |

## 3. IPC 通信

| 规范术语 | 定义 | 禁用/易错叫法 |
|---|---|---|
| **频道（channel）** | IPC 消息的逻辑名字符串，如 `'statistics'` | ❌ "通道"（该词留给 pipe） |
| **管道（Mojo message pipe）** | 底层传输通道，与窗口一对一；Windows 上的 OS 层实现叫**命名管道** | ❌ 与"频道"混用 |
| **载荷（payload）** | 消息携带的数据 | — |
| **三种通信形态** | ① 请求-响应（invoke/handle）② 单向消息（send/on）③ 推送（webContents.send → ipcRenderer.on） | 方向记法 R→M / M→R（R=渲染进程侧，M=主进程） |
| **监听表** | ipcRenderer / ipcMain 内部按频道登记回调的注册表 | ❌ "事件表" |
| **序列化** | structured clone（V8 ValueSerializer），函数不可序列化（`DataCloneError`） | — |

## 4. 函数代理（contextBridge 机制）

| 规范术语 | 定义 |
|---|---|
| **函数代理**（机制名，function proxy） | contextBridge 为跨世界调用生成替身函数的机制 |
| **代理壳**（实例名） | 具体的替身函数。本仓库共三种：**壳① 暴露壳**（暴露时包装，如 `window.electron.subscribeStatistics`）、**壳② 反向壳**（实参过桥时包装，如 `cbProxy`）、**壳③ 返回值壳**（返回值过桥时包装，如主世界拿到的 `unsub`） |
| **真身** | 被代理的原函数 |
| **wrapper** | `ipcOn` 内剥离 event 的本地闭包（**非代理**） |
| **cb** | UI 传给 `subscribeStatistics` 的回调真身 |

铁律：**函数代理只产于边界 A（contextBridge）；边界 B（IPC）是纯数据通道，0 个代理。**

## 5. 构建视角 vs 运行视角（两份文档的对应）

| 构建视角（ARCHITECTURE.md） | 运行视角（ELECTRON-GUIDE.md） | 源码 |
|---|---|---|
| 主进程侧 | 主进程 | `src/electron/*.ts`（不含 preload.cts） |
| 渲染层 / UI 层 | 主世界（页面） | `src/ui/**` + `index.html` |
| preload | preload 世界 | `src/electron/preload.cts` |

## 6. 泳道标准标签（时序图 / 流程图）

- `主世界（React）` 或 `主世界（页面）`
- `contextBridge（C++）`——可括注"海关"
- `preload 世界`
- `主进程（Node）`——图首注明"独立 OS 进程"

## 7. 一页记忆图

```mermaid
flowchart TB
    subgraph RP["渲染进程（OS 进程）"]
        subgraph ISO["V8 Isolate（同堆同 GC）"]
            MW["主世界（React/页面）"]
            PW["preload 世界"]
        end
        CB["contextBridge（C++）＝边界 A，进程内跨世界"]
    end
    RP <-.->|"IPC（Mojo 管道）＝边界 B，跨进程"| MP["主进程（Node，独立 OS 进程）"]
```
