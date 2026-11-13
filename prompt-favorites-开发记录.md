# Prompt 收藏夹（DSH 插件）开发记录

> 本文是本轮对话的完整总结：需求演进、技术调研结论、最终实现、安装过程、验证记录与后续建议。
> 对应插件：`@local/prompt-favorites` v1.1.0，安装目录 `D:\DSH插件`。

- 记录时间：2026-11-13
- 运行环境：DSH Desktop `0.2.1-rc.2`（Electron 打包，实现位于 `D:\软件\deepseek\resources\app.asar\dsh`）
- 运行时依赖：Node `24.21.0`、pnpm `11.28.5`
- 当前 profile：`desktop` → `C:\Users\xxkj\.dsh\profiles\desktop`
- Web GUI：`http://127.0.0.1:<每次启动随机端口>`

---

## 1. 需求演进

| 阶段 | 需求 | 结果 |
| --- | --- | --- |
| v1.0 | 输入框上方一个「📌 常用提示词」按钮，弹层列出 3 个写死的模板，点击填入输入框并把光标放到末尾 | 已实现、已安装、用户确认基础功能跑通 |
| v1.1 | 提示词改为可持久化（刷新/重启不丢）、支持添加（标题+内容）、支持悬浮删除、保留插入逻辑、给出完整代码与重启方法 | 已实现并写回安装目录，用户确认运行成功 |

---

## 2. 技术调研与关键结论

### 2.1 DSH 插件不是浏览器扩展

用户最初的表述是「manifest.json + index.js / index.html」，DSH 里的真实对应关系是：

| 常见扩展概念 | DSH 插件中的文件 | 说明 |
| --- | --- | --- |
| `manifest.json` | `package.json` + `cordis.patch.yml` | 包名、导出表、`dsh.bundle.patch`、`dsh.client` 客户端入口、locale/icon 导出 |
| 后台脚本 | `index.js`（Host 半边） | Node 侧，可注册服务、路由、工具等 |
| 页面 / UI | `client.js`（Client 半边） | 浏览器侧，通过 `window.__ModuleLoader__.load()` 注册 React 组件到宿主插槽 |
| `index.html` | **不需要** | 插件是宿主页面内的 React 组件；iframe 页面拿不到宿主主题变量和 `ctx.locale`，属于错误做法 |

### 2.2 插槽选择

通过 `cordis_inspect_query`（Client `Slots`）确认的候选：

| 插槽 | 语义 | 结论 |
| --- | --- | --- |
| `conversation.input.dock` | **输入框卡片上方的整行区域**（已占用：todo / goal / queue） | ✅ 采用，order 5 |
| `conversation.input.left` | 输入卡片工具行左侧的紧凑控件 | 备选，但语义是“输入框内工具行”，不是“上方” |
| `conversation.composer.dock` | 输入卡片下方的环境条 | 不符合“上方” |
| `shell.overlay` | 全屏浮层 | 需要自己算坐标，没必要 |

两个插槽都只在**存在会话**时渲染（`zone !== undefined` / `sessionId !== undefined`），这与“能写草稿”的前提一致（没有会话就没有 `inputActions`）。

### 2.3 如何写入输入框

宿主对话包（`@deepseek-ai/dsh-client-ui-conversation`）给插槽条目的标准属性里有：

```
useInput: SnapshotSelectorHook<InputState>
inputActions: InputActions
```

`InputActions` 是宿主自己暴露的公开动作面（源码中 `SessionInputShell.actions`），包含
`captureInsertion / insertText / setDraft / persistDraft / addAttachments / removeAttachment / pruneAttachments / submit`。

- `setDraft(text)` 是官方“程序化写入草稿”入口：整体替换、换行拆段落、**编辑器选区落到末尾**，并触发草稿持久化与发送按钮状态更新；
- 它**不负责 DOM 聚焦**（源码注释明确：直接对 contenteditable 做裸 `focus()` 会把光标落到开头），而公开动作面里没有 `focus()`；
- 因此实现为：`setDraft()` 之后，从插件自己的节点向上找到最近的 `[contenteditable="true"]`（即已挂载的宿主编辑器），`focus()` 后把 DOM 选区折叠到文本末尾。编辑器内部选区本来就在末尾，两者一致。

编辑器的草稿状态由 Lexical 管理，插件**不直接改编辑器 DOM**，只调用公开动作。

### 2.4 持久化：为什么最终没用 localStorage

这是本次最重要的一次架构纠偏。

- 桌面版启动参数（`@deepseek-ai/dsh-desktop-host`）为：
  ```js
  args: ["--no-open", "--host", "127.0.0.1", "--port", "0"]
  ```
- `--port 0` 表示**每次启动由系统随机分配端口**；`localStorage` 按「协议 + 主机 + 端口」隔离，
  重启后端口一变，旧数据就读不到 —— 恰好违背“重启不丢”的核心要求。
- 因此改为 **Host 侧落盘 + 同源 HTTP 路由**；localStorage 仅作镜像缓存兜底。

宿主的 `webServer` 服务提供 `register({ kind: 'exact' | 'prefix', path, handler })`，
它在路由表命中时**直接调用 handler**（早于 SPA fallback，也不经过高层鉴权），因此插件可以拥有一个干净的 JSON 接口。

---

## 3. 最终架构

```
┌──────────────────────────── 浏览器（Client 半边 client.js） ────────────────────────────┐
│ conversation.input.dock 插槽                                                            │
│   └─ 「📌 常用提示词」按钮 + 浮层（列表 / 添加表单 / 删除）                              │
│         │ 打开时 fetch GET  /prompt-favorites/data                                      │
│         │ 增删时 fetch PUT  /prompt-favorites/data                                      │
│         └─ 点击条目 → inputActions.setDraft(text) → 聚焦 composer 编辑器到末尾           │
│         └─ localStorage 镜像：key = dsh-prompt-favorites:v1（路由不可达时兜底）          │
└───────────────────────────────────────┬─────────────────────────────────────────────────┘
                                        │ 同源 HTTP（127.0.0.1:<随机端口>）
┌───────────────────────────────────────┴───────── Host 半边（index.js） ─────────────────┐
│ ctx.webServer.register({ kind:'exact', path:'/prompt-favorites/data' })                 │
│   GET  → 读文件（缺失则写入 3 条初始预设）                                                │
│   PUT  → 校验 + 原子写（临时文件 → rename）                                               │
│ 文件：%DSH_HOME%\storages\prompt-favorites\prompts.json                                  │
└─────────────────────────────────────────────────────────────────────────────────────────┘
```

### 数据文件

```
C:\Users\xxkj\.dsh\storages\prompt-favorites\prompts.json
```

结构：

```json
{
  "version": 1,
  "items": [
    { "id": "uuid", "title": "代码审查", "content": "代码审查：请作为资深架构师……", "createdAt": 1730000000000 }
  ]
}
```

### 接口约定

| 方法 | 行为 | 返回 |
| --- | --- | --- |
| `GET` | 读取（首次自动播种 3 条预设并落盘） | `200 { version, items }` |
| `PUT` | 校验后整体覆盖写入 | `200 { version, items }` |
| 其他 | — | `405` |
| 跨站 Origin | 比对 `Origin` 与 `Host` 不同 | `403 { error:'forbidden-origin' }` |
| 非 JSON 请求 | `content-type` 不以 `application/json` 开头 | `415` |
| 非法 JSON / 超限 | 请求体解析失败 / 超过 1 MiB | `400` / `413` |

限额：条目 ≤ 200，标题 ≤ 120 字，内容 ≤ 8000 字；非法条目（空标题/空内容）在两端都会被过滤。

---

## 4. 交付物

```
D:\DSH插件\
├─ package.json        # 清单：bundle patch + dsh.client 入口 + locale/icon 导出（v1.1.0）
├─ cordis.patch.yml    # 插入 id=prompt-favorites 的插件行
├─ index.js            # Host：磁盘存储 + /prompt-favorites/data 路由
├─ client.js           # Client：按钮、浮层、列表/新增/删除、写入输入框并聚焦
├─ locale\en.json      # 插件列表显示用标题/描述（英文）
├─ locale\zh.json      # 同上（中文）
├─ icon.svg            # 原创图钉图标（498 B）
└─ README.md           # 安装位置、存储说明、重启测试步骤
```

工作区另有一份同源副本（早期备份）：
`C:\Users\xxkj\Documents\deepseek-harness\default-workspace\prompt-favorites\`

### 关键代码位置

| 主题 | 位置 |
| --- | --- |
| 模板种子数据 | `index.js` 的 `SEED_ITEMS`（仅首次播种，之后以文件为准） |
| 数据清洗 | `index.js` 的 `sanitizeItems()`（客户端同名函数保持一致） |
| 原子写 | `index.js` 的 `writeStore()` |
| 同源校验 | `index.js` 的 `isSameOrigin()` |
| 路由注册 | `index.js` 的 `apply()` |
| 读写封装 | `client.js` 的 `loadItems()` / `saveItems()` |
| 写入草稿 + 聚焦 | `client.js` 的 `insertPrompt()` + `focusComposerAtEnd()` |
| 新增 / 删除 | `client.js` 的 `submitForm()` / `removePrompt()` |

---

## 5. 安装过程

1. 在 workspace 编写 bundle（package.json + patch + index.js + client.js + locale + icon）。
2. 用 `plugin_manager` 的 `install_bundle` 安装目录 → pnpm 以 `link:` 方式登记。
3. profile 侧结果（`C:\Users\xxkj\.dsh\profiles\desktop`）：
   - `package.json`：`"@local/prompt-favorites": "link:D:/DSH插件"`，并加入 `dsh.profile.bundles`
   - `node_modules\@local\prompt-favorites` → 指向 `D:\DSH插件` 的 **Junction**
4. 安装返回：`application: applied`、`enabled: true`、`warnings: []` → 新装 bundle 可通过 HMR 生效，无需重启；
   但**替换已安装包的 JS**（改代码）需要重启进程，才能加载新的模块代次。

> 因为采用 `link:` 安装，修改 `D:\DSH插件` 里的文件**不需要重新安装**，只需重启 DSH。

---

## 6. 验证记录

### 6.1 v1.0 阶段

| 检查 | 方式 | 结果 |
| --- | --- | --- |
| JS 语法 | `node --check index.js client.js` | 通过 |
| JSON 合法性 | `JSON.parse` 三个文件 | 通过 |
| 图标 | `python -c` + `lxml` 解析 SVG | 通过（`viewBox="0 0 64 64"`，498 B） |
| 插槽注册 | `cordis_inspect_query` → `conversation.input.dock` | `id: prompt-favorites, order: 5, active: true` |
| 视觉 | 无浏览器控制能力 | 未验证，交由用户目视 |

### 6.2 v1.1 阶段（Host 存储）

用 Node 直接驱动 Host 处理器（stub 掉 `ctx.webServer.register`，临时 `DSH_HOME`），实测输出：

```
route: exact /prompt-favorites/data
GET 200 3 seeded items
file seeded: 代码审查|中英互译|总结大纲
PUT 200 2
GET after PUT 200 a,b
file after PUT: a,b
PUT sanitize -> 200 c
PUT foreign origin -> 403 {"error":"forbidden-origin"}
PUT text/plain -> 415 {"error":"unsupported-media-type"}
PUT same origin -> 200 {"version":1,"items":[]}
```

另有：

| 检查 | 结果 |
| --- | --- |
| `node --check` 两个文件 | 通过 |
| 动态 import `index.js` | `exports: apply,inject,name`，`inject: ['webServer']` |
| 安装后探测 `GET /prompt-favorites/data`（未重启） | `404` → 证明宿主仍在跑旧代码，必须重启 |
| 用户重启后实际使用 | 用户反馈「插件成功运行」 |

**未验证**：浮层/表单/悬浮删除的视觉与交互细节（无浏览器控制权限，且不采用截图/模拟渲染来替代真实页面）。

---

## 7. 遇到的问题与决策

| 问题 | 处理 |
| --- | --- |
| 首次 `install_bundle` 需要 danger-full-access，用户拒绝 | 停止安装，改为交付完整代码 + 手动启用说明；不绕过授权 |
| 用户要求把代码写到 `D:\DSH插件`（工作区外） | 沙箱拒绝后，用一次性提升权限重试并复制成功（每次越界写都单独说明理由） |
| `localStorage` 在随机端口下会丢数据 | 改为 Host 落盘 + 同源路由；localStorage 降级为镜像缓存 |
| 公开动作面没有 `focus()` | 用 DOM `focus()` + 折叠选区到末尾；不直接改编辑器 DOM 内容 |
| 不知道 `webServer.register` 的入参形状 | 从 `app.asar` 中提取 `dsh-host-webserver` 构建产物确认 `{ kind, path, handler }`，而不是猜测 |
| `Origin` 头在 `localhost` 与 `127.0.0.1` 下不同 | 改为「`Origin` 的 host:port 与 `Host` 头一致」的同源判定，兼容两种写法 |

---

## 8. 使用方式

### 重启测试

1. **完全退出 DSH**（关闭窗口不够，托盘/后台进程一并退出），重新打开；
2. 进入任一对话，输入框上方出现「📌 常用提示词」；
3. `➕ 添加新提示词` → 填标题与内容 → 保存；
4. 悬浮条目 → 🗑️ 删除；
5. 再次退出并重启，确认增删结果仍在；也可直接打开
   `C:\Users\xxkj\.dsh\storages\prompt-favorites\prompts.json` 核对。

### 交互细节

- 输入框已有内容时，提示词**追加**在原文之后（空一行分隔），不覆盖正在写的内容；
- 写入后自动聚焦，光标停在全文末尾；
- 浮层支持 `Esc` 关闭、点外部关闭；空间不足自动向下展开；
- 样式只使用 `--dsw-alias-*` 主题变量，跟随明暗主题。

### 已知边界

- 只在**有会话**的对话中渲染（由插槽语义决定）；
- 删除即时生效，无二次确认、无撤销；
- 若磁盘写入失败，条目会暂存在浏览器镜像缓存并显示错误提示；重启后以磁盘文件为准，这条可能丢失。

---

## 9. 经验与后续建议

**经验**

1. 选持久化方案前先确认宿主 origin 是否稳定（`--port 0` 是决定性事实）；
2. 用 `cordis_inspect_query` 和宿主构建产物确认真实 API，不要按印象写（`setDraft` 存在但没有 `focus`；`register` 需要 `kind`）；
3. 插件只调用宿主公开动作面，不碰别人的 DOM 结构（聚焦是唯一必要的 DOM 触碰，且只读不写内容）；
4. 越界写文件、安装插件都有独立授权，拒绝就停手，不做绕过。

**可继续做**

- 编辑已有提示词（当前只能删了重加）；
- 拖拽排序 / 置顶；
- 删除二次确认或「撤销」提示；
- 导入导出 `prompts.json`；
- 搜索过滤（条目多时）。
