# WorkBuddy self-improving hook 集成

白泽（Baize）+ self-improving skill 在 WorkBuddy 上的事件桥集成方案。本仓落档配套 spec 与 ADR，作为会话产出归档。

## 仓内容

| 文件 | 用途 |
|---|---|
| `docs/specs/0001-self-improving-hook.md` | 用户故事 / 测试决策 / 实现决策（spec 主体）|
| `docs/adr/0001-self-improving-hook-architecture.md` | 16 决策的架构决策记录 |

## 实现状态

- ✅ Hook 安装（`~/.workbuddy/hooks/self_improving_bridge.py` + config）
- ✅ settings.json 注册 3 个 WorkBuddy 事件（UserPromptSubmit / PostToolUse (Bash) / SessionStart）
- ✅ Smoke test T1-T10 全 PASS
- ⏸️ Weekly upgrade cron 模板 status=PAUSED 等触发率证据

## 触发原理

```
WorkBuddy 事件 → hook stdin JSON → 推断事件类型 → subprocess 调 self-improving bash
  → activator.sh / error-detector.sh → 输出 reminder 注入白泽上下文
  → 白泽读 reminder → 决定是否写 .learnings/
```

## 关键设计原则

按主人分层原则（ADR 0001 决策 17）：

- **主动 = 价值观层**——决定看什么、想什么
- **克制 = 行为准则层**——决定怎么做
- 两层不冲突，**主动**给「方向 + 探索」，**克制**给「边界 + 风险」

## 关联

- 本地路径：`D:/WorkBuddy/2026-09-28-10-47-08/`
- 仓上游：白泽会话 2026-09-28 14:32-17:08 全流程产出
