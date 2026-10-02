# dsh-plugin-session-delete-uni

**通用会话删除插件，带强制归档门槛** —— 为 [DeepSeek Harness](https://github.com/topics/dsh-plugin) (DSH) 设计。

> 先归档，才能永久删除。
>
> Archive first. Only then can you delete.

---

## 为什么需要它

DSH 原生支持**归档**会话：把不用的会话收起来，眼不见为净，但数据还在、随时可取回。这是一个安全的缓冲层。

问题是：**归档之后，会话依然永远躺在磁盘上**。DSH 没有提供删除已归档会话的办法。日积月累，`~/.dsh/sessions/` 会堆满再也不会打开的会话；导入会话（`oc-*`、`qoder-*`）尤其如此。

社区已有几个删除插件，但它们都是**确认框 + 直接删**——把"能不能删"完全交给一次点击，误操作没有任何拦截。

本插件补上这一环，并把**归档作为删除的前置条件**：

```
未归档会话  →  拒绝删除（UI 禁用 + Host 返回 409）
已归档会话  →  风险确认后，彻底清除
```

这个设计的价值在于：**归档是一个可逆的、低成本的整理动作；删除是不可逆的。** 强制你先归档，等于强制你多走一步、多想一次，而这一步本身不会丢失任何数据。

---

## 功能

### 删除范围（彻底清除，零残留）

一次删除会清理 **全部四类痕迹**：

| 目标 | 说明 |
|---|---|
| 会话日志 | `~/.dsh/sessions/<workspace>/<sessionId>/` 及其内容 |
| 投影缓存 | session projection cache 中的对应行 |
| 工作区归属 | 会话在其工作区中的登记 |
| 归档记账 | `workspace.global.archivedSessionIds` 中的条目 |

最后一项容易被忽略：删了会话却留下归档条目，就会产生永远无法清理的孤儿引用。本插件在确认日志已彻底删除**之后**才移除记账，并把"文件删不干净"视为失败（不会留下半删状态的会话掉进 "Ungrouped"）。

### 通用 ID 支持

接受所有 DSH 会话 ID 形态：

- `session-<uuid>`（原生）
- `<uuid>`（裸 UUID）
- `oc-*` / `qoder-*` / `oc-merge-*`（导入会话）

ID 仅允许安全字符、最长 200 字符，**拒绝任何路径分隔符**——防止拼接路径逃逸出会话目录。

### 运行中会话

删除正在运行的会话时，先取消任务、等待静默（quiescence）、flush 并 detach，然后才删磁盘日志。

### 三个入口

| 入口 | 位置 |
|---|---|
| 垃圾桶按钮 | 会话头部，标题旁 |
| 菜单项 | 侧栏会话行「…」菜单（**不切换当前会话**） |
| Agent 工具 | `workbench_session_delete` |

---

## 安装

```bash
dsh plugin --profile desktop add dsh-plugin-session-delete-uni
```

或从本地路径安装：

```bash
dsh plugin --profile desktop add file:/path/to/session-delete-uni
```

> **⚠️ 安装后必须完全退出并重启 DSH**（不是关窗口）。
>
> 客户端半边是 `window.__ModuleLoader__.load()` 动态模块，只随页面 `__DSH_BOOT__` 图重新扫描加载。`F5` / `Ctrl+R` 会被 Electron 壳拦掉，无效。
>
> 不重启时：Host 端工具照常可用，但 UI 按钮停在 `active: false` —— 这个不对称很容易被误判成"装失败"。

### 验证是否加载

用 `cordis_inspect_query`（client 平台 / `Slots` provider / `listSubTree`），`root` 传具体 slot 名：

```
conversation.session.header.actions
```

看 `selected.occupants[]` 里是否有 `{ id: "session-delete", active: true }`。

> 注意：不指定 `root` 只列声明、不列占用者，会误判成"没装"。

---

## 使用

1. 在 DSH 侧栏**归档**目标会话（原生功能）。
2. 点会话头部垃圾桶，或侧栏行「…」菜单里的删除项。
3. 确认对话框会显示**会话名称 + ID**（防止点错），运行中的会话附带警告。
4. 确认后彻底删除。

未归档的会话：对话框会提示先归档，**确认按钮禁用**；即使绕过 UI 直接调 Host 接口，也会返回 HTTP 409。

---

## 设计取舍

**为什么不做"归档并删除"一键通道？**

因为那会让归档退化成一次点击的过场，"缓冲"的意义就没了。归档和删除应当是**两个独立、有先后、各自可停下的动作**。这是刻意的摩擦。

**为什么删除前先查归档状态而不信任客户端？**

UI 可以被绕过。归档门槛在 Host 的删除 Core 里再查一次（`workspace.global.archivedSessionIds`），这是权威判定。

---

## 兼容性

| 项目 | 要求 |
|---|---|
| DSH | `0.2.0-rc.2` 实测通过 |
| `@deepseek-ai/dsh-tools` | `>=0.1.0-rc.6 <0.3.0` |
| Node.js | `>=20` |

> **关于 peerDependencies 范围**：本插件只使用 `defineTool` 一个 API，其 `name` / `description` / `parameters` / `output` / `execute` 字段在 `0.1.0-rc.6` 与 `0.2.0-rc.2` 中均存在，因此范围放宽到 `<0.3.0` 是经过核对的，而非盲目放宽。若你用的是 0.3.x，请先确认 `defineTool` 签名未变。

---

## 来源与许可

**MIT License.**

本项目基于 [`lsz-asd/dsh-plugin-session-delete`](https://github.com/lsz-asd/dsh-plugin-session-delete) v0.3.1（MIT）修改而来。原项目的删除能力（日志 / 投影缓存 / 工作区记账清理、运行中会话处理）是其成果；本版本的主要增量是：

1. **强制归档门槛** —— 未归档一律拒绝（UI + Host 双重检查）
2. **通用 ID 支持** —— 从仅 UUID 放宽到任意安全字符 ID，覆盖 `oc-*` / `qoder-*` 导入会话
3. **归档记账清理** —— 删除时同步移除 `archivedSessionIds` 条目，避免孤儿引用
4. **peerDependencies 范围修正** —— 兼容 0.2.x 运行时

原始版权归原作者所有，详见 [LICENSE](./LICENSE)。

### 与同类插件对比

| 插件 | 归档门槛 | 说明 |
|---|---|---|
| `dsh-plugin-session-delete` | ❌ | 确认框后直接删 |
| `lsz-asd/dsh-plugin-session-delete` | ❌ | 确认框后直接删 |
| **本插件** | ✅ | **未归档一律拒绝** |

---

## English

**Universal session delete for DeepSeek Harness, with a mandatory archive gate.**

DSH lets you *archive* sessions — a safe, reversible way to clear your list without losing data. But archived sessions stay on disk forever, and DSH offers no way to remove them. Existing community delete plugins go straight from a confirmation dialog to permanent deletion, with nothing standing between a stray click and lost work.

This plugin makes **archiving a precondition for deletion**:

- **Not archived** → refused (confirmation disabled in the UI; HTTP 409 from the host)
- **Archived** → permanently removed after an explicit risk confirmation

The point of the friction: archiving is reversible, deletion is not. Forcing the archive step first makes you take one deliberate, lossless action before an irreversible one.

**Complete removal** clears all four traces: the on-disk session log, the projection cache row, workspace accounting, and the archive-bookkeeping entry in `workspace.global.archivedSessionIds` — the last of which is easy to miss and leaves permanently un-cleanable orphan references if skipped.

**Universal ID support**: `session-<uuid>`, bare UUID, and imported ids (`oc-*`, `qoder-*`, `oc-merge-*`). Path separators are rejected outright to prevent directory traversal.

**Three entry points**: header trash button, sidebar session-row menu (without switching the active conversation), and the `workbench_session_delete` agent tool.

```bash
dsh plugin --profile desktop add dsh-plugin-session-delete-uni
```

> **Fully quit and restart DSH after installing** — the client half is a dynamic `window.__ModuleLoader__.load()` module loaded from the page's `__DSH_BOOT__` graph. `F5` / `Ctrl+R` are swallowed by the Electron shell. Until you restart, the host tools work but the UI button sits at `active: false`.

Based on [`lsz-asd/dsh-plugin-session-delete`](https://github.com/lsz-asd/dsh-plugin-session-delete) v0.3.1 (MIT). MIT licensed.

---

## License

[MIT](./LICENSE)
