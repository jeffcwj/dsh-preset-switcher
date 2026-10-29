# dsh-preset-switcher

> [English](#english) | **中文**

---

## 概述

DSH Web 插件：在**任何时候**切换当前会话的代理预设（模式），包括对话已经开始之后 — 原生 UI 不允许这样做（"已开始的对话的代理预设已固定"）。

### 核心功能

- **顶部下拉菜单**：会话标题操作栏中的「当前模式 ▾」按钮，列出所有预设（如 `standard`=标准模式、`code`=PTC 模式、`minimal`=极简模式、`cordis`=创造模式），用 ✓ 标记当前预设，点击即切换。
- **失效预设修复**：如果会话记录的预设已被删除（自定义预设被移除），恢复将失败导致会话不可用。下拉菜单中的「修复失效模式会话」操作（及 `repair` API）会扫描所有会话日志，向引用失效的会话**追加**一条指向默认预设的 `agent-preset/selected` 事件（只追加、不改写；恢复时最后一个事件生效），并跳过活跃会话。
- **未打开会话也能切**：没有活跃 agent 的会话会通过写句柄把选择追加为延迟事件，重新打开该会话时自动按新预设组合。
- **受控路由**：`POST /preset-switch/api`，参数 `{ method: 'list' | 'switch' | 'repair', sessionId, presetId, target? }`（connection 信任围栏，与 dsh-sounds 相同）。

### 工作原理

原生 API（`agentPresets.select`）通过空白会话检查来保护切换操作。本插件底层调用同一原子原语 — `agentPresets.recompose(agent.ctx, presetId)` — 将代理的作用域重新链接到目标预设的常驻挂载点。新组合在链接移动前确保完成，因此损坏的预设不会影响会话。成功后插件追加与代理记录相同的 `agent-preset/selected` 会话事件，使投影和标题标签同步更新。

未加载的会话没有可重组的 agent，插件改用官方 `sessionPersistence` 句柄 API（`open(id,'write')` → `append` → `flush`）把选择记录进日志；恢复会话时按最后一个 `agent-preset/selected` 事件组合，所以重新打开即生效。读操作始终用 `open(id,'read')`，不会抢占活跃会话的写所有权。

**限制**：子代理拥有的会话将被拒绝；新的工具集/人设从下一个模型步骤生效。

## 兼容性

面向 **DSH 0.2.x**（已在 0.2.2-alpha.2 上适配）：

- 宿主端用 `connection` 服务的 `requestRejection()` 做信任围栏（0.1.x 的 `webRuntime` 已不存在）；
- 会话读写用 `sessionPersistence.open()`/`stat()`/`list()`（旧的 `inspect()`/`append()` 已移除）；
- 客户端 bundle 未变，继续使用 `conversation.session.header.actions` 插槽。

## 安装 / 更新

```powershell
dsh plugin --profile 0.2.2-alpha.2 add C:\Users\Lenovo\dsh-plugins\dsh-preset-switcher
# 重启 dsh web（profile 启动时合并 cordis.patch.yml）：
powershell -NoProfile -ExecutionPolicy Bypass -File scripts\restart-web.ps1
```

## 文件结构

| 文件 | 说明 |
|------|------|
| `cordis.patch.yml` | Bundle 补丁：将插件行插入 profile 的 cordis 树（`dsh.bundle.patch`） |
| `lib/index.js` | 服务端：受控 `/preset-switch/api` 路由（list / switch / repair）及会话日志修复逻辑 |
| `lib/client.js` | 浏览器端：顶部下拉菜单（ClientModuleSystem bundle，React 通过 module-table externals） |
| `scripts/` | `restart-web.ps1`（终止+重启+探测）、`inspect-session.mjs` / `repair-session.mjs`（独立日志工具） |

## 许可证

MIT

---

<a id="english"></a>

# dsh-preset-switcher

> **English** | [中文](#概述)

---

## Overview

DSH web plugin: switch the current session's agent preset (mode) at **any time**, including after the conversation has already started — the stock UI refuses that ("a started conversation's agent preset is fixed").

### Features

- **Header dropdown**: a "当前模式 ▾" button in the session header action row lists every preset (e.g. `standard`=标准模式, `code`=PTC 模式, `minimal`=极简模式, `cordis`=创造模式), marks the session's current one with ✓, and switches on click.
- **Stale-preset repair**: if a session's recorded preset was deleted (a custom preset removed), resume fails and the session becomes unusable. The dropdown's "修复失效模式会话" action (and `repair` API) scans every session log and APPENDS a corrective `agent-preset/selected` event pointing at the default preset (append-only; the last event wins on resume), skipping live sessions.
- **Unopened sessions too**: a session with no live agent gets the choice appended as a deferred event through a write handle, so reopening it composes from the new preset.
- **Fenced route**: `POST /preset-switch/api` with `{ method: 'list' | 'switch' | 'repair', sessionId, presetId, target? }` (Connection trust fence, same as dsh-sounds).

### How it works

The stock API (`agentPresets.select`) guards its swap with a blank-session check. This plugin calls the same atomic primitive underneath — `agentPresets.recompose(agent.ctx, presetId)` — which re-links the agent's scope to the target preset's standing mount. The new composition is ensured before the link moves, so a broken preset leaves the session untouched. On success the plugin appends the same `agent-preset/selected` session event the proxy records, so projections and the header label follow.

A not-loaded session has no live agent to recompose, so the plugin records the choice with the official `sessionPersistence` handle API (`open(id,'write')` → `append` → `flush`). Resume composes from the LAST `agent-preset/selected` event, so reopening the session applies the new mode. Reads always use `open(id,'read')`, which never claims the single-writer slot of a live session.

### Compatibility

Targets **DSH 0.2.x** (adapted on 0.2.2-alpha.2):

- host fence via the `connection` service's `requestRejection()` (the 0.1.x `webRuntime` service is gone);
- session access via `sessionPersistence.open()`/`stat()`/`list()` (the old `inspect()`/`append()` are gone);
- the client bundle is unchanged and still uses the `conversation.session.header.actions` slot.

## Install / update

```powershell
dsh plugin --profile 0.2.2-alpha.2 add C:\Users\Lenovo\dsh-plugins\dsh-preset-switcher
# restart dsh web afterwards (profile boot merges cordis.patch.yml):
powershell -NoProfile -ExecutionPolicy Bypass -File scripts\restart-web.ps1
```

## Layout

| File | Description |
|------|-------------|
| `cordis.patch.yml` | Bundle patch: inserts the plugin row into the profile's cordis tree (`dsh.bundle.patch`). |
| `lib/index.js` | Host half: the fenced `/preset-switch/api` routes (list / switch / repair) and the session-log repair logic. |
| `lib/client.js` | Browser half: the header dropdown (ClientModuleSystem bundle, React via module-table externals). |
| `scripts/` | `restart-web.ps1` (kill + restart + probe), `inspect-session.mjs` / `repair-session.mjs` (standalone log tools). |

## License

MIT
