---
schema_version: 1
module_id: yard-income-item
analysis_id: yard-income-item-20261024
document_type: developer-guide
status: sample-feedback-pending
source_root: /www/yard/web
source_commit: 47c33b90efd47975f059431c221e30a20eec9553
source_dirty: false
verified_date: 2026-10-24
timezone: Asia/Shanghai
verification: static-code
coverage: partial
---

# 收入单据：开发者功能说明（真实代码样例）

本样例分析堆场后台“财务管理 → 收入管理 → 收入单据”，覆盖列表查询、新建、导出、快捷编辑和共享列 Map。未完整展开收入账单、发票、收款、差异审批、Vue 批量新增与箱子详情。没有登录页面、调用业务接口或查询数据库；线上角色、实际表结构及部署行为待确认。

AI 接手入口：[ai-context.md](ai-context.md)。同名角色的实际授权以运行配置为准，本文用权限码描述可确认范围。

## 业务流程

| 功能 ID | 谁、页面与操作 | 实现与结果 | 证据 |
| --- | --- | --- | --- |
| F-001 | 具有收入单据菜单访问条件的后台用户，在收入单据页筛选客户、箱号、币种和账单状态 | 页面 POST `page`，服务关联箱子及业务对象，返回分页记录、列配置和金额统计 | E-001、E-003、E-004 |
| F-002 | 到达该页的用户点击“新建”，填写箱号、客户、费用归属、费用名称、单价等 | POST `create` → `redirectCreate()`；服务端固定 `type=2`，计算实际/预算费用，再创建收入单据 | E-001、E-003、E-005 |
| F-003 | 用户在列表快捷编辑单价、数量、其他费用、单位、总金额、税率或备注 | 前端要求 `is_confirm == 1`；POST `item?edit_type=quitEdit`；控制器限制字段，服务更新后触发差异提醒检查，金额变更触发模型重算 | E-001、E-002、E-003、E-005、E-006 |
| F-004 | 用户点击“导出全部” | POST `page?is_export=1`；服务在 `is_export` 非空时跳过分页；导出文件由前端动作处理，未实际运行 | E-001、E-004 |

收入单据记录的是应收费用。上述列表中的实际金额不是银行到账金额；本样例不覆盖从账单到开票和收款核销的完整结算链。

`type=11` 的页面复用为“货代收入单据”，固定过滤类型并使用 `huodaiOpeIncome` 列配置。任意非空 `type` 都会使本页新建入口不被插入，而不只是 `type=11`。[E-001]

## 关键字段定义与流转

以下字段为样例选取，非页面全部字段清单。字段均未核对实际数据库类型、长度、可空性和数据库默认值。

| 字段 ID | 页面字段与含义 | 请求/响应与模型位置 | 来源或约束 | 证据 |
| --- | --- | --- | --- | --- |
| FIELD-001 | `box_id`：关联箱子；`box_number`：展示箱号 | 新建提交 `box_id`；列表 `box_number` 为关联查询结果 | 新建表单必填；展示箱号取货代箱号与堆场箱号的 `COALESCE` 结果，不能当作收入表原始列 | E-001、E-004 |
| FIELD-002 | `custom_id`：付费客户 | 新建字段 → 收入模型同名字段 | 页面必填，候选项来自 `/b/p/custom/options`；本样例未展开该接口的请求方式与客户有效性校验 | E-001、E-005 |
| FIELD-003 | `amount`：含税实际金额 | 页面公式/快捷编辑 → 服务计算 → 模型同名字段 | 新建展示为禁用计算项；服务按单价×数量＋其他费用重算；修改总金额会进入反算逻辑 | E-001、E-005、E-006 |
| FIELD-004 | `budget_amount`：预算金额 | 服务端生成 → 收入模型字段 | 直接新建时通常从输入计算；负数实际收入的预算置零，不能认为预算永远等于实际 | E-005 |
| FIELD-005 | `is_account`：账单关联状态 | 页面 Map、枚举、收入模型同名字段 | 实际值为 `1/8/9`；旧模型注释的 `2=已开账单` 与实现冲突 | E-002、E-006、E-007 |
| FIELD-006 | `is_confirm`：账单确认状态 | Map、枚举及快捷编辑条件 | `1` 才允许当前页快捷编辑；`8` 枚举存在，但本列缺少对应 Map 文案 | E-002、E-007 |

新建表单数值默认：币种 `1`、费用归属 `1`、数量 `1`、其他费用 `0`、税率 `6`、单位“次”。服务在缺少税率时使用 `0`，不能把页面默认值当作服务默认值。[E-001、E-005]

页面金额显示/输入通常为两位小数；后端 `FinanceTool::taxCompute()` 使用四位小数计算。文档应保留计算口径差异，不统一改写成“两位小数存储”。[E-001、E-010]

## Map 与枚举对照

| 绑定字段 | 页面 Map 的实际键值 | 后端依据与差异 | 证据 |
| --- | --- | --- | --- |
| `type` | `1=任务关联；2=直接创建；3=堆场自动生成；4=工人外派；5=槽罐车清洗；11=货代费用` | `IncomeTypeEnum` 和模型 `typeMap` 另有 `12=运输业务`，共享页面 Map 未配置 `12`，也没有 `*`；具体未命中显示效果待运行确认 | E-002、E-006、E-007 |
| `currency` | `1=人民币；2=美元` | 本样例已核对表单与共享列 Map，未展开所有币种来源 | E-001、E-002 |
| `is_account` | `1=未开账单；8=无需账单；9=已开账单` | `IsAccountEnum` 与页面一致；旧模型注释不一致 | E-002、E-006、E-007 |
| `is_confirm` | `1=未确认；9=已确认` | `IsConfirmEnum` 另含 `8=无需确认`；是否在此页出现该状态待确认 | E-002、E-007 |
| `diff_approval_status` | `1=正常；2=待填写原因；3=审批中；4=已拒绝；*=正常` | 后端枚举含四个明确值；未知值在本列被映射成正常，可能掩盖异常状态 | E-002、E-007 |
| `invoice_status` | `1=未开票；2=部分开票；9=已开票；*=未开票` | 本次只确认页面 Map；不能用支出模块“收票状态”的同名枚举替代收入语义 | E-002 |

Map 中的 HTML 标签负责状态样式，本文保留业务文案；`*` 是配置的兜底键，不是数据库合法枚举值。字段显示还受 `columnField` 控制。[E-002、E-004]

## 页面动作、显示权限与请求地址

前提：菜单定义包含 `yardIncome.item`，并定义 `.read/.add/.edit/.del` 权限点；实际用户角色绑定需查运行配置。下表分别描述本页代码条件，不把菜单种子中的权限码自动视作每个按钮已经接入了权限校验。[E-008]

| 动作 ID | 页面操作 | 显示/可用条件 | 页面发出的请求 | 证据 |
| --- | --- | --- | --- | --- |
| BTN-001 | 查询 | 本页表单无额外权限表达式，页面可达性前提仍适用 | POST `/b/y/finance/incomeItem/page`，附筛选与固定类型参数 | E-001 |
| BTN-002 | 重置 | 本页本地重置动作，未配置独立 API；是否随后刷新依赖 AMIS 运行行为 | 无独立请求地址 | E-001 |
| BTN-003 | 新建/弹窗提交 | `fixedType` 为空时插入“新建”；页面未见 `.add` 绑定表达式；表单必填约束仍适用 | 提交 POST `/b/y/finance/incomeItem/create`；打开弹窗本身不写入 | E-001、E-003 |
| BTN-004 | 导出全部 | 本页动作无额外显示/禁用条件 | POST `/b/y/finance/incomeItem/page?is_export=1`，另附固定类型参数 | E-001、E-004 |
| BTN-005 | 行内快捷编辑 | 列存在于 `columnField` 且 `this.is_confirm == 1 && 1 == 1`；本页传入 `can_edit=1` | POST `/b/y/finance/incomeItem/item?id=<id>&edit_type=quitEdit` | E-001、E-002、E-003 |
| BTN-006 | “移除”行操作 | 本页传入 `ope_button_array=[]`，操作列 `visibleOn` 为 `false`，因此当前页不展示；其他调用页可能不同 | 共享组件配置了 `/b/y/finance/incomeAccount/deleteIncome`，但不是当前页可用操作 | E-001、E-002 |
| BTN-007 | 删除 | 共享组件中的删除配置已注释，当前页没有生效的删除按钮 | 后端 `/b/y/finance/incomeItem/delete` 仍有路由；接口存在不等于本页有入口 | E-002、E-009 |

前端 `post:` 是 AMIS 请求方式前缀，不属于 URL。后端由 `route_map()` 的 `Route::any()` 注册相应路由，前端实际发 POST 与后端允许的方法范围分别记录。[E-009]

后端共同限制：路由组挂载登录检查；基础服务针对后台企业登录注入 `corp_id`。`delete()` 服务会拒绝 `is_confirm=9` 的记录。快捷编辑控制器限制 `quitEdit` 可写字段，但已阅读的 `update()` 链路没有同等的“已确认不可编辑”检查。按钮权限码的后端强制校验在本次已读链路内未发现；插件或其他运行入口未完全核验，不据此声称“所有登录用户都能越权”。[E-003、E-005、E-009、E-011]

## 优化建议

| 建议 ID | 类型与问题 | 建议与验收方向 | 优先级与依据 |
| --- | --- | --- | --- |
| OPT-001 | 已确认文档差异：模型旧注释的账单状态与枚举冲突 | 对齐字段注释与枚举；核对 `1/8/9` 的表单、筛选和列表展示 | 中；避免开发者按错误状态写条件，E-006、E-007 |
| OPT-002 | 已确认映射缺口：类型 `12`、确认状态 `8` 不在对应 Map 中 | 先确认这些值是否在此页出现，再补映射和未命中展示规则；用对应记录做非写入展示验证 | 中；具体展示影响待运行确认，E-002、E-007 |
| OPT-003 | 条件性权限一致性建议：菜单有操作权限点，本页按钮未显式绑定；编辑状态限制主要在前端 | 核对全局封装后统一按钮权限及服务端状态校验；在隔离测试环境验证无权限和已确认记录的拒绝行为 | 高；涉及费用修改，不能只依赖按钮显示，E-001、E-005、E-008、E-009 |

以上均未实施，不表示本轮已修改业务代码。

## 证据索引

| ID | 代码位置与关键符号 |
| --- | --- |
| E-001 | [收入单据页面](/www/yard/web/public/backend/pages/yard/finance/incomeItem.js:4)：`BASE_API`、`fixedType`、`form()`、`headerToolbar`、`quickSaveItemApi`、`incomeColumn(1, [])` |
| E-002 | [共享列](/www/yard/web/public/backend/pages/yard/finance/components/finance.js:8)：`incomeColumn()`、Map、`quickEditEnabledOn`、操作列 |
| E-003 | [控制器](/www/yard/web/app/controller/backend/yard/finance/IncomeItemController.php:57)：`page/create/update/item/delete` |
| E-004 | [分页与列选择](/www/yard/web/app/common/service/basic/finance/income/IncomeItemService.php:1308)：`yardPage()`、`getTableColumn()` |
| E-005 | [收入服务](/www/yard/web/app/common/service/basic/finance/income/IncomeItemService.php:94)：`update()`、`delete()`、`redirectCreate()` |
| E-006 | [收入模型](/www/yard/web/app/common/model/finance/income/IncomeItemModel.php:54)：旧注释、`typeMap`、`onBeforeUpdate()`，物理表映射为 `901_income_item` |
| E-007 | [账单枚举](/www/yard/web/app/common/enum/finance/IsAccountEnum.php:12)、[确认枚举](/www/yard/web/app/common/enum/finance/IsConfirmEnum.php:12)、[类型枚举](/www/yard/web/app/common/enum/finance/income/IncomeTypeEnum.php:10)、[差异枚举](/www/yard/web/app/common/enum/finance/income/DiffApprovalStatusEnum.php:7) |
| E-008 | [权限菜单种子](/www/yard/web/app/common/service/basic/platform/power/PowerService.php:523)；`getPower()` 在同文件第 86 行开始解析角色权限 |
| E-009 | [路由生成](/www/yard/web/app/common/route/route.php:14)、[收入路由](/www/yard/web/app/common/route/backend/yardRoute.php:69)、[登录中间件](/www/yard/web/app/common/middleware/login/LoginCheckMiddleware.php:21)、[全局中间件](/www/yard/web/config/middleware.php:17) |
| E-010 | [金额计算](/www/yard/web/app/common/tools/base/FinanceTool.php:42)：`taxCompute()` |
| E-011 | [基础服务](/www/yard/web/app/common/service/basic/BasicBaseService.php:51)：`search()`、`update()`；[控制器基类](/www/yard/web/app/controller/BaseController.php:40) |

## 待确认与样例反馈

- Q-001：线上各角色与权限点的实际绑定、页面外层公共权限控制。
- Q-002：实际数据库约束与历史迁移是否应用；本次没有可确认的堆场数据库查询结果。
- Q-003：`type=12`、`is_confirm=8` 在此页是否可达，以及 Map 未命中时的实际展示。
- Q-004：`is_account`、`is_confirm` 的完整写入方和状态迁移，需扩展账单流程分析。
- 本样例是静态代码产物，不宣称已完成浏览器验证、接口测试或收入模块全量整理。
- 请求反馈：两份文档的分工、Map 展开深度、按钮表粒度是否符合实际使用习惯？合格标准与额外操作边界在反馈后补充。
