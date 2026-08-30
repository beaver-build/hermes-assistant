# Dida365 MCP：已验证的 API 行为

所有结论来自 2026-08 实测；服务升级后以当前工具 schema 为准。本文件只记行为，token、任务内容一律留在 .env 和滴答里。

## 接入与验证

- 端点 `https://mcp.dida365.com`，Bearer 认证；用 `hermes mcp add dida365 --url … --auth header`，token 走隐藏输入（shell 参数会进历史和日志）。
- 验证：`hermes mcp list` 显示已启用，`hermes mcp test dida365` 显示连接成功、Authorization 已遮蔽、发现工具。新工具在新会话生效。
- 工具在注册表中带前缀 `mcp__dida365__`，经 `tool_call` 调用；原始名：`list_projects / get_project_by_id / create_project / update_project`、`create_task / update_task / get_task_by_id / get_task_in_project / delete_task / complete_task / move_task`、`search_task / search / filter_tasks / list_undone_tasks_by_date / list_undone_tasks_by_time_query / list_completed_tasks_by_date`、`batch_add_tasks / batch_update_tasks`、`get_user_preference`（时区）。
- 「想法」项目：`list_projects` 精确匹配"想法"，有则复用 ID，无则 `create_project`；项目 ID 是运行时状态，从查询结果取。

## 日期与时段

- 时区以 `get_user_preference` 为准。服务端返回 UTC，展示前换算回本地。
- **全天任务编码**：本地某日全天 = `startDate` = `dueDate` = 本地 00:00 换算成的 UTC 时刻（UTC+8 下为前一天 `16:00:00+0000`，如 8月18日全天 = `2026-08-17T16:00:00+0000`），`isAllDay=true`。读到前一天的 UTC 时刻是"次日全天"，不是"前一天下午"。
- **定时区间会被折叠**：`update_task` / `batch_update_tasks` 提交不同的 `startDate`/`dueDate`，服务端可能归一为同一时刻（以 `dueDate` 为准）；单独写其中一个也会同步另一个。整段语义的可靠写法：`startDate` = `dueDate` = 段起点（上午 08:00 / 下午 13:00），`isAllDay=false`，`timeZone` 为用户时区，零偏移提醒在起点，`content` 写完整区间（如"周二下午（13:00–18:00）"）。写后比对原时长；被折叠就停止覆盖，保留起点钟点和提醒，并在回复里说明时长未保留。
- 范围接口 `list_undone_tasks_by_date` 有范围上限；结果可能倒序；可能返回请求钟点窗口之外的任务（查 12:00–18:00 仍返回上午、逾期、全天、跨期）；可能漏掉重复主任务的当期实例和刚移动的定时任务。它是候选集，过滤、排序、补查在本地做。
- `list_completed_tasks_by_date` 只返回已完成的父任务；打开状态的 CHECKLIST 父任务里已勾选的子项要另行回读。

## 部分更新与空值

- `repeatFlag: null` = 不修改；清除重复规则用 `repeatFlag: ""`。
- `reminders: []` 等空数组也可能被当作不修改；清除字段用当前 schema 明确支持的值，回读仍有旧值就如实说"未清除"。
- 一次 `update_task` 里同时改 `projectId` 和日期可能返回空响应或 JSON 解析错误：先在原项目 `update_task` 改日期，再 `move_task(fromProjectId, toProjectId, taskId)`。`move_task` 返回体大多为 null，只留 ID/etag——按 ID 回读判断结果。

## 重复任务

- **结束整组重复**（用户说"结束了""不用再提醒"）：`get_task_by_id` 核对 `repeatFlag` → `update_task` 置 `repeatFlag: ""` → 回读确认为空 → `complete_task(projectId, taskId)` → 回读 `status: 2`、`completedTime` 存在、`repeatFlag` 为空。先完成再清规则会生成下一次实例。
- **提醒型每日重复**：只 `complete_task` 当前实例，回读确认主任务推进到未来日期；仍停在今天或更早就再完成一次，直到越过；`repeatFlag` 保持不动。
- 过期的每日重复完成一次后可能推进到同一天的另一实例，同样按回读判断。

## Checklist

- `kind: CHECKLIST` 的任务 `update_task` 时，即使带 `items[].id`，服务端也可能重建全部条目并分配新 ID；父任务 ID 不变。
- 提交完整的目标条目集（保留项 + 新增项），服务端不做部分合并。
- 子项完成 `status: 1`；父任务完成 `status: 2`。父任务完成顺序：先更新标题、备注、全部子项，再 `complete_task`，再回读。
- 后续勾选只用回读得到的最新 `items[].id`。

## 排序与优先级

- 精确执行顺序用 `sortOrder`：值越负越靠前；用安全间隔的递减序列（int64 范围内）。
- `priority` 只有 0/1/3/5 四档，表达不了"1…5 的精确顺序"。
- 范围接口按全天/优先级分组展示时，以逐任务回读的 `sortOrder` 为准报告执行顺序。

## 批量与超时

- 批量更新超时 = 结果未知，不是失败：逐个按父任务 ID 回读，只对未落地的任务重试。
- 复杂 checklist 批量写入超时后，改为逐项 `update_task`，每项带最新 `etag` 和完整任务内容。
- 超时后直接再批量提交会覆盖较新进度。

## 检索、去重与删除

- `search_task` 是模糊检索，长标题会返回大量只共享个别词的结果。查重同时核对：标准化标题 + 项目 + 状态 + 本地日期时间；没有精确候选才创建。
- 已删除的任务按 ID 读不到：用精确标题搜索 + 相关日期/项目列表核验"确已不在"，单次 ID 读失败不算证据。

## 写入证据分级

1. `create_task` / `update_task` 返回目标 ID 和期望字段；
2. `get_task_by_id` 回读并核对全部关键字段（宣称时间、提醒、重复前必须有）；
3. 回读超时时，精确标题搜索只证明"存在"。
