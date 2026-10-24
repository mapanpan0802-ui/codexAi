---
schema_version: 1
module_id: yard-income-item
analysis_id: yard-income-item-20261024
document_type: ai-context
status: sample-feedback-pending
source_root: /www/yard/web
source_commit: 47c33b90efd47975f059431c221e30a20eec9553
source_dirty: false
verified_date: 2026-10-24
timezone: Asia/Shanghai
verification: static-code
coverage: partial
developer_document: developer-guide.md
---

# 收入单据：AI 接手上下文（真实代码样例）

先读本文定位实现，字段完整语义和按钮证据见 [开发者功能说明](developer-guide.md)。接手后先核对代码基线及工作区变化；本文是现状说明，不授权实施建议、修改数据库或执行文档中的业务动作。

## 当前范围与阅读顺序

范围：收入单据 AMIS 列表页、共享列、查询/新建/快捷编辑/导出入口及相关 Map。状态：部分静态核验。未覆盖收入账单、发票、收款、完整审批、Vue 批量新增和线上角色配置。

| 顺序 | 文件/符号 | 用途 |
| --- | --- | --- |
| 1 | [AGENTS.md](/www/yard/web/AGENTS.md) | 项目规范、文档约定、数据库只读约束；每次按目标项目当前规则执行 |
| 2 | [incomeItem.js](/www/yard/web/public/backend/pages/yard/finance/incomeItem.js:4) | 实际列表页及 `BASE_API`，不能只搜索 Vue 页面 |
| 3 | [finance.js](/www/yard/web/public/backend/pages/yard/finance/components/finance.js:8) | `incomeColumn()` 提供列 Map、快捷编辑和行操作；收入与支出共用此文件 |
| 4 | [IncomeItemController](/www/yard/web/app/controller/backend/yard/finance/IncomeItemController.php:57) | `page/create/item/update/delete`；快捷编辑字段白名单 |
| 5 | [IncomeItemService](/www/yard/web/app/common/service/basic/finance/income/IncomeItemService.php:94) | 更新后差异提醒、`yardPage/getTableColumn`、`redirectCreate/delete` |
| 6 | [IncomeItemModel](/www/yard/web/app/common/model/finance/income/IncomeItemModel.php:68)、[FinanceTool](/www/yard/web/app/common/tools/base/FinanceTool.php:42) | 物理表映射、金额更新钩子与计算 |
| 7 | [yardRoute](/www/yard/web/app/common/route/backend/yardRoute.php:69)、[PowerService](/www/yard/web/app/common/service/basic/platform/power/PowerService.php:86) | 真实路由与权限来源；菜单定义不能代替接口授权证据 |

## 已核对的关键事实

- F-001：页面实际发 POST `/b/y/finance/incomeItem/page`；`route_map()` 使用 `Route::any()`；前端 `post:` 前缀不是 URL 内容。
- F-002：新建走 `redirectCreate()`，固定 `type=2`、堆场所属系统；缺少税率时服务默认 `0`，页面默认 `6`。
- F-003：快捷编辑由 `incomeColumn(1, [])` 开启，前端条件是 `is_confirm==1`；提交接口为 `item?edit_type=quitEdit`。控制器接受单价、数量、其他费用、单位、总金额、税率、备注及 ID。
- F-004：导出复用 `page?is_export=1`，服务跳过分页；本次未验证浏览器导出结果和大数据量表现。
- FIELD-005：`is_account` 实际为 `1/8/9`，旧模型注释的 `2` 不可靠。
- FIELD-006：`is_confirm` 枚举为 `1/8/9`，共享 Map 只包含 `1/9`；类型枚举存在 `12`，共享 Map 未包含它。
- BTN-006：当前列表传入空操作数组，操作列隐藏；共享组件存在“移除”配置不代表本页有这个按钮。
- BTN-007：删除按钮配置已注释，但接口路由与服务仍存在；服务会拒绝 `is_confirm=9` 的删除请求。

## 迭代时应保留或重新确认的规则

- 新建时实际金额与预算分开计算；负数实际收入预算置零。快捷编辑金额会触发模型重算及服务差异提醒检查。相关逻辑变更需检查所有更新入口，不能只改页面公式。
- 显示条件、是否可编辑、角色授权和服务端状态校验分开判断。前端禁用字段不能证明后端拒绝对应修改。
- `type/source_type/is_account/is_confirm/invoice_status/diff_approval_status` 含义不同，值相同不代表同一业务语义。不要将支出收票枚举的文字用于收入开票状态。
- 共享 `finance.js` 同时定义支出列；修改 `incomeColumn()` 还影响收入账单、专项账单中的嵌入列表。先搜索所有调用位置及其 `can_edit/ope_button_array` 实参。
- 代码中的菜单种子不是线上角色绑定快照；所有角色名、实际 DDL 和运行验证缺口保留待确认状态。

## 按需求定位修改范围

| 需求类型 | 首要检查位置 | 需要同时核对 |
| --- | --- | --- |
| 新增或调整列表状态文案 | `finance.js::incomeColumn`、相关枚举 | 后端实际取值、兜底 `*`、`columnField` 和其他调用页面 |
| 修改可编辑条件 | `incomeColumn`、`incomeItem.js`、控制器 `item`、服务 `update` | 后端状态检查与所有调用入口，不只依赖前端表达式 |
| 新增字段 | 页面控件、控制器白名单、服务 `addFields/editFields`、模型映射 | 计算来源、响应字段、实际表结构；数据库变更只提出方案，不执行 |
| 修改金额计算 | `redirectCreate/update`、模型钩子、`FinanceTool` | 预算/实际、负数收入、计算精度及差异提醒；本样例未穷尽关联流程 |

## 已知问题与候选建议

OPT-001：旧注释与枚举冲突。OPT-002：Map 有未覆盖值。OPT-003：需核对页面按钮权限与后端约束的一致性。完整证据、优先级与验收方向见开发者文档；三项均未实施，不属于当前已批准的开发任务。

## 接手时的验证方向

以下是建议用例，尚未运行，涉及写入的验证必须在另行授权的隔离环境进行：

- 状态 Map：`type=12`、`is_confirm=8` 和未知差异状态的展示。
- 页面动作：普通入口与 `type=11` 复用入口的新建按钮差别；空操作数组下不存在行移除入口。
- 字段与业务：默认税率差异、负数实际收入预算、金额更新后的重算结果。
- 权限与状态：无操作权限、非本人企业记录、已确认收入的修改/删除；当前样例只证明删除状态检查存在。

## 后续维护

每次功能迭代后同步更新两份文档的同一 `analysis_id`、核验基线、覆盖范围和关联条目。原有 ID 不重排；只复核部分时不能把整个收入模块标为已完成。

本次记录：2026-10-24，首次生成真实页面样例；未修改 `/www/yard/web`，未执行页面操作或数据库查询。待用户反馈后完善验收口径及额外边界。
