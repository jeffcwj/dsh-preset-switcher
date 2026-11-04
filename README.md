# dsh-preset-switcher

> [English](#english) | **中文**

---

<img width="656" height="350" alt="image" src="https://github.com/user-attachments/assets/c9ecdb1a-446e-48a8-825d-a3e495ff773b" />


## 概述

DSH Web 插件：在**任何时候**切换当前会话的代理预设（模式），包括对话已经开始之后 — 原生 UI 不允许这样做（"已开始的对话的代理预设已固定"）。

### 核心功能

- **顶部下拉菜单**：会话标题操作栏中的「当前模式 ▾」按钮，列出所有预设（如 `standard`=标准模式、`code`=PTC 模式、`minimal`=极简模式、`cordis`=创造模式），用 ✓ 标记当前预设，点击即切换。
- **失效预设修复**：如果会话记录的预设已被删除（自定义预设被移除），恢复将失败导致会话不可用。下拉菜单中的「修复失效模式会话」操作（及 `repair` API）会扫描所有会话日志，向引用失效的会话**追加**一条指向默认预设的 `agent-preset/selected` 事件（只追加、不改写；恢复时最后一个事件生效），并跳过活跃会话。
- **未打开的会话也能切（自动载入）**：会话若只是「本进程还没载入」，插件先走宿主会话控制器的 `resolveAgent()`（`/api/session/*` 用的同一条 resolve-or-resume 路径）把它载入，再就地重组；只有载入失败（归档、不可读、写所有权在别的实例）时才退回写句柄追加延迟事件，下次打开时生效。
- **受控路由**：`POST /preset-switch/api`，参数 `{ method: 'list' | 'switch' | 'repair', sessionId, presetId, target? }`（connection 信任围栏，与 dsh-sounds 相同）。

### 工作原理

原生 API（`agentPresets.select`）通过空白会话检查来保护切换操作。本插件底层调用同一原子原语 — `agentPresets.recompose(agent.ctx, presetId)` — 将代理的作用域重新链接到目标预设的常驻挂载点。新组合在链接移动前确保完成，因此损坏的预设不会影响会话。成功后插件追加与代理记录相同的 `agent-preset/selected` 会话事件，使投影和标题标签同步更新。

未加载的会话没有可重组的 agent，插件改用官方 `sessionPersistence` 句柄 API（`open(id,'write')` → `append` → `flush`）把选择记录进日志；恢复会话时按最后一个 `agent-preset/selected` 事件组合，所以重新打开即生效。读操作始终用 `open(id,'read')`，不会抢占活跃会话的写所有权。

**生效时机**：切换是就地重组，**不需要重启 DSH**；新的工具集/人设从**下一个模型步骤**开始生效（已经发出的那一步仍用旧工具集）。子代理拥有的会话会被拒绝（403）。

> 实测（DSH 0.2.2-alpha.2）：`minimal → standard` 之后下一个请求的 `request/header.tools` 多出 29 个工具（`read`/`write`/`edit`/`glob`/`grep`/`todo_write`/`web_search`…），`standard → minimal` 再减掉同一批 29 个，双向即时生效、无需重启。若在 0.2.1-rc.2 上遇到「切档后工具集不变」，请升级到 0.2.2-alpha.2 复测（见 issue #2）。

## 兼容性

面向 **DSH 0.2.x**（已在 0.2.2-alpha.2 上适配）：

- 宿主端用 `connection` 服务的 `requestRejection()` 做信任围栏（0.1.x 的 `webRuntime` 已不存在）；
- 会话读写用 `sessionPersistence.open()`/`stat()`/`list()`（旧的 `inspect()`/`append()` 已移除）；
- 未载入会话的切换走 `sessionController.resolveAgent()`（宿主会话控制器的 resolve-or-resume 入口）；
- 客户端 bundle 未变，继续使用 `conversation.session.header.actions` 插槽。

## 安装 / 更新

```powershell
# 直接从 GitHub 安装（tag 名与 DSH 版本对应）
dsh plugin --profile 0.2.2-alpha.2 add "git+https://github.com/jeffcwj/dsh-preset-switcher.git#dsh-0.2.2-alpha.2"

# 或者：克隆后从本地目录安装（开发用）
git clone https://github.com/jeffcwj/dsh-preset-switcher.git
dsh plugin --profile 0.2.2-alpha.2 add .\dsh-preset-switcher
```

装完重启 dsh web（profile 启动时才会合并 `cordis.patch.yml`）。

> `--profile` 换成你自己的 profile 名（对应 `~/.dsh/profiles/<name>`），安装路径换成你自己的克隆位置。

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
- **Unopened sessions too (auto-load)**: a session that is merely not loaded in this process is first resolved through the host Session controller (`resolveAgent()`, the same resolve-or-resume entry behind `/api/session/*`) and recomposed live. Only when that fails (archived, unreadable, writer held by another instance) does the plugin append a deferred selection event instead, applied on the next open.
- **Fenced route**: `POST /preset-switch/api` with `{ method: 'list' | 'switch' | 'repair', sessionId, presetId, target? }` (Connection trust fence, same as dsh-sounds).

### How it works

The stock API (`agentPresets.select`) guards its swap with a blank-session check. This plugin calls the same atomic primitive underneath — `agentPresets.recompose(agent.ctx, presetId)` — which re-links the agent's scope to the target preset's standing mount. The new composition is ensured before the link moves, so a broken preset leaves the session untouched. On success the plugin appends the same `agent-preset/selected` session event the proxy records, so projections and the header label follow.

A not-loaded session has no live agent to recompose, so the plugin records the choice with the official `sessionPersistence` handle API (`open(id,'write')` → `append` → `flush`). Resume composes from the LAST `agent-preset/selected` event, so reopening the session applies the new mode. Reads always use `open(id,'read')`, which never claims the single-writer slot of a live session.

### When it applies

The swap is an in-place recompose — no DSH restart. The new tool set/persona applies from the **next model step**; a step already in flight keeps the old one. Sessions owned by a subagent are refused (403).

> Measured on DSH 0.2.2-alpha.2: `minimal → standard` added 29 tools to the next `request/header.tools` (`read`/`write`/`edit`/`glob`/`grep`/`todo_write`/`web_search`…), `standard → minimal` removed the same 29, both immediately, no restart. If tools did not change for you on 0.2.1-rc.2, re-test on 0.2.2-alpha.2 (issue #2).

### Compatibility

Targets **DSH 0.2.x** (adapted on 0.2.2-alpha.2):

- host fence via the `connection` service's `requestRejection()` (the 0.1.x `webRuntime` service is gone);
- session access via `sessionPersistence.open()`/`stat()`/`list()` (the old `inspect()`/`append()` are gone);
- switching a not-yet-loaded session goes through `sessionController.resolveAgent()` (the host's resolve-or-resume entry);
- the client bundle is unchanged and still uses the `conversation.session.header.actions` slot.

## Install / update

```powershell
# install straight from GitHub (the tag name mirrors the DSH version)
dsh plugin --profile 0.2.2-alpha.2 add "git+https://github.com/jeffcwj/dsh-preset-switcher.git#dsh-0.2.2-alpha.2"

# or: clone and install the local directory (development)
git clone https://github.com/jeffcwj/dsh-preset-switcher.git
dsh plugin --profile 0.2.2-alpha.2 add ./dsh-preset-switcher
```

Restart dsh web afterwards (the profile merges `cordis.patch.yml` at boot).

> Replace `--profile 0.2.2-alpha.2` with your own profile name and the path with your clone location.

## Layout

| File | Description |
|------|-------------|
| `cordis.patch.yml` | Bundle patch: inserts the plugin row into the profile's cordis tree (`dsh.bundle.patch`). |
| `lib/index.js` | Host half: the fenced `/preset-switch/api` routes (list / switch / repair) and the session-log repair logic. |
| `lib/client.js` | Browser half: the header dropdown (ClientModuleSystem bundle, React via module-table externals). |
| `scripts/` | `restart-web.ps1` (kill + restart + probe), `inspect-session.mjs` / `repair-session.mjs` (standalone log tools). |

## License

MIT
