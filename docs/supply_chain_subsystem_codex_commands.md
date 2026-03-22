# 供应链跟进子系统：Codex 命令说明（需求概要版）

> 目标：连接供应方、甲方采购、原物料、物流、品检、生产全流程；提供审批流转、到货预判、财务对接、可视化前端展示能力。

## 1) 数据库设计（订单 / 库存 / 金额）

### 1.1 核心表（建议）

#### A. 主数据
- `suppliers`（供应商）
  - `id` PK
  - `supplier_code` 唯一编码
  - `name`
  - `contact_info` JSON
  - `rating` 供应评级
  - `status`（active/inactive）
- `materials`（原物料）
  - `id` PK
  - `material_code` 唯一编码
  - `material_type`（用于“同类型订单”分析）
  - `spec`
  - `unit`
  - `safety_stock`

#### B. 业务单据
- `purchase_orders`（采购订单）
  - `id` PK
  - `po_no` 唯一单号
  - `supplier_id` FK
  - `buyer_org_id`
  - `currency`
  - `total_amount`
  - `tax_amount`
  - `status`（draft/approving/approved/in_transit/arrived/closed）
  - `expected_arrival_date`（系统预判）
  - `committed_arrival_date`（供应商承诺）
  - `created_at/updated_at`
- `purchase_order_items`（采购订单行）
  - `id` PK
  - `po_id` FK
  - `material_id` FK
  - `qty`
  - `unit_price`
  - `line_amount`

#### C. 履约与物流
- `shipments`（发运单）
  - `id` PK
  - `shipment_no`
  - `po_id` FK
  - `logistics_provider`
  - `tracking_no`
  - `departed_at`
  - `eta`
  - `arrived_at`
  - `status`（booked/in_transit/arrived/delayed）
- `shipment_events`（物流轨迹事件）
  - `id` PK
  - `shipment_id` FK
  - `event_time`
  - `event_type`
  - `location`
  - `remark`

#### D. 品检与入库
- `qc_inspections`（品检）
  - `id` PK
  - `po_item_id` FK
  - `inspection_result`（pending/pass/fail/conditional_pass）
  - `defect_rate`
  - `inspector`
  - `inspected_at`
- `inventory_transactions`（库存流水）
  - `id` PK
  - `material_id` FK
  - `txn_type`（inbound/outbound/adjustment/reserved）
  - `qty`
  - `warehouse_id`
  - `biz_ref_type`（po/shipment/production）
  - `biz_ref_id`
  - `txn_time`
- `inventory_balances`（库存余额，日结或实时）
  - `id` PK
  - `material_id`
  - `warehouse_id`
  - `on_hand`
  - `reserved`
  - `available`

#### E. 审批流
- `workflow_instances`（流程实例）
  - `id` PK
  - `biz_type`（po/shipment/qc/payment）
  - `biz_id`
  - `status`（running/approved/rejected/withdrawn）
  - `current_node`
- `workflow_tasks`（审批任务）
  - `id` PK
  - `instance_id` FK
  - `node_code`
  - `approver_id`
  - `action`（approve/reject/transfer）
  - `comment`
  - `acted_at`

#### F. 财务对接
- `finance_settlements`（应付结算）
  - `id` PK
  - `po_id` FK
  - `invoice_no`
  - `invoice_amount`
  - `payable_amount`
  - `payment_status`（unpaid/partial/paid）
  - `erp_sync_status`（pending/success/failed）
  - `erp_sync_at`

---

### 1.2 建库命令（Codex 可直接生成）

```bash
# 1) 生成数据库迁移草案
codex "Create SQL migrations for tables: suppliers, materials, purchase_orders, purchase_order_items, shipments, shipment_events, qc_inspections, inventory_transactions, inventory_balances, workflow_instances, workflow_tasks, finance_settlements with PK/FK/indexes."

# 2) 增加关键索引
codex "Add indexes on purchase_orders(status, expected_arrival_date), shipments(status, eta), inventory_balances(material_id, warehouse_id), workflow_tasks(instance_id, approver_id), finance_settlements(erp_sync_status)."

# 3) 增加审计字段
codex "Update all business tables to include created_by, updated_by, created_at, updated_at, deleted_at for soft delete support."
```

---

## 2) 接口设计（对母系统 / 财务系统）

### 2.1 API 分组
- 采购与订单：`/api/v1/po/*`
- 物流跟踪：`/api/v1/logistics/*`
- 品检管理：`/api/v1/qc/*`
- 库存管理：`/api/v1/inventory/*`
- 审批中心：`/api/v1/workflow/*`
- 预判服务：`/api/v1/eta/*`
- 财务同步：`/api/v1/finance/*`
- 导出服务：`/api/v1/export/*`

### 2.2 关键接口示例
- `POST /api/v1/po/create`：创建采购单
- `POST /api/v1/workflow/start`：发起审批
- `POST /api/v1/workflow/action`：审批动作（通过/驳回）
- `POST /api/v1/logistics/event`：写入物流轨迹
- `GET /api/v1/eta/predict?poNo=...`：获取预判到货日期
- `POST /api/v1/finance/push-payable`：推送应付到财务系统
- `GET /api/v1/export/orders?format=json|csv|xlsx`：导出数据

### 2.3 数据输出格式
- JSON：母系统联调与前端展示
- CSV：运营批量分析
- Excel（XLSX）：管理层报表

### 2.4 接口命令（Codex 生成）

```bash
# 1) 生成 OpenAPI 文档
codex "Generate OpenAPI 3.1 spec for supply chain module APIs including PO, workflow, logistics, QC, inventory, ETA prediction, finance integration, and export endpoints."

# 2) 生成后端路由与 DTO
codex "Based on this OpenAPI, generate controller routes, request/response DTOs, validation rules, and error codes."

# 3) 增加财务系统回调与重试
codex "Implement /finance/push-payable with idempotency key, retry policy, and dead-letter logging for failed sync."
```

---

## 3) 前端界面设计（高效简洁，避免重表格）

### 3.1 交互原则
- 首页以“流程态”展示，不以大表格为主。
- 用可视化组件替代：
  - 全链路进度条（供应商确认→采购审批→发运→在途→品检→入库→投产）
  - 泳道流程图（按角色分层：采购/物流/品检/生产）
  - 状态标签（正常、预警、延误、驳回）
  - 风险卡片（即将延误、质检异常、库存告警）

### 3.2 页面建议
- `订单跟进看板`：按单据查看当前节点 + 风险
- `在途监控`：地图/轨迹时间线
- `审批中心`：待办卡片流
- `到货预测`：预测日期、可信度、影响因子

### 3.3 前端命令（Codex 生成）

```bash
# 1) 生成看板页面骨架
codex "Create a frontend dashboard with process timeline, lane-based workflow graph, status badges, and risk alert cards; avoid dense table-first layout."

# 2) 生成状态标签体系
codex "Implement reusable StatusBadge component with states: normal, warning, delayed, rejected, completed."

# 3) 生成流程图组件
codex "Build a workflow visualization component that highlights current node and approval history from workflow_instances and workflow_tasks APIs."
```

---

## 4) 审批流程设计（与数据库联动）

### 4.1 标准审批节点
1. 采购申请提交
2. 部门负责人审批
3. 采购负责人审批
4. 财务预算校验
5. 供应商确认
6. 发运确认
7. 到货品检放行
8. 入库确认
9. 应付结算审批

### 4.2 流程联动规则
- 每次审批动作写入 `workflow_tasks`；
- 实例状态更新 `workflow_instances`；
- 关键节点回写业务单：
  - 采购单状态 `purchase_orders.status`
  - 发运状态 `shipments.status`
  - 品检状态 `qc_inspections.inspection_result`
  - 财务状态 `finance_settlements.payment_status`

### 4.3 审批命令（Codex 生成）

```bash
# 1) 定义可配置流程
codex "Implement configurable approval workflow engine with node definitions, role-based approvers, reject/rollback support, and audit trail."

# 2) 联动业务单状态机
codex "When workflow node changes, update related purchase order/shipment/QC/settlement statuses in a transactional way."

# 3) 待办与通知
codex "Generate APIs and message hooks for pending approvals, escalation reminders, and SLA timeout alerts."
```

---

## 5) 预判到货（基于历史同类型订单）

### 5.1 数据特征建议
- 订单维度：`material_type`, `supplier_id`, `qty`, `season`
- 物流维度：承运商、路线、历史延误率
- 时间维度：下单日、节假日、港口拥堵指数（可选）
- 质量维度：是否出现返工/复检

### 5.2 预测逻辑（分层）
- V1：规则+统计（同类订单中位到货时长）
- V2：回归模型（XGBoost/LightGBM）
- 输出：
  - `predicted_arrival_date`
  - `confidence_score`
  - `risk_reason`（例如“同路线近30天延误率高”）

### 5.3 预判命令（Codex 生成）

```bash
# 1) 生成特征工程脚本
codex "Create ETA feature engineering pipeline from purchase_orders, shipments, shipment_events, and qc_inspections with supplier/material/route/time features."

# 2) 训练与评估
codex "Train an ETA prediction model using historical similar orders and output MAE, RMSE, and feature importance."

# 3) 提供在线预测接口
codex "Implement /api/v1/eta/predict with model inference, confidence score, and top risk factors explanation."
```

---

## 6) 端到端交付建议（执行顺序）

1. 数据库建模与迁移（主数据 + 业务表 + 审批表）
2. OpenAPI 定义 + 后端骨架
3. 审批引擎与状态机联动
4. 前端看板与流程图
5. ETA V1（统计法）上线
6. 财务系统对接 + 导出服务
7. ETA V2（模型化）迭代

---

## 7) 一次性总命令（可直接给 Codex）

```bash
codex "Design and scaffold a supply-chain follow-up subsystem including: \
(1) database schema for orders/inventory/amounts/workflow/finance settlement; \
(2) OpenAPI endpoints for PO, logistics, QC, inventory, workflow, ETA, finance sync, export (JSON/CSV/XLSX); \
(3) frontend dashboard with process visualization (timeline/workflow graph/status badges/risk cards, not table-first); \
(4) configurable approval workflow linked to transactional status updates; \
(5) ETA prediction based on historical similar orders with confidence and risk reasons; \
(6) initial test cases and seed data." 
```
