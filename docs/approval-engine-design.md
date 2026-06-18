# 审批引擎设计（自研轻量版）

> **目标**：覆盖 HR 系统 90% 审批场景，避免引入 Camunda/Flowable 等重型 BPMN 引擎。
> **设计原则**：够用就好，不做过深的流程抽象，不引入额外的流程定义语言。

---

## 一、HR 审批场景分析

### 常见审批链

```
请假审批       员工 → 直接主管 → HRBP → 部门负责人（条件：>3天）
加班审批       员工 → 直接主管
报销审批       员工 → 直接主管 → 财务（条件：>1000元）
招聘需求       部门主管 → HRBP → VP → CEO（条件：薪资>预算）
转正审批       员工发起 → 直接主管 → HRBP → 部门负责人
合同续签       HR发起 → 直接主管 → HR负责人
采购申请       申请人 → 部门主管 → 行政 → CEO（条件：金额>5000）
```

### 审批模式归纳

| 模式 | 说明 | HR 示例 |
|------|------|---------|
| **单人审批** | 一个节点仅需一人审批通过 | 主管审批请假 |
| **会签** | 节点需全部审批人通过（一票否决） | 多位总监联合审批 |
| **或签** | 节点中任意一人通过即进入下一节点 | 主管或代理人审批 |
| **条件分支** | 根据表单字段值走向不同分支 | 金额 >5000 → VP 审批 |
| **超时自动处理** | 超时未审批，自动升级或通过 | 催办 + 超时转交上级 |
| **委托审批** | 审批人指定他人代为审批 | 出差期间委托同事 |

---

## 二、核心数据模型

### ER 图（核心表）

```
┌─────────────────┐       ┌─────────────────────┐
│  审批流程定义      │       │  审批节点定义          │
│  approval_flow   │──1:N─→│  approval_node       │
│─────────────────│       │─────────────────────│
│ id               │       │ id                  │
│ name             │       │ flow_id             │
│ code（唯一标识）   │       │ name（节点名）        │
│ status（启用/停用）│       │ type（单人/会签/或签） │
│ initiator_type   │       │ order_index（排序）  │
│ description      │       │ timeout_hours       │
└─────────────────┘       └─────────┬───────────┘
                                    │
                                    │ 1:N
                                    ▼
                          ┌─────────────────────┐
                          │  审批人规则定义        │
                          │  approver_rule       │
                          │─────────────────────│
                          │ id                  │
                          │ node_id             │
                          │ rule_type           │
                          │ rule_config (JSON)  │
                          └─────────────────────┘

┌─────────────────┐       ┌─────────────────────┐
│  审批实例         │──1:N─→│  审批记录            │
│  approval_inst   │       │  approval_record    │
│─────────────────│       │─────────────────────│
│ id               │       │ id                  │
│ flow_id          │       │ instance_id         │
│ business_type    │       │ node_id             │
│ business_id      │       │ operator_id         │
│ title            │       │ action（通过/驳回/退回）│
│ initiator_id     │       │ comment             │
│ form_data (JSON) │       │ created_at          │
│ status           │       └─────────────────────┘
│ created_at       │
│ finished_at      │
└─────────────────┘
```

### 约定

- 所有表使用 `BIGSERIAL` 作为自增主键
- 时间字段统一使用 `TIMESTAMPTZ`（`TIMESTAMP WITH TIME ZONE`）
- `created_at` / `updated_at` 每表必加，`deleted_at` 按需添加（软删除）
- 标志位用 `BOOLEAN` 而非 `SMALLINT`
- JSON 字段用 `JSONB`（PostgreSQL 二进制 JSON，支持索引和高效查询）
- 字段注释通过 `COMMENT ON COLUMN` 定义

### approval_flow（流程定义）

```sql
CREATE TABLE approval_flow (
    id              BIGSERIAL       PRIMARY KEY,
    name            VARCHAR(100)    NOT NULL,
    code            VARCHAR(50)     NOT NULL UNIQUE,
    description     VARCHAR(500),
    status          SMALLINT        NOT NULL DEFAULT 1,   -- 1=启用 0=停用
    initiator_type  VARCHAR(50),                           -- ALL / EMPLOYEE_TYPE / DEPT
    version         INT             NOT NULL DEFAULT 1,
    created_at      TIMESTAMPTZ     NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ     NOT NULL DEFAULT now(),
    deleted_at      TIMESTAMPTZ                              -- 软删除，归档旧流程定义
);

COMMENT ON TABLE  approval_flow        IS '审批流程定义';
COMMENT ON COLUMN approval_flow.name   IS '流程名称，如"请假审批"';
COMMENT ON COLUMN approval_flow.code   IS '流程编码，如 LEAVE_DAILY';
COMMENT ON COLUMN approval_flow.status IS '1=启用 0=停用';
COMMENT ON COLUMN approval_flow.initiator_type IS '发起人限制：ALL=所有人 / EMPLOYEE_TYPE=按员工类型 / DEPT=按部门';
COMMENT ON COLUMN approval_flow.version IS '版本号，每次修改递增';
```

### approval_node（审批节点）

```sql
CREATE TABLE approval_node (
    id              BIGSERIAL       PRIMARY KEY,
    flow_id         BIGINT          NOT NULL REFERENCES approval_flow(id),
    name            VARCHAR(100)    NOT NULL,
    type            VARCHAR(20)     NOT NULL,               -- APPROVE / COUNTERSIGN / OR_SIGN
    order_index     INT             NOT NULL,
    timeout_hours   INT             DEFAULT 48,
    skip_empty      BOOLEAN         NOT NULL DEFAULT FALSE,  -- 审批人为空时自动跳过
    reject_to       VARCHAR(20)     NOT NULL DEFAULT 'prev', -- prev / start / end
    node_config     JSONB,                                   -- 扩展配置
    created_at      TIMESTAMPTZ     NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ     NOT NULL DEFAULT now(),
    deleted_at      TIMESTAMPTZ
);

COMMENT ON TABLE  approval_node         IS '审批节点定义';
COMMENT ON COLUMN approval_node.type    IS 'APPROVE(单人审批) / COUNTERSIGN(会签) / OR_SIGN(或签)';
COMMENT ON COLUMN approval_node.order_index IS '节点顺序，从1开始';
COMMENT ON COLUMN approval_node.timeout_hours IS '超时小时数，超时自动催办';
COMMENT ON COLUMN approval_node.skip_empty IS '当审批人为空时是否自动跳过此节点';
COMMENT ON COLUMN approval_node.reject_to IS '驳回目标：prev=上一节点 start=重新发起 end=终止';
```

`node_config` 示例：
```json
{
  "allowAddCc": true,
  "autoApproval": false,
  "formPermissions": { "view": ["*"], "edit": [] }
}
```

### approver_rule（审批人规则）

```sql
CREATE TABLE approver_rule (
    id              BIGSERIAL       PRIMARY KEY,
    node_id         BIGINT          NOT NULL REFERENCES approval_node(id),
    rule_type       VARCHAR(50)     NOT NULL,
    rule_config     JSONB           NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ     NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ     NOT NULL DEFAULT now(),
    deleted_at      TIMESTAMPTZ
);

COMMENT ON TABLE  approver_rule          IS '审批人规则定义';
COMMENT ON COLUMN approver_rule.rule_type IS '审批人规则类型（见下方表格）';
COMMENT ON COLUMN approver_rule.rule_config IS '规则配置 JSON，不同 rule_type 结构不同';
```

**rule_type 支持的类型：**

| rule_type | 说明 | rule_config 示例 |
|-----------|------|-----------------|
| `REPORT_TO` | 直接上级（最常见） | `{"level": 1}` |
| `DEPT_HEAD` | 部门负责人 | `{}` |
| `HRBP` | 对应 HRBP | `{}` |
| `FIXED_USER` | 固定审批人 | `{"user_ids": [1, 2]}` |
| `ROLE` | 按角色 | `{"role_code": "CEO"}` |
| `REPORT_TO_LEVEL` | 上N级审批 | `{"level": 2}` |
| `SELF` | 发起人自己（通常仅在开始节点） | `{}` |
| `VARIABLE` | 表单字段指定 | `{"field": "approver"}` |
| `DYNAMIC` | 扩展点，代码动态计算 | `{"bean": "customApproverService"}` |

> **核心设计**：审批人规则解析器 (`ApproverResolver`) 根据 `rule_type` + `rule_config` + `form_data` 动态计算出具体审批人列表。每种 `rule_type` 只需实现一个 Resolver，可插拔。

### approval_instance（审批实例）

```sql
CREATE TABLE approval_instance (
    id              BIGSERIAL       PRIMARY KEY,
    flow_id         BIGINT          NOT NULL REFERENCES approval_flow(id),
    business_type   VARCHAR(50)     NOT NULL,
    business_id     BIGINT          NOT NULL,
    title           VARCHAR(200)    NOT NULL,
    initiator_id    BIGINT          NOT NULL,
    form_data       JSONB,
    status          VARCHAR(20)     NOT NULL DEFAULT 'PENDING',
    current_node_id BIGINT          REFERENCES approval_node(id),
    created_at      TIMESTAMPTZ     NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ     NOT NULL DEFAULT now(),
    finished_at     TIMESTAMPTZ,
    deleted_at      TIMESTAMPTZ
);

COMMENT ON TABLE  approval_instance              IS '审批实例（每次审批发起生成一条记录）';
COMMENT ON COLUMN approval_instance.business_type IS '业务类型：LEAVE / OVERTIME / REIMBURSE / CONTRACT / ...';
COMMENT ON COLUMN approval_instance.business_id   IS '业务记录 ID';
COMMENT ON COLUMN approval_instance.title         IS '审批标题，如"张三的请假申请"';
COMMENT ON COLUMN approval_instance.initiator_id  IS '发起人 employee_id';
COMMENT ON COLUMN approval_instance.form_data     IS '表单数据快照（JSONB）';
COMMENT ON COLUMN approval_instance.status        IS 'PENDING / APPROVED / REJECTED / CANCELLED';
COMMENT ON COLUMN approval_instance.current_node_id IS '当前待审批节点（null 表示待发起或已完成）';
COMMENT ON COLUMN approval_instance.finished_at   IS '审批完成时间';
COMMENT ON COLUMN approval_instance.deleted_at    IS '软删除/归档时间';
```

### approval_record（审批记录）

```sql
CREATE TABLE approval_record (
    id              BIGSERIAL       PRIMARY KEY,
    instance_id     BIGINT          NOT NULL REFERENCES approval_instance(id),
    node_id         BIGINT          NOT NULL REFERENCES approval_node(id),
    operator_id     BIGINT          NOT NULL,
    action          VARCHAR(20)     NOT NULL,
    comment         VARCHAR(1000),
    attachments     JSONB,
    created_at      TIMESTAMPTZ     NOT NULL DEFAULT now()
);

COMMENT ON TABLE  approval_record            IS '审批操作记录（变更审计日志，不可修改/删除）';
COMMENT ON COLUMN approval_record.operator_id IS '操作人 employee_id';
COMMENT ON COLUMN approval_record.action      IS 'APPROVE(通过) / REJECT(驳回) / RETURN(退回) / DELEGATE(转交) / ADD_APPROVER(加签)';
COMMENT ON COLUMN approval_record.comment     IS '审批意见';
COMMENT ON COLUMN approval_record.attachments IS '附件列表 JSONB';
```

> `approval_record` 是审计日志，**不设 `deleted_at` 和 `updated_at`**，数据不可变（immutable）。

### 索引

```sql
-- approval_flow
CREATE INDEX idx_flow_code     ON approval_flow(code)    WHERE deleted_at IS NULL;

-- approval_node
CREATE INDEX idx_node_flow_id  ON approval_node(flow_id) WHERE deleted_at IS NULL;
CREATE UNIQUE INDEX uk_node_order ON approval_node(flow_id, order_index) WHERE deleted_at IS NULL;

-- approver_rule
CREATE INDEX idx_rule_node_id  ON approver_rule(node_id) WHERE deleted_at IS NULL;

-- approval_instance（查询最频繁的表）
CREATE INDEX idx_inst_status      ON approval_instance(status);
CREATE INDEX idx_inst_initiator   ON approval_instance(initiator_id);
CREATE INDEX idx_inst_business    ON approval_instance(business_type, business_id);
CREATE INDEX idx_inst_flow_status ON approval_instance(flow_id, status);
CREATE INDEX idx_inst_created     ON approval_instance(created_at DESC);
CREATE INDEX idx_inst_deleted     ON approval_instance(deleted_at) WHERE deleted_at IS NOT NULL;

-- approval_record
CREATE INDEX idx_record_instance  ON approval_record(instance_id);
CREATE INDEX idx_record_operator  ON approval_record(operator_id);
CREATE INDEX idx_record_created   ON approval_record(created_at DESC);
```

---

## 三、核心流程

### 审批生命周期

```
发起申请 ─→ 节点1  ─→ 节点2  ─→ ... ─→ 节点N  ─→ 审批完成
  │          │         │                    │
  │          ├─ 通过   ├─ 通过              ├─ 全部通过 → APPROVED
  │          ├─ 驳回   ├─ 驳回              │
  │          │         │                    │
  │          └─ 退回   └─ 退回              │
  │               ↓          ↓              │
  │           退回到上一节点或重新发起         │
  │                                          │
  └── 申请人撤回 →  CANCELLED                 │
                                             │
                     申请人再次提交 ←─────────┘
```

### 节点流转逻辑（伪代码）

```java
class ApprovalEngine {

    /** 提交审批 */
    Long start(flowCode, businessType, businessId, formData, initiatorId) {
        // 1. 查询流程定义（启用状态）
        // 2. 校验发起人是否有权发起
        // 3. 创建审批实例，状态 PENDING
        // 4. 解析第一个节点的审批人
        // 5. 创建待办任务并发通知
        // 6. 返回 instanceId
    }

    /** 审批操作 */
    ApprovalResult approve(instanceId, operatorId, action, comment) {
        // 1. 校验操作人是否为当前节点审批人
        // 2. 记录审批记录
        // 3. 根据节点类型判断节点是否完成：
        //    - APPROVE: 单人 → 直接完成节点
        //    - COUNTERSIGN: 需要全部审批人通过
        //    - OR_SIGN: 任一审批人通过即可
        // 4. 节点完成 → 推进到下一节点
        // 5. 没有下一节点 → 实例完成，status=APPROVED
        // 6. 回调业务方（onApproved / onRejected）
    }

    /** 驳回 / 退回 */
    ApprovalResult reject(instanceId, operatorId, comment) {
        // 1. 记录驳回记录
        // 2. 根据 node_config.rejectTo 决定退回目标：
        //    - prev: 退回上一节点
        //    - start: 退回发起人重新提交
        //    - end: 直接终止（REJECTED）
        // 3. 清除相关节点的待办
        // 4. 回退节点重新生成待办
    }

    /** 解析审批人 */
    List<Long> resolveApprovers(nodeId, formData, initiatorId) {
        // 遍历当前节点的 approver_rule 列表
        // 按 rule_type 路由到对应的 ApproverResolver
        // 聚合所有规则的去重结果
        // 返回审批人 ID 列表
    }
}
```

### 各节点类型的完成条件

```java
class NodeCompletionStrategy {

    boolean isNodeCompleted(Node node, List<ApprovalRecord> records) {
        if (node.type == APPROVE) {         // 单人审批
            return records.stream().anyMatch(r -> r.action == APPROVE);
        }
        if (node.type == COUNTERSIGN) {     // 会签（一票否决）
            boolean hasReject = records.stream().anyMatch(r -> r.action == REJECT);
            boolean allApproved = resolvedApprovers.stream()
                .allMatch(uid -> records.stream().anyMatch(r -> r.operatorId == uid && r.action == APPROVE));
            if (hasReject) return true;      // 有驳回 → 节点结束，走驳回逻辑
            return allApproved;
        }
        if (node.type == OR_SIGN) {          // 或签
            return records.stream().anyMatch(r -> r.action == APPROVE);
        }
        return false;
    }
}
```

---

## 四、审批人解析器设计

### 解析器接口

```java
public interface ApproverResolver {
    /**
     * @param ruleConfig  规则配置（JSON）
     * @param formData    表单数据
     * @param initiator   发起人
     * @return 审批人ID列表（有序、去重）
     */
    List<Long> resolve(JSONObject ruleConfig, JSONObject formData, Employee initiator);
}
```

### 内置解析器实现

| 解析器 | rule_type | 逻辑 |
|--------|-----------|------|
| `ReportToResolver` | `REPORT_TO` | 查员工表的 `report_to` 字段，一级级向上找 |
| `DeptHeadResolver` | `DEPT_HEAD` | 查部门表 `head_id`，取部门负责人 |
| `HrbpResolver` | `HRBP` | 查员工 HRBP 映射关系 |
| `FixedUserResolver` | `FIXED_USER` | 直接返回配置的 userIds |
| `RoleResolver` | `ROLE` | 查角色-用户映射表 |
| `VariableResolver` | `VARIABLE` | 从 form_data 中取指定字段的值作为审批人 |
| `DynamicResolver` | `DYNAMIC` | Spring Bean 方式扩展，业务方自己实现复杂逻辑 |

### 组织架构查询（关键依赖）

审批人解析严重依赖组织架构数据，需要以下查询能力：

```java
// 查找员工的直接上级
Employee getReportTo(Long employeeId);

// 查找部门负责人
Employee getDeptHead(Long deptId);

// 查找员工的第N级上级
Employee getNthLevelReportTo(Long employeeId, int level);

// 查找员工的 HRBP
Employee getHrbp(Long employeeId);
```

---

## 五、超时与催办

### 定时任务设计

```
┌─────────────────────────────────────────────┐
│                XXL-JOB 调度中心                │
├─────────────────────────────────────────────┤
│  approval-timeout-checker （每5分钟执行一次）  │
│                                              │
│  1. 查询所有 PENDING 状态 & current_node_id 非空 │
│     且 created_at + node.timeout_hours < now  │
│  2. 对超时的待办：                              │
│     a. 发送催办通知（站内信 + 企微/钉钉）        │
│     b. 每超一次，超时计数 +1                    │
│     c. 超时计数 >= max_remind_count → 升级审批 │
│        升级规则：上报当前节点的上级              │
└─────────────────────────────────────────────┘
```

### 催办策略

| 策略 | 实现 |
|------|------|
| 首次催办 | 超时后立即发送通知 |
| 周期性催办 | 每超时 N 小时催办一次 |
| 升级审批 | 超时超过阈值后，自动加签给上级 |

---

## 六、业务方集成方式

### 业务方只需要三步

```java
// 1. 定义审批流程（管理后台配置，或代码初始化）
approvalFlowService.createFlow("LEAVE_DAILY", "日常请假", ...);
approvalFlowService.addNode("LEAVE_DAILY", "主管审批", APPROVE, 1, ...);
approvalFlowService.addApproverRule(nodeId, REPORT_TO, "{}");
approvalFlowService.addNode("LEAVE_DAILY", "HRBP审批", APPROVE, 2, ...);
approvalFlowService.addApproverRule(nodeId, HRBP, "{}");

// 2. 提交审批时发起
LeaveRecord leave = leaveService.create(leaveForm);
Long instanceId = approvalEngine.start(
    "LEAVE_DAILY",      // 流程编码
    "LEAVE",            // 业务类型
    leave.getId(),      // 业务ID
    leaveForm,          // 表单数据（作为快照）
    employeeId          // 发起人
);

// 3. 监听审批完成回调（事件监听方式）
@Component
class LeaveApprovalListener {
    @EventListener
    void onApproved(ApprovalCompletedEvent event) {
        if (!"LEAVE".equals(event.getBusinessType())) return;
        leaveService.updateStatus(event.getBusinessId(), "APPROVED");
    }
    @EventListener
    void onRejected(ApprovalRejectedEvent event) {
        if (!"LEAVE".equals(event.getBusinessType())) return;
        leaveService.updateStatus(event.getBusinessId(), "REJECTED");
    }
}
```

### 审批事件表

| 事件 | 触发时机 | 业务方用途 |
|------|---------|-----------|
| `ApprovalStartedEvent` | 审批发起后 | 更新业务状态为"审批中" |
| `ApprovalNodeCompletedEvent` | 每节点完成 | 进度跟踪 |
| `ApprovalCompletedEvent` | 全部通过 | 执行业务（如更新请假状态） |
| `ApprovalRejectedEvent` | 驳回/退回 | 更新业务状态为"已驳回" |
| `ApprovalCancelledEvent` | 发起人撤回 | 更新业务状态为"已取消" |

---

## 七、管理后台功能

### 流程定义管理

- **流程列表**：展示所有流程定义，支持启用/停用
- **流程设计器**：可视化的节点配置界面（拖拽排序）
  - 节点基本设置（名称、类型、超时时间）
  - 审批人规则配置（下拉选择规则类型 + 参数填写）
  - 驳回策略设置

### 审批实例管理

- **我的待办**：待我审批的列表
- **我发起的**：我发起的审批历史
- **审批中心（HR 视角）**：全局审批监控，强制干预权限
  - 查看进行中实例
  - 审批人转交（当审批人离职时）
  - 审批终止

### 审批日志

- 每个审批实例的完整轨迹图（时间线 + 节点图）
- 操作人、操作时间、审批意见、附件

---

## 八、设计要点总结

| 关注点 | 方案 |
|--------|------|
| **流程定义** | 数据库配置，非 BPMN 文件，前端操作即可 |
| **审批人解析** | 插件式 Resolver，每种规则一个实现类，可扩展 |
| **节点类型** | 单人 / 会签 / 或签，三种覆盖 HR 全部场景 |
| **驳回策略** | 可配置：退回上一节点 / 退回发起人 / 直接终止 |
| **超时催办** | XXL-JOB 定时扫描 + 通知推送 + 自动升级 |
| **业务解耦** | Spring Event 异步通知，业务方监听事件即可 |
| **前端集成** | 提供待办列表 API、审批操作 API，前端 Vue 组件封装 |
| **扩展性** | DynamicResolver 支持代码扩展，不修改引擎核心 |

---

## 九、一期 MVP 范围建议

> 第一期实现 **精简版**，覆盖核心场景即可，不要一次性做完。

**一期必做：**
- [ ] 流程定义 CRUD（代码初始化 + 管理后台查看）
- [ ] 审批节点定义（单人审批 + 会签 + 或签）
- [ ] 审批人规则（REPORT_TO / DEPT_HEAD / FIXED_USER / ROLE 四种即可）
- [ ] 审批实例管理（发起、审批通过、驳回、撤回）
- [ ] 待办列表 API
- [ ] Spring Event 回调
- [ ] 超时催办（简单的 XXL-JOB 定时扫描）

**二期再做：**
- [ ] 可视化流程设计器（拖拽 UI）
- [ ] 动态审批人（VARIABLE / DYNAMIC 类型）
- [ ] 委托审批
- [ ] 超时自动升级
- [ ] 审批转交（管理员干预）
- [ ] 审批轨迹图

---

*本文档由 Reasonix 生成。*
