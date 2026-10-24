---
schema_version: 1
module_id: yard-tank-clean
analysis_id: yard-tank-clean-20261024
document_type: ai-context
status: sample-feedback-pending
source_root: /www/yard/web
source_commit: 47c33b90efd47975f059431c221e30a20eec9553
source_dirty: true
source_changes: [AGENTS.md]
verified_date: 2026-10-24
timezone: Asia/Shanghai
verification: static-code
coverage: partial
developer_document: developer-guide.md
---

# 槽罐车业务：AI 接手上下文

用途：新 AI 接手槽罐车清洗模块时，快速找到实现、现状和修改影响。详细业务流程、31项功能、33组字段、Map、页面按钮与报表入口见 [developer-guide.md](/www/ai/skill-previews/code-module-feature-map/yard-tank-clean/developer-guide.md)。6张流程图使用字母及分支数字后缀编号，功能表通过“流程图/节点”对应图中字母，并标注操作端，稳定功能 ID 仍用 F-xxx，按钮沿用 BTN-xxx 关联功能；编号数量不表示全项目覆盖率。

本文是已核对代码的快照。新的需求、候选建议、测试方案和历史记录均不自动授权开发、业务操作或数据库写入；接手后先读取当前任务和目标项目规范。

## 1. 当前范围与快速结论

- 源项目 `/www/yard/web`，基线见 YAML；本次业务代码与提交一致，工作区仅 `AGENTS.md` 有未提交修改。`coverage=partial` 表示槽罐车主要端到端入口已静态核对，但财务通用服务、线上授权、实际数据库和运行验证不完整。
- 主体是槽罐车清洗，不是箱罐列表，也不是整个收入管理模块。电脑端在 AMIS `public/backend/pages/`，司机/工人端在 Uni-App 页面。
- 正常状态链：待确认 `1` → 确认后待开始 `2` → 开始后清洗中 `3` → 完成 `9`；撤销 `8` 与删除不是同一动作。
- 收入 `type=5`，关联 `connect_id=预约ID`；专项账单 `source_type=12`，两种类型不要混用。
- 作业与财务没有强制单一串行顺序；先收款后开票会产生已登记日记账但尚未核销的中间状态。

## 2. 建议阅读顺序与代码地图

| 顺序 | 位置/符号 | 接手用途 |
| --- | --- | --- |
| 1 | [目标项目规则](/www/yard/web/AGENTS.md)、[已有槽罐车说明](/www/yard/web/docs/m/b-m-tank-clean.md) | 按项目约定开发；旧说明作为线索，与当前实现核对，不能照搬已过时结论 |
| 2 | [AMIS主页面](/www/yard/web/public/backend/pages/yard/extraOpe/tankCleanOrder.js:4) | 预约、财务表单、Map、工具栏、行操作、图片及统计全部在此文件 |
| 3 | [后台控制器](/www/yard/web/app/controller/backend/yard/extraOpe/TankCleanOrderController.php:103) | `fillFinanceState()` 生成按钮控制标志；`detail()` 负责费用回显；核对真实返回值而非仅看方法注释 |
| 4 | [核心服务](/www/yard/web/app/common/service/basic/yard/extraOpe/TankCleanOrderService.php:140) | 事务、状态迁移、财务联动、删除约束、报表与证明 |
| 5 | [模型](/www/yard/web/app/common/model/yard/extraOpe/TankCleanOrderModel.php:21)、[验证器](/www/yard/web/app/common/validate/yard/extraOpe/TankCleanOrderValidate.php:29) | 实际表映射、关联、搜索器、分端必填及后端校验 |
| 6 | [司机表单](/www/yard/web/npm/mobile-cli/src/pages/driver/tankClean/form.vue:143)、[司机列表/详情](/www/yard/web/npm/mobile-cli/src/pages/driver/tankClean/list.vue:89)、[司机控制器](/www/yard/web/app/controller/driver/extraOpe/TankCleanOrderController.php:73) | 默认值、草稿、司机提交字段白名单与归属检查 |
| 7 | [工人列表](/www/yard/web/npm/mobile-cli/src/pages/worker/tankClean/list.vue:38)、[工人详情](/www/yard/web/npm/mobile-cli/src/pages/worker/tankClean/detail.vue:28)、[工人控制器](/www/yard/web/app/controller/api/worker/TankCleanOrderController.php:29) | 工人三种状态入口、开始/完成和图片上传 |
| 8 | [工人图片入口](/www/yard/web/app/controller/api/worker/TankCleanOrderImageController.php:28)、[后台图片入口](/www/yard/web/app/controller/backend/yard/extraOpe/TankCleanOrderImageController.php:26)、[图片服务](/www/yard/web/app/common/service/basic/yard/extraOpe/TankCleanOrderImageService.php:43) | 两端上传校验不同；本地/OSS、归属与软删除 |
| 9 | [专项账单页](/www/yard/web/public/backend/pages/yard/finance/generalIncomeAccount.js:542)、[账单导出](/www/yard/web/app/common/service/basic/finance/income/IncomeAccountService.php:1077) | 合并导出的权限、来源约束和12列格式 |
| 10 | [后台路由](/www/yard/web/app/common/route/backend/yardRoute.php:56)、[司机路由](/www/yard/web/app/common/route/driver/driverRoute.php:18)、[工人路由](/www/yard/web/app/common/route/api/workerRoute.php:12)、[权限种子](/www/yard/web/app/common/service/basic/platform/power/PowerService.php:437) | 方法存在、路由注册、菜单授权三者分开核对 |

## 3. 当前实现中影响开发决策的事实

### 3.1 状态与操作边界

- F-003：`confirm()` 只校验原状态 `1` 并写 `2`；它不是“保存并确认”接口。当前方法描述包含同步表单等旧信息，不能据此扩展行为。
- F-004/F-005：`start()` 要求 `2`，写 `3` 和开始时间；`complete()` 要求 `3`，写 `9`、结束时间及 `verify_time`。
- F-013：普通编辑检查撤销和财务锁定；已完成但未锁财务的记录仍可能普通编辑。F-006 独立编辑时间不走这些财务锁，服务只排除撤销；页面单独限制完成状态。
- F-009：司机页面只显示待确认取消；服务不限定原状态为 `1`，但拒绝 `8/9` 和收入已确认。电脑端 `cancel()` 方法存在，路由却未注册；司机路由仍可用。
- F-015 / BTN-021：预约删除的服务还检查部分开票/收款，页面 `has_invoice/has_payment` 主要由关联对象及全额状态生成。删除预约及收入不等于级联撤销账单、发票、日记账和图片文件。

### 3.2 字段与 Map 的易错点

- FIELD-006：车挂在电脑创建为可选，在司机提交为必填；不要只看通用 `checkOptionalPositive()` 推断所有端一致。
- FIELD-007：`plate_number` 是关联拼接展示值，模型无同名搜索器；方法参数注释写了车牌关键字也不证明筛选已经实现。
- FIELD-009：货类数值相同但展示方法不同：电脑/司机 `getDesc()`，工人/账单导出 `getNumberedDesc()`。`20` 普货不带编号。
- FIELD-019：`IN_PROGRESS=1` 当前含义为待确认，不是清洗中；清洗中是 `STARTED=3`。
- FIELD-021：支付方式常量 `ALIPAY/CASH/OTHER` 当前文案为扫码/对公转账/月结；`WECHAT=1` 存在且会通过 cases 校验，但 `label()` 返回 `-`。勿按常量名或旧迁移注释重写业务。
- FIELD-026：费用对象中的 `need_invoice` 必须明确；页面默认、司机提交转换、验证器与服务归一逻辑分开看。
- FIELD-028/FIELD-031：本模块费用创建使用自己的一套默认与计算；四位中间精度与两位 UI 显示不代表数据库精度。更新收入还会进入通用收入服务及模型钩子。
- `invoice_state_format/collection_state_format` 是派生字符串，不是原始状态。页面虽然有“部分”Map，当前 `fillFinanceState()` 并不直接产生对应部分状态文案。

### 3.3 页面权限与后端允许执行

- 后台前缀 `yardExtraOpe.tankCleanOrder.`，按 `read/add/edit/confirm/start/complete/del` 分别判断。
- BTN-008/009：图片弹窗需要父入口 `read`，内部上传/删图另需要 `edit`。不能仅复制子按钮表达式而遗漏父容器。
- BTN-018/020：开票/重传/收款按钮无额外财务权限表达式，依赖 `can_*`。这些标志由控制器计算，必须与服务校验一起阅读。
- `can_invoice` 有两个 OR 分支，第二个分支是无需开票且有收款无发票；开票服务又明确拒绝无需开票。页面对撤销状态的隐藏，也没有对应为模块 `invoice/collection` 方法中的显式撤销校验；不能假定 UI 与后端完全一致，共享财务服务约束须另行复核。
- 工人 API 登录中间件不等于细分岗位授权证明。司机详情/取消有明确 `driver_id+corp_id` 检查；后端普通数据范围仍需理解基础服务。
- BTN-034：账单导出接受两种导出权限之一，且批量动作是否被构造还受固定来源控制。不能把通用账单导出条件照搬到槽罐车清洗主页面。

### 3.4 财务、图片及证明

- F-007：首次快捷开票创建并确认专项账单，再登记实际发票；如已存在账单，内部创建账单会拒绝。F-014 重传发票在已全额开票时仅更新预约独立关联的发票，不替代通用发票操作。
- F-008：收款创建日记账；需开票未开时暂不核销，后开票时尝试自动核销。无需开票时创建模拟发票参与核销，模拟发票不是对外实际票据。
- F-016：工人上传需要 `state∈{3,9}`，后台图片入口没有同等状态限制。手机 `uploadNext()` 当前只处理传输 success/fail，未检查响应业务状态。
- F-019/F-020：证明接口会生成文件，本轮未调用。证明的 `0.5Bar`、是否有压及固定声明来自模板/清洗类型，不是现场检测数据。当前接口和页面均未统一要求已完成。
- F-012：统计未按币种分组、未默认只看已完成；`collection_amount` 是收入核销口径，先收未核销的现金不能直接计入。F-021 槽罐车账单导出12列，不复用旧格式的额外费用列。

## 4. 按需求定位修改范围

| 需求 | 优先检查 | 必须评估的关联影响 |
| --- | --- | --- |
| 修改预约字段/默认值/必填 | 两端表单、三类控制器、验证器及服务白名单 | 司机与后台输入差异、回显、收入嵌套字段、声明迁移及真实表结构 |
| 调整状态/按钮权限 | 页面表达式、`fillFinanceState()`、服务状态校验、路由与权限来源 | 工人列表/详情重复入口、父容器条件、历史状态、直接 API 调用 |
| 修改支付方式/货类/清洗选项 | 三份相关枚举、电脑表单、司机选项、后端校验 | 旧值兼容、工人展示、财务统计、账单导出、证明模板 |
| 修改财务流程 | 本服务 `upsertIncomeItem/invoice/collection/_createAccount/autoWriteOff` | 通用收入/账单/发票/日记账服务、撤销状态、先收后票、重复请求和并发 |
| 修改图片上传/删除 | 两端页面、两个图片控制器、图片服务与模型 | 业务错误反馈、状态/企业/归属校验、本地/OSS、软删与文件清理边界 |
| 修改统计或导出 | `customStatistic()`、统计页签、`IncomeAccountService::exportTankCleanAccounts()` | 币种、核销口径、未设置客户、筛选透传、合并来源及Excel格式 |
| 接通日志 | 主页面日志 source、当前 `log()`、现有日志字典与统一查询机制 | 当前方法固定空数组，不能只改配置或沿用旧文档“已接通”结论 |

## 5. 已知问题、候选建议与需求状态

候选建议 OPT-001～OPT-008 详见开发者文档：日志空实现、支付历史值、工人上传状态/反馈、权限与状态一致性、统计口径、缓存隔离及预约来源。

本轮用户已确认的任务仅为“以槽罐车业务制作真实样例”。没有已批准的业务代码优化清单或下一版本功能计划；所有候选建议均未实施。正式样例验收标准与额外操作边界待用户反馈后明确。

## 6. 后续开发验证方向

当前文档以源码静态核对为基础。浏览器 AMIS 渲染、手机客户端流程、实际登录授权、请求返回、后端集成测试及实际数据库未核验；现有测试目录未发现明确的槽罐车专用自动化测试，不盲目运行可能写库的脚本。

后续若用户授权实现相关需求，按影响范围选择隔离环境用例：

| 用例方向 | 应核对的行为 |
| --- | --- |
| 正常主流程 | 两端创建 → 后台确认 → 工人开始/完成，核对状态与时间及收入关系 |
| 字段/枚举边界 | 缺车挂的两端差异；支付 `1/2/3/4`；未知货类/状态；空时间与时间倒置 |
| 权限/状态 | 无操作权限、父容器权限缺失、跨企业/司机、已撤销、已确认/部分及全部财务状态 |
| 上传 | 待开始被拒、清洗中/已完成可传；HTTP/业务错误不能误显示整批成功；重复点击与部分成功 |
| 财务 | 未设客户/零额、需票先收后开、无需开票、独立发票缺失、已有账单、重复/并发请求 |
| 统计/文件 | 混币、未设置客户、先收未核销、跨来源合并、12列格式、未完成记录证明生成策略 |
| 日志/缓存 | 请求是否返回真实日志；同设备切换司机是否残留草稿和最近车牌 |

## 7. 人工背景与决策

用户已要求以槽罐车业务制作真实样例，并按当前 skill 重新生成开发者说明。现行文档不再单列角色/页面操作矩阵；流程图使用 A-B-C-D、分支建议 A-B1-C1-D1，功能表对应图中字母并列明电脑/手机操作端；按钮表描述功能与可用条件，报表固定使用“报表名称、报表链接、简述”三列，不另附本轮验证报告或维护日志。

本轮未提供已批准的业务代码优化清单或下一版本功能计划；OPT-001～OPT-008 均为候选建议。正式样例验收标准待用户反馈，不能用静态分析替代业务验收。

后续使用本文前重新核对源项目规范、代码基线及工作区变化；保留现有功能/字段/按钮/建议/证据编号和人工决策。实际授权与数据库缺口见第1、3节，外部档案和通用财务边界见开发者说明第1、2节；不使用其他项目数据库替代堆场核验。
