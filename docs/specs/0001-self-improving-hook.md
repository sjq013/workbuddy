# Spec 0001 — self-improving hook 集成（WorkBuddy 事件桥）

> wayfinder:spec · 起始 2026-09-28
> 状态：✅ 已实施 + TDD 验证（T1-T10 全 PASS）
> 关联文档：[`CONTEXT.md`](../../CONTEXT.md) 术语表 / [`../adr/0001-self-improving-hook-architecture.md`](../adr/0001-self-improving-hook-architecture.md) 架构决策 / [`.learnings/LEARNINGS.md`](../../.learnings/LEARNINGS.md) LRN-20260928-001 / 20260928-MEMORY.md 决策域「主动 vs 克制」分层原则
> 发布状态：draft（issue tracker 未 setup，主人配 `/setup-matt-pocock-skills` 后可发）

---

## Problem Statement

白泽（self-improving skill 的使用者）在 WorkBuddy 上没有自改进事件桥——自改进 skill 自带 bash 脚本（`activator.sh` / `error-detector.sh`）专为 Claude Code / Codex 设计，WorkBuddy 之前只注册了 2 个 `PreToolUse` hook（config-guard + auto_backup），自改进相关事件（`UserPromptSubmit` / `PostToolUse (Bash)`）全部未启用。

白泽自建的 self-improving runtime（`~/.workbuddy/self-improving/`，2026-09-23 落地）已注释明文「心跳机制走 `automation_update`」但**未激活**（`last_heartbeat_started_at: never`）。

后果：白泽只能在对话中手动「想起要 log」才能写 `.learnings/`——闭环**事件驱动部分完全缺失**。

## Solution

按主人分层原则（ADR 0001 + MEMORY 决策域 2026-09-28 落档的「主动 vs 克制：价值观层 vs 行为准则层」）构建**薄事件桥 hook**：

- hook = adapter pattern，不持业务逻辑
- 3 类 WorkBuddy 事件 → 调 self-improving 上游逻辑（Python 重实现，mirror-of 上游 bash 脚本语义；Windows 不适用 subprocess 调 bash）
- 与 proactive-agent 共享 `.learnings/` 目录作为协同通道
- HEARTBEAT auto-cleanup / Autonomous Crons 按主人分层原则 **显式禁用**（行为层与白泽硬 Gate 冲突）
- 升级检测走「weekly `automation_update` + 本地 hash + 远端 clawhub.ai WebFetch」，当前 status=PAUSED 等待触发率证据（per MEMORY 第一性原理第③问强制子项）

## User Stories

1. As 白泽（主人），I want 自改进 reminder 在每次我发消息后注入白泽上下文（UserPromptSubmit → activator.sh），so that 白泽每次会话结束前评估是否要写 .learnings/LEARNINGS.md。
2. As 白泽，I want 错误检测 reminder 在 Bash 工具调用失败后注入上下文（PostToolUse (Bash) → error-detector 逻辑），so that 非显然错误自动被白泽看到。**平台限制**：WorkBuddy 的 PostToolUse `tool_response` 不含 stdout/stderr 文本且 exitCode 恒 0 → 该提醒在本平台为 no-op；有效自动路径为 `UserPromptSubmit` activator 提醒 + 白泽自觉落档（per LRN-20260928-002）。逻辑保留以兼容 Claude Code 等上游环境。
3. As 白泽，I want SessionStart 时 hook 写一条 lifecycle log，so that 会话边界有可见记录（不替代功能）。
4. As 白泽，I want hook config 是 Python 模块而非 JSON / YAML，so that 类型检查免费 + 与 hook 同语言。
5. As 白泽，I want activator / error-detector 语义与上游 bash 脚本逐字对齐（mirror-of 锚点），so that upstream 升级时同步注释锚点即可，无需重写逻辑。**实现修正**：Windows `bash`→wsl.exe 被沙箱拦截 + PostToolUse 不传工具输出 → 改为 Python 重实现（非 subprocess 复用）。
6. As 白泽，I want hook 配置里 60s debounce 抑制重复触发，so that log 不被刷屏。
7. As 白泽，I want 事件触发场景 hook 失败时 fail-open（exit 0），so that hook 异常不阻塞白泽；升级场景反过来 fail-safe 失败回滚，so that skill 不可用 = 阻塞工作。
8. As 白泽，I want self-improving 与 proactive-agent 通过 `.learnings/` 共享状态，so that 错题本写的 → 小雷达读 → 下次提前防住（数据驱动接力）。
10. As 白泽，I want HEARTBEAT auto-cleanup / Autonomous Crons 显式禁用（按分层原则），so that 不可逆 / 后台执行被白泽硬 Gate 拦住。
11. As 白泽，I want 升级检测 cron 有 status=PAUSED 模板（不实际启动），so that 触发率证据齐前不抢跑——按 MEMORY 第③问强制子项 + adr/0025 反面案例 guard_bulk_read 教训。
12. As 白泽，I want self-improving LRN 条目用 Pattern-Key + Recurrence-Count 字段，so that promote 阈值（Recurrence-Count ≥ 3 + 跨 ≥ 2 任务 + 30 天）触发自动升级。
13. As 白泽，I want hook 测试用 vertical slices 跑通 6+ seams（不是 horizontal slicing），so that 每个 seam 独立验证 + 失败定位明确。
14. As 白泽，I want 新增 MEMORY 规则走 META 四查 + 主人审批 + flag 落档，so that 4 件套硬约束（≤20 条）不被破坏。
15. As 白泽，I want 错推三步走（认错 → 立即修正 + 写 daily log → 暴露新规则时主人批后入 MEMORY）作为自我批评流程，so that 推荐类失误有强制纠错路径。
16. As 白泽，I want LRN Pattern-Key 跨多次会话可被未来白泽 grep 触发提示，so that「推荐前不 Read 源」一类盲区会自我修复（per 闭环思维）。

## Testing Decisions

#### 决策 1：测试方法学 = tdd「red → green」vertical slices

- 一个测试 → 一个最小验证 → 重复
- 每个 seam 独立断言，不写 implementation 路径 mock
- 期望值必须独立来源（spec / 已知字面量），不与代码共构（tautological）

#### 决策 2：测试面（seams）

- **Hook 文件可执行 + Python 语法**（T1）
- **settings.json 注册 3 个事件**（T2）
- **Hook 接收 UserPromptSubmit**（T3）
- **Hook 接收 PostToolUse (Bash)**（T4）
- **Hook 接收 SessionStart**（T5）
- **Manual 直接调 bridge Python（activator 路径）**（T6，替代原 activator.sh）
- **Manual 喂失败 tool_response 给 bridge（error-detect 路径）**（T7）
- **Manual 喂干净 tool_response 给 bridge → 不输出**（T7b，反面断言）
- 注：原 T6/T7「直接调 bash 脚本」已废弃（Windows bash→wsl 被沙箱拦截，改 Python 重实现，见 LRN-20260928-002）
- **Skill 工具调用 self-improvement**（T8）
- **Manual 直接 edit .learnings/LEARNINGS.md**（T9）
- **闭环验证：LRN-20260928-001 在 LEARNINGS.md**（T10）

#### 决策 3：测试断言原则

- **独立期望**：log 文件新增数 +1（不是 "first line contains X" 这种 fragile 断言）
- **二元断言**：PASS / FAIL 显式输出
- **反面断言**：T7b（干净输入不应输出 reminder）防 false positive

#### 决策 4：测试 harness = Bash + Python inline

- 不引 pytest / unittest（避免重型依赖）
- 每次断言明确 inline（不是隐藏的测试框架）
- 测试可重复——每次跑前清 `.last_trigger` debounce

#### 决策 5：测试范围

- ✅ **测试了**：hook 文件 / 注册 / 事件分发 / debounce / Python 提醒注入（activator + error-detect）/ Skill 加载 / 文件落档
- ❌ **没测试**：automation_update weekly cron（status=PAUSED，未启）
- ❌ **没测试**：clawhub.com.au 远端拉取（升级流程未启）
- ❌ **没测试**：workbuddy 主进程实际调用 hook（session 内事件触发由 IDE 负责）

## Implementation Decisions

#### 决策 1：Hook placement = `~/.workbuddy/hooks/`

理由：与现有 7 个 hooks 同层管理；复用 settings.json hook 调度机制；避开 skillhub 重装覆盖。

#### 决策 2：Implementation language = Python

理由：匹配 WorkBuddy 现有 hooks 全 Python；与 binary runtime 一致。

#### 决策 3：Hook vs Skill 关系 = 桥接层（bridge）

理由：以 self-improving 为 master；hook 不持业务逻辑，事件转译到 skill 官方脚本语义（Python 重实现，mirror-of 上游 activator.sh / error-detector.sh）。

#### 决策 4：订阅事件 = UserPromptSubmit + PostToolUse (Bash) + SessionStart

理由：UserPromptSubmit / PostToolUse (Bash) 是 self-improving SKILL.md 推荐事件；SessionStart 用于 hook lifecycle init（不替代功能）。

#### 决策 5：Self-improving 反馈循环改 hook 参数 = trigger frequency

理由：MVP 起步；按 MEMORY 第③问子项等触发率证据再扩 matcher + promotion threshold。

#### 决策 6：升级目标 = self-improving skill 本体 + 自带 scripts + 白泽 runtime

理由：ecosystem 一致性；不能只换 skill 让 runtime 引用错版本。

#### 决策 7：升级检测 source = 本地 hash + 远端 WebFetch

理由：双源最稳；本地 hash 来自 `_skillhub_meta.json` 的 `installedContentHash`，远端从 clawhub.ai 拉 manifest。

#### 决策 8：Upgrade cadence = weekly via `automation_update` + SessionStart hook 初始化

理由：升级检测走 cron，SessionStart 仅做 lifecycle init 写 log。

#### 决策 9：Rollback target = `~/.workbuddy/skills.backup/`

理由：复用 auto_backup.py 已落地协议（3-份 留底 + manifest）。

#### 决策 10：Rollback trigger = fail-safe

理由：与 auto_backup.py 的 fail-open 相反——升级场景下 fail-safe 更安全。

#### 决策 11：Hook-runtime 契约 = 文件系统共享态

理由：三层文件体系 + 共享 `.learnings/` + `heartbeat-state.md` 解耦通信；无 IPC。

#### 决策 12：Configuration = Python 模块（`config.py`）

理由：与 hook 同语言；类型检查免费。

#### 决策 13：Activator / error-detector = Python 重实现（mirror-of 上游 bash 脚本）

理由：以 self-improving 为 master → 语义须对齐上游；但 Windows `bash`→wsl.exe 被沙箱拦截 + PostToolUse 不传工具输出，subprocess 调 bash 不可行（详见 ADR 0001 决策 13 + LRN-20260928-002）。error-detector 逻辑保留不删（兼容 Claude Code 等上游环境）。

#### 决策 14：`.learnings/` first-use init = Skill 保留

理由：SSOT；责任留在 skill 本体。

#### 决策 15：Upgrade smoke test = frontmatter 解析 + bash -n

理由：frontmatter 漏字段 = 严重错误；`bash -n` 零副作用静态检查捕获 90% 升级翻车。

#### 决策 16：Idempotency = 信任 skill 的 LRN-ID

理由：trust skill layer 的 idempotency 是「以 self-improving 为准」应有之义；hook 加去重是重复 skill 的责任。

#### 决策 17：分层原则（governs） = 主动 = 价值观层 / 克制 = 行为准则层

理由：主人的范式跃迁——消解同层冲突（Q3 旧框架失败案例）。

#### 决策 18：proactive-agent 启用/禁用清单

- 启用：WAL Protocol / Working Buffer / 反向 prompt / 6 Pillars / Draft-don't-send / ADL+VFM Protocol
- 禁用：HEARTBEAT auto-cleanup / Autonomous Crons

理由：行为层与白泽硬 Gate 冲突。

## Out of Scope

- **Weekly upgrade cron 启动**（status=PAUSED 等触发率证据）
- **跨事件 debounce → per-event debounce 改造**（触发率证据不足）
- **Owner 校验**（依赖 clawhub.ai 公钥机制研究）
- **Concurrent runs 协调**（hook ↔ automation_update 同时触发）
- **Self-improving 自改 matcher + threshold**（MVP 仅 debounce）
- **Backup helper 独立抽象**（升级模板内联实现）
- **CI 集成**（无 CI 基础设施）
- **MEMORY 新规则直接入**（依赖 self-improving promote 机制自然累积）
- **AGENTS.md 新增段**（同上）

## Further Notes

- **关联 ADR**：0001（hook 架构 16 决策）+ 0047（分层原则）
- **关联 runtime 资产**：`~/.workbuddy/self-improving/automation/upgrade-check.json.template`（status=PAUSED）+ README.md（启用流程）
- **观察期**：2026-09-28 14:48 → 2026-10-05 14:48（1 周）
- **触发率数据点**：截至本 spec 落档，hook UserPromptSubmit 实测 5+ 次，PostToolUse (Bash) 实测 2 次，SessionStart 实测 1 次
- **新规则候选**：白泽给推荐前必须 Read 直接相关源文件（已被主人采纳 B 方案：依赖 self-improving promote 机制，不入 MEMORY）
- **session 闭环**：TDD T1-T10 全 PASS + LRN-20260928-001 已落档 + 观察期启动 + 主人选 B 方案
- **未来 promote 触发**：Recurrence-Count ≥ 3 + 跨 ≥ 2 任务 + 30 天（per self-improving SKILL.md L358-372）