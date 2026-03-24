# 公司 AI Agent 系统 PRD v1.1（MVP 修订稿）

基于 `Company_Agent_PRD_v1.0.docx` 的评审结论与业务方确认，本文档用于收敛一期范围，降低交付风险，并补足可测试定义。

## 1. 一期范围（MVP）与二期边界

### 1.1 一期MVP（25天内交付）
- 请假审批：完整流程 + 状态机
- 报销审批：固定三级审批规则 + 发票OCR基础能力
- 知识库问答：基础RAG（上传、检索、引用）

### 1.2 明确不在一期范围
- 企业微信对话能力（仅保留扩展接口）
- 复杂审批流与自定义规则引擎
- 销售线索管理
- 公章流程（移至二期）
- 工龄自动计算、复杂假种规则

### 1.3 二期预留
- 企业微信Webhook接入
- 企业微信账号映射
- 自定义审批流配置
- 高级请假规则（工龄、假种策略）

## 2. 统一技术与集成决策

- 登录：统一采用钉钉SSO
- 用户生命周期：入离职由钉钉同步，不在系统内维护
- 日历：一期统一使用钉钉日历（不使用Google Calendar）
- 审批渠道：付款与报销统一在钉钉财务模块执行
- 知识库规模：一期按 `<500` 文档设计，架构支持扩展到 `2000+`

## 3. 审批规则（一期固定）

## 3.1 报销审批（固定三级流程，不支持自定义）
- 一级审批（部门一审）：部门内部复核/统计/验收 + 部门负责人签审
- 二级审批（财务二审）：会计审批 + 财务总监审批
- 三级审批（公司领导三审）：总经理审批（当前统一口径）
- 三级审批必须完整齐备，缺一不可，不得越级审批
- 日常报销流程（固定顺序）：
  报销人提交 -> 部门经理审批 -> 会计审批 -> 财务总监审批 -> 总经理审批 -> 出纳挂款 -> 会计复核 -> 结束
- 其他付款流程（制度基线）：
  经办人提交 -> 部门经理审批 -> 总经理审批 -> 财务总监审批 -> 出纳挂款 -> 会计复核 -> 结束
- 本期实现边界：系统内实现“日常/差旅报销”全流程；“其他付款申请”流程在PRD保留制度定义，Agent入口放二期
- 特殊情况：允许邮件说明及批复，并抄送相关人员留痕

## 3.2 请假审批（基础版）
- 不做工龄自动计算
- 不做复杂假种规则
- 仅校验“剩余额度”（由后台手动维护）
- 默认流程：主管审批
- 可选扩展：必要时增加二级审批（暂不开放给业务自行配置）

## 3.3 公章审批（二期规则冻结）
- 必填材料清单（附件上传）
- 用途必须命中白名单（合同/证明/授权等）
- 审批流程：主管 -> 行政（最终保管人）

## 4. 状态机（一期必须实现）

## 4.1 请假单状态机
- `draft`：草稿（会话中未提交）
- `submitted`：已提交
- `manager_approved`：主管已通过
- `second_level_pending`：二级待审（如触发）
- `approved`：审批通过
- `rejected`：审批拒绝
- `withdrawn`：员工撤回
- `closed`：流程关闭

流转约束：
- `submitted -> manager_approved/rejected/withdrawn`
- `manager_approved -> second_level_pending/approved`
- `second_level_pending -> approved/rejected`
- `approved/rejected/withdrawn -> closed`

## 4.2 报销单状态机
- `draft`
- `submitted`
- `manager_pending`
- `manager_approved`
- `accountant_pending`
- `accountant_approved`
- `finance_director_pending`
- `finance_director_approved`
- `gm_pending`
- `gm_approved`
- `cashier_pending`
- `cashier_paid`
- `accountant_review_pending`
- `accountant_reviewed`
- `approved`
- `rejected`
- `withdrawn`
- `closed`

流转约束：
- `submitted -> manager_pending/rejected/withdrawn`
- `manager_pending -> manager_approved/rejected`
- `manager_approved -> accountant_pending`
- `accountant_pending -> accountant_approved/rejected`
- `accountant_approved -> finance_director_pending`
- `finance_director_pending -> finance_director_approved/rejected`
- `finance_director_approved -> gm_pending`
- `gm_pending -> gm_approved/rejected`
- `gm_approved -> cashier_pending`
- `cashier_pending -> cashier_paid/rejected`
- `cashier_paid -> accountant_review_pending`
- `accountant_review_pending -> accountant_reviewed/rejected`
- `accountant_reviewed -> approved`
- `approved/rejected/withdrawn -> closed`

## 5. 验收指标（可测试）

## 5.1 功能验收
- 请假：提交后 100% 生成审批单号并可查询状态
- 报销：审批与执行链路命中率 100%，节点顺序必须为“部门经理 -> 会计 -> 财务总监 -> 总经理 -> 出纳挂款 -> 会计复核”
- 知识库：回答必须附来源文档名与版本日期，命中率（Top-3含正确文档）>= 90%

## 5.2 性能验收（工作时段）
- 普通对话：`p95 < 5s`
- 知识库检索问答：`p95 < 8s`
- 涉及钉钉审批创建：`p95 < 10s`

## 5.3 稳定性验收
- 审批回调去重成功率 100%（同一回调不重复入库）
- 外部API失败重试：3次指数退避后进入人工处理队列
- 邮件通知失败：记录失败原因并支持后台重放

## 6. 异常处理与幂等（一期必须）

- 钉钉回调：使用 `event_id + process_instance_id` 幂等键
- 审批创建：客户端重试时以业务单号幂等创建，防止重复审批单
- OCR失败：允许手动填单并标记 `ocr_failed`
- 知识库低置信度：返回“低置信度”提示并展示原文入口
- 外部服务故障：前端提示“稍后重试”，后端记录错误码与重试轨迹

## 7. 权限模型（MVP版）

- 员工：仅访问本人请假/报销数据
- 部门经理：仅对本部门支出有一审权限
- 会计：执行报销会计审批与出纳后复核
- 财务总监：所有支出二审必经
- 出纳：仅执行挂款节点
- 总经理：所有支出三审必经（当前一期固定口径）
- 分公司负责人：按制度保留审批权限（本期不作为系统必经节点）
- 管理员：系统配置、文档管理、审计日志

## 8. 25天交付里程碑（MVP重排）

- D1-D3：基础框架 + 钉钉SSO + 权限骨架 + Orchestrator
- D4-D8：请假审批全流程 + 状态机 + 通知
- D9-D13：报销审批 + 固定三级审批 + OCR + 状态机
- D14-D18：知识库上传/切片/检索/RAG引用
- D19-D21：异常处理、幂等、审计日志
- D22-D24：联调、压测、UAT
- D25：上线发布与回滚预案演练

## 9. 仍需最终确认（仅1项）

- 当前口径已确认：公司领导三审节点暂定统一走总经理。若后续恢复“分公司负责人”场景，再在二期引入组织路由配置。
