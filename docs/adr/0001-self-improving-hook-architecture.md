# 0001 — self-improving hook 架构（2026-09-28 实施）

> wayfinder:adr · 起始 2026-09-28
> 触发：主人「为 self-improving 这个 skill 安装 hook」 + grill Round 1-3
> 来源：本会话 grill + 调研（`research/workbuddy-hooks-architecture.md`，636 行）
> 替代：无（hook 是新建设计，无前置 ADR）

## Context

self-improving skill（`~/.workbuddy/skills/self-improvement/`，v3.0.24，skillhub_pskoett）已有 OpenClaw / Claude Code 集成示例（`SKILL.md` L493-547），但**WorkBuddy 原生集成缺失**：
- `~/.workbuddy/settings.json:1-23` 当前仅注册 2 个 PreToolUse hook（config-guard + auto_backup）
- self-improving 推荐事件（`UserPromptSubmit` + `PostToolUse (Bash)`）**全部未注册**
- 自带 bash 脚本 `activator.sh` / `error-detector.sh` 无 WorkBuddy 调度入口

白泽自建 self-improving runtime（`~/.workbuddy/self-improving/`，2026-09-23 落地）已注释明文「心跳机制走 `automation_update`」但**未激活**（`last_heartbeat_started_at: never`）。

主人原 5 项 capability 需求（自动触发 / 自我改进 / 自动升级 / 可配置可观测 / 安全回滚），与 self-improving 现状对比后真正需要补的是 4 项：
- cap 1（自动触发）：**事件 → 脚本入口桥接**
- cap 3（自动升级）：**检测 + smoke test**
- cap 4（可配置可观测）：**config + 日志**
- cap 5（安全回滚）：**编排升级前备份 + 失败回滚**

cap 2（自我改进）已被 self-improving skill 核心职责覆盖，**hook 不重复**。

## Decision

### 决策 1 — Hook placement = `~/.workbuddy/hooks/`

落点位：`~/.workbuddy/hooks/self_improving_bridge.py`（适配器脚本）+ `~/.workbuddy/settings.json` `hooks` 块注册。

理由：与 WorkBuddy 现有 7 个 hooks（config-guard / guard_config / guard_secrets / auto_backup / grant_once / review_feedback）同层管理；复用 settings.json hook 调度机制；避开 skillhub 重装覆盖风险。

### 决策 2 — Implementation language = Python

理由：匹配 WorkBuddy 现有 hooks 全 Python；与 binary runtime 一致；self-improving 自带 bash 脚本可作为子进程被 Python 调用，保留上游版本。

### 决策 3 — Hook vs Skill 关系 = 桥接层（bridge）

hook = 薄适配器层；事件 → bash 脚本入口（activator.sh / error-detector.sh）转译；**不持业务逻辑**（升级 / 自改进 / 回滚决策权归 skill 与 runtime）。

理由：以 self-improving 为 master；避免双写规则的平行扩展层；hook 是「事件驱动激活层」与 runtime「定时激活层」互补。

### 决策 4 — 订阅事件 = UserPromptSubmit + PostToolUse (Bash)

**仅 2 个事件**。严格匹配 `SKILL.md` L493-547 推荐事件清单。

排除：
- PreToolUse —— config-guard + auto_backup 已覆盖 Write|Edit 守卫
- SessionStart —— 不做升级检测（升级检测走 `automation_update`）
- 其他 23+ 事件 —— 按 MEMORY 第一性原理第③问子项「全订阅触发率 100% × 误拦成本大概率负」排除

### 决策 5 — Self-improving 反馈循环改 hook 参数

hook 自我改参的颗粒度：**仅 trigger frequency（事件触发 debounce / 节流间隔）**。

理由：MVP 起步；按 MEMORY 第③问子项「拿不出频率证据 → 待验证」原则，先单参数；触发率证据齐再考虑扩 matcher filter + learning promotion threshold。

### 决策 6 — 升级目标 = self-improving skill 本体 + 自带 scripts + 白泽 runtime

升级整个 ecosystem：
- `~/.workbuddy/skills/self-improvement/` （SKILL.md + scripts/）
- `~/.workbuddy/self-improving/`（runtime 配置）

理由：ecosystem 一致性；不能只换 skill 让 runtime 引用错版本。

### 决策 7 — 升级检测 source = 本地 hash + 远端 WebFetch

本地 `_skillhub_meta.json` 的 `installedContentHash` 对比 + 远端 `clawhub.ai/pskoett/skills/self-improving-agent` WebFetch 拉 manifest。

理由：纯本地检测不到新版；纯远端不安全（无对照基线）；双源最稳。

### 决策 8 — Upgrade cadence = weekly via `automation_update` + SessionStart hook 初始化

升级检测走 `automation_update` weekly cron（白泽已批准能力范围）；SessionStart 时仅做 hook lifecycle 初始化（写日志 + 读配置），**不做远端拉取**。

理由：避免重复消耗（远端拉取放 cron 足够）；SessionStart 频繁触发不能用作升级检测时机。

### 决策 9 — Rollback target = `skills.backup/`（仿 auto_backup.py 协议）

升级前备份到 `~/.workbuddy/skills.backup/<skill>/<date>/`；3-份留底 + manifest。

理由：复用 auto_backup.py 已落地协议（hook 可调用同一备份 helper）；manifest 记录 sha256 可验证。

### 决策 10 — Rollback trigger = fail-safe（任何异常都回滚）

升级失败 / health-check 失败 / 任何异常都回滚到最近备份。

理由：与 auto_backup.py 的 fail-open **相反**——升级场景下 fail-safe 更安全（升级失败导致 skill 不可用 = 阻塞工作）；按 SOUL 硬 Gate 4「审慎」与主人分层原则「可逆操作优先」叠加。

### 决策 11 — Hook-runtime 契约 = 文件系统共享态

三层文件体系 + 文件系统共享态通信：
- hook 层（`hooks/self_improving_bridge.py`）→ 接收 WorkBuddy 事件 → 调 bash 子进程
- bash 脚本层（`skills/self-improvement/scripts/{activator,error-detector}.sh`）→ 写 `.learnings/`
- runtime 层（`self-improving/`）→ 读 `.learnings/` + `heartbeat-state.md`，靠 `automation_update` 定时

排除：
- Python import runtime：当前 runtime 全 .md，无 Python 入口（违反 SSOT）
- bash CLI 桥：runtime 当前是 markdown 模板集合，非 executable

### 决策 12 — Configuration format = Python 模块（`config.py`）

`~/.workbuddy/hooks/self_improving_config.py` 暴露 `CONFIG = {...}` 字典。

理由：与 hook 同语言，类型检查免费；JSON/YAML/TOML 都需解析层；Python 配置唯一缺点是不能运行时热改——但 hook 配置通常启动加载，热改需求低。

### 决策 13 — Activator / error-detector 复用 vs 重写 = subprocess 调 bash

hook 通过 `subprocess.run` 调 `activator.sh` / `error-detector.sh`；保留上游实现。

理由：以 self-improving 为 master 前提要求保留官方脚本；自动升级时 bash 脚本跟着新版本走，hook 不需感知。

### 决策 14 — `.learnings/` first-use init 责任 = Skill 保留

skill SKILL.md L18-23 明文：首次使用前 init；**责任留在 skill 本体**；hook 与 runtime 不重复。

理由：SSOT 原则；init 是 skill 第一次使用逻辑。

### 决策 15 — Upgrade smoke test = frontmatter 解析 + bash 语法检查

升级后验证：
- SKILL.md frontmatter 解析（`name` / `version` / `description` 必有）
- `bash -n` 语法检查（不执行）

理由：frontmatter 漏字段 = 严重错误；`bash -n` 零副作用静态检查捕获 90% 升级翻车；不跑真实 activator 是避免写 `.learnings/` 副作用污染。

### 决策 16 — Idempotency / 去重 = 信任 skill 的 LRN-ID 机制

hook 不做去重；activator.sh 设计本就是 lightweight 提醒（hooks-setup.md L196-202：~50-100 tokens）；self-improving 的 LRN-ID 格式 `LRN-YYYYMMDD-XXX` 天然去重。

理由：trust skill layer 的 idempotency 是「以 self-improving 为准」的应有之义；hook 加去重是重复 skill 的责任。

## Consequences

### 正向

- 5 项 capability 中 4 项有明确落地（cap 1 事件桥 / cap 3 检测 / cap 4 配置 / cap 5 回滚）；cap 2 由 skill 自带
- hook 是「薄适配器」——代码量预计比独立子系统方案少 30-50%
- 全 ecosystem 升级一致（skill + scripts + runtime 同步）
- 三层文件体系 + 共享态通信——零新机制，纯组合现有能力
- 与 WhiteZe 既有体系（config-guard / auto_backup / grant_once / review_feedback）模式一致

### 反向 / 副作用

- hook 强耦合 self-improving skill 存在；**未来替换 self-improving skill 时 hook 须同步重写**——这是「以 self-improving 为准」的 trade-off
- PreToolUse 不订阅 = 失去 PreToolUse 时机的 self-improving 触发（如配置改动前提醒）；如需补回，加 Round 4 grill
- 升级失败回滚 = 可能阻塞 skill 工作（fail-safe 副作用）；按 SOUL 硬 Gate 4「审慎」叠加选择

### 适用范围

- **本工作空间**：D:/WorkBuddy/2026-09-28-10-47-08/ —— 当前会话工作区
- **跨工作空间**：可能（其他用 self-improving 的工作空间可复用同一 hook）
- **非适用范围**：白泽全局配置（SOUL / MEMORY）——本 ADR 不动

## Alternatives considered

- **方案 A：Hook 为独立子系统**（含完整升级 / 自改进 / 回滚能力，self-improving 只是目标之一）—— 拒绝。违反「以 self-improving 为准」前提；cap 2 由 skill 自带，hook 重复是反 SSOT。
- **方案 B：Hook 仅 PreToolUse**（写文件时触发 activator）—— 拒绝。PreToolUse 已被 config-guard / auto_backup 占用；self-improving 推荐事件是 UserPromptSubmit + PostToolUse，强行改 PreToolUse 偏离 skill 设计意图。
- **方案 C：SessionStart 做升级检测** —— 拒绝。SessionStart 频率高（每次会话），远端拉取消耗大且主人提示疲劳；weekly `automation_update` 足够。
- **方案 D：Hook 重写 Python 版 activator** —— 拒绝。违反「以 self-improving 为准」；双维护成本高；上游升级时 Python 版要同步改。
- **方案 E：Hook 通过 Python import runtime** —— 拒绝。runtime 当前全 .md 无 Python 入口；强加会破坏 SSOT 与分层。

## Status

- ✅ 2026-09-28 设计完成
- ✅ 2026-09-28 实施完成（smoke test 全过）：
  - `~/.workbuddy/hooks/self_improving_config.py` —— 路径常量 + 配置字典
  - `~/.workbuddy/hooks/self_improving_bridge.py` —— 事件分支 + bash subprocess + debounce + 日志
  - `~/.workbuddy/settings.json` `hooks` 块 —— 新增 3 个 hook 入口（UserPromptSubmit + PostToolUse + SessionStart）
  - `.config-routed-settings.json.flag` —— 刷新（mtime 14:42，24h 有效）
  - `~/.workbuddy/self-improving/automation/upgrade-check.json.template` —— weekly cron 模板（status=PAUSED，不启用）
  - `~/.workbuddy/self-improving/automation/README.md` —— 启用流程 + 触发率证据检查清单
- smoke test 覆盖：6 个用例（4 事件 + invalid + empty），全部 exit 0，log 写入正常
- 观察期起算：2026-09-28 → 2026-10-05（1 周）—— 触发率证据积累后再决定启用 weekly cron
- 关联文档：`CONTEXT.md`（术语表）/ `research/workbuddy-hooks-architecture.md`（调研，636 行）
- 关联 skill：self-improving skill + self-improving runtime + WorkBuddy hooks 调度
- 关联决策：基于 grill Round 1-3 主人批准