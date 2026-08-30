---
name: personal-assistant-intake
description: "滴答清单复杂操作：编号批量、结束重复、checklist、清空字段、部分完成、进度对账、窄时段/全部日程、cron"
version: 2.0.0
author: Hermes Agent
license: MIT
platforms: [macos, linux, windows]
metadata:
  hermes:
    tags: [personal-assistant, intake, scheduling, ideas, ticktick, dida365]
---

# Personal Assistant Intake（复杂场景）

热路径——消息分类、时段约定、最短路径、回复格式——在 SOUL.md。本技能只覆盖下面的分支。

**停止条件（所有分支通用）**：每个受影响任务一次写入 + 一次按 ID 回读；回读的日期/时区/全天、标题、备注、提醒、重复、项目、status 与意图一致即停，不一致才继续修。

## 分支 → 参考

| 请求 | 加载 |
|---|---|
| 窄时段（"下午的日程"）、"今天之后全部日程"、今日已完成汇总、重排前总览 | `references/agenda-queries.md` |
| 编号批量操作、结束整组重复、checklist、优先级重排、部分完成、进度汇报对账、文档作证据、想法升级为待办、任务关联、模糊顺延、截图事件 | `references/task-edits.md` |
| 创建或修改 cron（次日顺延、日程简报、提醒型自动完成） | `references/automation.md` |
| 写入前拿不准 API 行为：全天编码、时段折叠、repeatFlag/空值、checklist ID 漂移、sortOrder、批量超时、move_task、删除核验 | `references/dida365-api.md` |

一次请求只加载命中的参考；同一会话内已加载的不再加载。

## 通用流程

1. **定位**：会话里已有 ID 直接用；否则 `search_task` 找候选 → `get_task_by_id` 拿完整字段。搜索结果是候选，回读才是事实。
2. **日期**：相对日期用 `date` 读一次，整条消息一次性解析；确认里给绝对日期。
3. **写入**：原地更新同一任务 ID；checklist 提交完整条目集；跨项目移动用 `move_task`。
4. **回读**：按停止条件核对。
5. **回复**：已完成 / 进行中 / 延期（绝对日期 + 原因）/ 方案替换，一行一项。

## 陷阱

- 模糊搜索当证据 → 历史重复项污染上下文，仍证明不了当前状态。
- MCP 调用返回就当成功 → 空值"不修改"、时段折叠、move_task 全 null 都需要回读才知道。
- 编号按新搜索顺序重解释 → 只用用户最后看到的那份编号表。
- 把"待测试""正在做"当完成 → 只勾选有证据的里程碑。
- 列日程超出请求范围 → "只列今天"就只有今天。
