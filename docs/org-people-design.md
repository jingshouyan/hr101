# 人员与组织模型设计

> 覆盖场景：集团内兼职、多主体用工、矩阵式项目组织

---

## 一、核心概念与关系

### 当前模型的问题

```
当前：employee ──→ emp_dept ──→ department
      ↑             ↑
    一个人       部门归属 + 汇报线

      ❌ 没有法人实体概念 → 集团内跨公司兼职无法表达
      ❌ 用工主体与发薪主体混在一起 → 劳务派遣/外包无法区分
      ❌ 没有项目组织 → 矩阵式汇报无法表达
      ❌ dept_path 基于单棵树 → 多棵组织树无法表达
```

### 重新设计后的模型

```
                    ┌──────────────────┐
                    │  legal_entity    │  ← 法人实体（法律主体）
                    │  (集团/子公司)    │
                    └────────┬─────────┘
                            │ 1:N
                    ┌───────┴─────────┐
                    │  department     │  ← 部门（归属于法人实体）
                    │  (组织树)        │
                    └───────┬─────────┘
                            │
          ┌─────────────────┼─────────────────┐
          │                 │                 │
          ▼                 ▼                 ▼
  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
  │ employment   │  │ employment   │  │ employment   │  ← 用工关系
  │ (法人A 全职)  │  │ (法人B 董事)  │  │ (法人C 顾问)  │    一人可以多份
  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘
         │                 │                 │
         ▼                 ▼                 ▼
  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
  │ emp_position │  │ emp_position │  │ emp_position │  ← 任职（部门+岗位）
  │ (技术部 总监) │  │ (无)         │  │ (顾问)       │    一份用工可有多段任职
  └──────────────┘  └──────────────┘  └──────────────┘

  ┌──────────────┐
  │ project      │  ← 项目（跨部门临时组织）
  │ (XX项目)     │
  └──────┬───────┘
         │
         ▼
  ┌──────────────┐
  │ emp_project  │  ← 人员-项目分配（矩阵管理）
  │ (张三 PM 50%)│
  └──────────────┘
```

---

## 二、核心表设计

### employee（人员 / 人）

> **"人"是系统的最顶层实体，独立于任何用工关系存在。**
> 一个人即使当前没有 employment（如离职状态、候选人），仍然在系统中有记录。
> 认证（登录）是"人"级别的 —— 输入账号密码的是这个人，登录后再选择以哪份用工关系操作。

```sql
CREATE TABLE employee (
    id              BIGSERIAL    PRIMARY KEY,
    name            VARCHAR(50)  NOT NULL,
    phone           VARCHAR(20),
    email           VARCHAR(100),
    password_hash   VARCHAR(255) NOT NULL,       -- 登录密码（BCrypt）
    status          VARCHAR(20)  NOT NULL DEFAULT 'ACTIVE',  -- ACTIVE / INACTIVE / LOCKED
    source          VARCHAR(20)  DEFAULT 'IMPORT',  -- IMPORT / REGISTER / RECRUIT
    created_at      TIMESTAMPTZ  NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ  NOT NULL DEFAULT now()
);

CREATE UNIQUE INDEX idx_emp_phone ON employee(phone) WHERE phone IS NOT NULL;
CREATE UNIQUE INDEX idx_emp_email ON employee(email) WHERE email IS NOT NULL;
```

> `employee` 就是"人"，不管他当前有没有 employment、在哪个法人实体。
> 一个人离职后，`employee` 记录保留，`employment` 标记为 TERMINATED。
> 未来如果这个人重新入职，只需要新建一条 employment，不需要重新建人。

### legal_entity（法人实体 / 用工主体）

> 集团下的每个子公司、分公司都是一个独立的 legal_entity。
> 每个实体有自己的组织树、发薪体系、社保登记。

```sql
CREATE TABLE legal_entity (
    id              BIGSERIAL    PRIMARY KEY,
    code            VARCHAR(50)  NOT NULL UNIQUE,  -- GROUP_A, SUBSIDIARY_B
    short_name      VARCHAR(100) NOT NULL,          -- 集团总部
    full_name       VARCHAR(500) NOT NULL,          -- XX集团有限公司
    tax_id          VARCHAR(50),                    -- 统一社会信用代码
    social_ins_no   VARCHAR(50),                    -- 社保登记号
    is_group        BOOLEAN      NOT NULL DEFAULT FALSE,  -- 是否为集团总部
    status          VARCHAR(20)  NOT NULL DEFAULT 'ACTIVE',
    created_at      TIMESTAMPTZ  NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ  NOT NULL DEFAULT now()
);
```

### department（部门）

> 每棵组织树从属于一个 legal_entity。不同法人实体的部门路径不互通。

```sql
CREATE TABLE department (
    id              BIGSERIAL    PRIMARY KEY,
    legal_entity_id BIGINT       NOT NULL REFERENCES legal_entity(id),
    parent_id       BIGINT       REFERENCES department(id),
    name            VARCHAR(100) NOT NULL,
    path            VARCHAR(500) NOT NULL,    -- 如 "GROUP_A/1/10/20/"
    level           SMALLINT     NOT NULL,
    dept_type       VARCHAR(20)  NOT NULL DEFAULT 'DEPT',  -- DEPT / PROJECT / TEAM
    cost_center     VARCHAR(50),                            -- 成本中心编码
    head_count      INT,                                    -- 编制数
    status          VARCHAR(20)  NOT NULL DEFAULT 'ACTIVE',
    created_at      TIMESTAMPTZ  NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ  NOT NULL DEFAULT now()
);

CREATE INDEX idx_dept_entity ON department(legal_entity_id);
CREATE INDEX idx_dept_path   ON department(path);
```

> `path` 包含 legal_entity 前缀，用于跨实体的数据隔离：
> - 集团总部的路径：`GROUP_A/1/10/`
> - 子公司B的路径：`SUBSIDIARY_B/1/20/`
> - 这两个部门的 ID 可能都是 10，但 `path` 前缀不同，权限过滤时天然隔离

### employment（用工关系）

> 这是最核心的新表 —— 连接"人"和"法人实体"。
> 一个人可以有多份 employment（比如在集团全职，同时在子公司任董事）。

```sql
CREATE TABLE employment (
    id              BIGSERIAL    PRIMARY KEY,
    emp_id          BIGINT       NOT NULL REFERENCES employee(id),
    legal_entity_id BIGINT       NOT NULL REFERENCES legal_entity(id),
    employee_no     VARCHAR(50),                    -- 在该法人下的工号
    emp_type        VARCHAR(20)  NOT NULL,           -- REGULAR / DISPATCH / OUTSOURCE / INTERN / ADVISOR
    contract_type   VARCHAR(20)  NOT NULL,           -- PERMANENT / FIXED_TERM / PROJECT / PART_TIME
    hire_date       DATE         NOT NULL,
    termination_date DATE,
    status          VARCHAR(20)  NOT NULL DEFAULT 'ACTIVE',  -- ACTIVE / TERMINATED / SUSPENDED
    is_primary      BOOLEAN      NOT NULL DEFAULT FALSE,     -- 是否主用工关系
    created_at      TIMESTAMPTZ  NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ  NOT NULL DEFAULT now(),
    deleted_at      TIMESTAMPTZ,
    UNIQUE (emp_id, legal_entity_id)
);
```

### 多主体用工的处理

```
employment 只解决"人和法人之间有没有用工关系"。
多主体（发薪与用工分离）通过额外字段解决：
```

```sql
-- 在 employment 中扩展多主体字段
ALTER TABLE employment ADD COLUMN payroll_entity_id   BIGINT REFERENCES legal_entity(id);  -- 发薪主体
ALTER TABLE employment ADD COLUMN social_ins_entity_id BIGINT REFERENCES legal_entity(id);  -- 社保缴纳主体
ALTER TABLE employment ADD COLUMN contract_entity_id   BIGINT REFERENCES legal_entity(id);  -- 合同签署主体
```

**典型场景：**

```yaml
场景：劳务派遣员工
  用工主体：        甲方公司（实际工作地）
  发薪主体：        派遣公司（乙方）
  社保缴纳主体：    派遣公司
  合同签署主体：    派遣公司
  employment：
    legal_entity_id = 甲方公司      ← 在谁的系统里
    payroll_entity_id = 派遣公司    ← 谁发工资
    social_ins_entity_id = 派遣公司
    contract_entity_id = 派遣公司
    emp_type = DISPATCH

场景：集团内部借调
  用工主体：        子公司B（借调部门）
  发薪主体：        集团总部（原单位）
  社保缴纳主体：    集团总部
  employment：
    legal_entity_id = 子公司B       ← 在子公司的系统里
    payroll_entity_id = 集团总部    ← 集团总部发工资
    contract_entity_id = 集团总部   ← 合同还是和集团签的
```

### emp_position（任职）

> 替代原来的 `emp_dept` 表。一份 employment 可以有 0-N 个 position。
> 例如：在集团总部任职"技术总监"（实线），同时在子公司B任职"技术顾问"（虚线）。

```sql
CREATE TABLE emp_position (
    id              BIGSERIAL    PRIMARY KEY,
    employment_id   BIGINT       NOT NULL REFERENCES employment(id),
    dept_id         BIGINT       NOT NULL REFERENCES department(id),
    dept_path       VARCHAR(500) NOT NULL,          -- 冗余，权限过滤用
    position_name   VARCHAR(100),                    -- 岗位名称，如"技术总监"
    job_level       VARCHAR(20),                     -- 职级，如 P8 / M3
    report_to_emp_id BIGINT      REFERENCES employee(id),  -- 汇报对象（人）
    report_type     VARCHAR(20)  NOT NULL DEFAULT 'SOLID',  -- SOLID(实线) / DOTTED(虚线)
    is_primary      BOOLEAN      NOT NULL DEFAULT FALSE,    -- 是否主岗位
    started_at      DATE         NOT NULL,
    left_at         DATE,
    created_at      TIMESTAMPTZ  NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ  NOT NULL DEFAULT now()
);

CREATE INDEX idx_pos_employment ON emp_position(employment_id);
CREATE INDEX idx_pos_dept       ON emp_position(dept_id);
CREATE INDEX idx_pos_report_to  ON emp_position(report_to_emp_id);
```

**`report_type` = 矩阵支持的关键：**

```
一个人可以有多个 position，每个 position 指定汇报类型：

张三在集团总部的任职：
  position 1: 技术部 → 总监 → 汇报给 CTO（实线 SOLID）
  position 2: 架构部 → 架构师 → 汇报给首席架构师（虚线 DOTTED）
  position 3: XX项目 → 技术负责人 → 汇报给项目经理（虚线 DOTTED）

实线（SOLID）：管绩效考评、晋升、薪资调整
虚线（DOTTED）：管日常工作、任务分配
```

### project（项目组织）

> 项目是临时性组织，生命周期有限，也挂靠在某个 legal_entity 下。

```sql
CREATE TABLE project (
    id              BIGSERIAL    PRIMARY KEY,
    code            VARCHAR(50)  NOT NULL UNIQUE,
    name            VARCHAR(200) NOT NULL,
    legal_entity_id BIGINT       NOT NULL REFERENCES legal_entity(id),
    dept_id         BIGINT       REFERENCES department(id),    -- 挂靠部门
    manager_emp_id  BIGINT       REFERENCES employee(id),      -- 项目经理
    start_date      DATE         NOT NULL,
    end_date        DATE,
    status          VARCHAR(20)  NOT NULL DEFAULT 'ACTIVE',    -- ACTIVE / CLOSED
    budget          DECIMAL(15,2),                              -- 项目预算
    created_at      TIMESTAMPTZ  NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ  NOT NULL DEFAULT now()
);
```

### emp_project（人员-项目分配）

> 矩阵管理的核心：一个人在项目中担任什么角色、投入多少、向谁汇报。

```sql
CREATE TABLE emp_project (
    id              BIGSERIAL    PRIMARY KEY,
    emp_id          BIGINT       NOT NULL REFERENCES employee(id),
    project_id      BIGINT       NOT NULL REFERENCES project(id),
    role_name       VARCHAR(100) NOT NULL,          -- 项目经理 / 开发 / 测试 / 产品
    report_to_emp_id BIGINT      REFERENCES employee(id),  -- 项目内汇报对象
    allocation_pct  SMALLINT     DEFAULT 100,       -- 投入百分比
    start_date      DATE         NOT NULL,
    end_date        DATE,
    status          VARCHAR(20)  NOT NULL DEFAULT 'ACTIVE',
    created_at      TIMESTAMPTZ  NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ  NOT NULL DEFAULT now(),
    UNIQUE (emp_id, project_id)
);
```

---

## 三、跨实体人员管理

> `employee` 是全局的，但各实体的 HR 只能管理自己实体的人。
> 核心矛盾：**避免重复建人 + 数据隔离**。

### 3.1 人员的可见范围

```
集团总部 HR      → 可见全集团 person（集团管控视角）
子公司 HR        → 仅可见本实体有 employment 的 person
                  + 可按身份证号搜索查重
                  + 查重结果不暴露详情
普通员工         → 只能看到自己
```

**数据隔离的 SQL：**

```sql
-- 子公司 HR 查 person 列表 → 只查自己实体的
SELECT e.* FROM employee e
WHERE EXISTS (
    SELECT 1 FROM employment em
    WHERE em.emp_id = e.id
      AND em.legal_entity_id = :current_entity_id
      AND em.status = 'ACTIVE'
);
```

### 3.2 身份证号 —— 全局查重依据

```sql
-- person 表增加身份证号（全局唯一，跨实体去重）
ALTER TABLE employee ADD COLUMN id_card_no VARCHAR(18) UNIQUE;
ALTER TABLE employee ADD COLUMN id_card_hash VARCHAR(64);  -- 哈希，合规场景用
```

> 身份证号是全局去重的核心。不同子公司用身份证号比对是否同一人。
> 隐私合规：可只存哈希，查重时比对哈希值。

### 3.3 子公司 HR 添加人的流程

**场景：子公司B 招聘了一名新员工 "李四"**

```
子公司B HR 打开"新增人员"页面
  ↓
1. 输入身份证号 / 手机号
  ↓
2. 系统全库搜索：
   - 找到匹配 person → "已在系统中（集团总部·张三），
     是否为其创建子公司B的用工关系？"
   - 未找到 → 进入新建 person 页面
  ↓
   新建 person：姓名、手机、身份证号、邮箱
  ↓
3. 创建 employment（自动关联当前 legal_entity_id = 子公司B）
   工号、用工类型、入职日期、合同类型
  ↓
4. 创建 position（部门、岗位、汇报对象）
  ↓
5. 完成 → 李四在子公司B的组织树下可见
```

**接口设计：**

```http
POST /api/employees/check-duplicate
  Authorization: Bearer xxx        ← 必须登录
  Rate-Limit: 100/hour             ← 限频防爬
  { "id_card_no_hash": "abc123..." }
  → { "exists": false }
  → { "exists": true, "masked_name": "张*", "entities": ["集团总部"] }

POST /api/employees  （新建 person + employment + position）
  {
    "person": { "name": "李四", "phone": "138...", "id_card_no": "110101..." },
    "employment": { "emp_type": "REGULAR", "hire_date": "2026-07-01" },
    "position": { "dept_id": 20, "position_name": "开发工程师", "report_to_emp_id": 50 }
  }

POST /api/employees/{id}/employments  （已有 person，新增用工关系）
  {
    "employment": { "emp_type": "PART_TIME", "hire_date": "2026-07-01" },
    "position": { "dept_id": 25, "position_name": "架构师" }
  }
```

### 3.4 集团 HR vs 子公司 HR 的操作范围

| 操作 | 集团 HR | 子公司 HR |
|------|---------|----------|
| 查看本实体人员 | ✅ 全集团 | ✅ 仅本实体 |
| 新增 person | ✅（关联自己的实体） | ✅（关联自己的实体） |
| 编辑 person 基本信息 | ✅ 全局覆盖 | ⚠️ 仅限本实体 employment 的人 |
| 创建 employment | ✅ 可为任何实体创建 | ✅ 仅限自己的实体 |
| 修改 employment/position | ✅ 任何实体 | ✅ 仅限自己的实体 |
| 终止 employment（离职） | ✅ 任何实体 | ✅ 仅限自己的实体 |
| 删除 person | ❌（只能禁用） | ❌（只能禁用） |
| 跨实体查重 | ✅ | ✅（仅返回 masked 信息） |

## 四、典型场景完整流程

### 场景 1：集团内兼职

```
集团总部下有 技术部，子公司B下有 架构部

张三是集团技术部总监（主岗位），同时在子公司B架构部兼职架构师

数据：
  employee: 张三 (id=1)
  legal_entity: 集团总部(id=1), 子公司B(id=2)

  employment 1: emp_id=1, legal_entity_id=1, emp_type=REGULAR, is_primary=TRUE
    └── position: 技术部, 总监, report_to=CTO, report_type=SOLID

  employment 2: emp_id=1, legal_entity_id=2, emp_type=PART_TIME, is_primary=FALSE
    └── position: 架构部, 架构师, report_to=首席架构师, report_type=DOTTED

权限：
  张三登录 → 选择以"集团总部 技术总监"身份操作
    → managed_paths = ["GROUP_A/1/10/"]  // 只看集团总部的技术部树
  张三登录 → 选择以"子公司B 架构师"身份操作
    → managed_paths = ["SUBSIDIARY_B/1/20/"]  // 只看子公司B的架构部
```

### 场景 2：多主体用工

```
李四是派遣员工，在甲方公司工作，但由派遣公司发薪缴社保

数据：
  employee: 李四 (id=2)
  legal_entity: 甲方(id=3), 派遣公司(id=4)

  employment: emp_id=2, legal_entity_id=甲方, emp_type=DISPATCH,
              payroll_entity_id=派遣公司, social_ins_entity_id=派遣公司,
              contract_entity_id=派遣公司
    └── position: 技术部, 开发工程师, report_to=技术经理

查询李四的薪资：
  → 走 payroll_entity_id = 派遣公司 的薪资体系
查询李四的社保：
  → 走 social_ins_entity_id = 派遣公司 的社保账户
查询李四的工作汇报：
  → 走 position.report_to = 甲方技术经理
```

### 场景 3：矩阵式项目组织

```
王五是技术部的高级工程师（实线汇报给技术经理）

同时被调到"核心交易系统项目"担任技术负责人
  → 项目内虚线汇报给项目经理
  → 项目投入 60% 时间（allocation_pct=60）

同时还在"数据中台项目"担任开发
  → 项目内虚线汇报给数据中台技术负责人
  → 项目投入 40% 时间（allocation_pct=40）

数据：
  employment: emp_id=王五, legal_entity_id=集团总部, emp_type=REGULAR

  position 1（实线）: 技术部, 高级工程师, report_to=技术经理, report_type=SOLID

  emp_project 1: 核心交易系统项目, role=技术负责人, report_to=项目经理, allocation=60%
  emp_project 2: 数据中台项目, role=开发, report_to=数据中台技术负责人, allocation=40%

绩效管理：
  年度考评时：
  → 技术经理（实线上级）负责王五的绩效打分、晋升评估
  → 项目经理（虚线上级）提供王五在项目中的表现评价
  → 综合 SOLID + DOTTED 评价得出最终绩效
```

---

## 五、数据权限变化

### dept_path 前缀规则

由于有了多个 legal_entity，组织树不再是单一根，`dept_path` 需要带上实体前缀：

```yaml
部门路径格式: "{legal_entity_code}/{部门路径}"

集团总部 技术部:     "GROUP_A/1/10/"
集团总部 行政部:     "GROUP_A/1/20/"
子公司B 技术部:      "SUBSIDIARY_B/1/10/"   ← 和集团的技术部 path 不同，天然隔离
子公司B 市场部:      "SUBSIDIARY_B/1/30/"
```

### 数据权限过滤

```sql
-- 数据权限过滤时，dept_path LIKE 匹配天然带实体前缀
-- 张三（集团技术总监）查员工：
WHERE (ed.dept_path LIKE 'GROUP_A/1/10/%')
  -- 只看集团技术部及其子部门，看不到子公司B的数据

-- 如果张三还兼任子公司B架构部：
WHERE (ed.dept_path LIKE 'GROUP_A/1/10/%' OR ed.dept_path LIKE 'SUBSIDIARY_B/1/20/%')
```

### JWT 变化

```json
{
  "emp_id": 1001,
  "active_employment_id": 1,       // 当前选择的用工关系
  "primary_dept_id": 10,
  "dept_ids": [10, 25],
  "managed_paths": ["GROUP_A/1/10/", "SUBSIDIARY_B/1/20/"],
  "exp": 300
}
```

---

## 六、员工端 UI 影响

### 身份选择器

多 employment 的员工在系统右上角加一个身份切换：

```
┌──────────────────────────────────────┐
│  [集团总部 · 技术总监 ▼]   张三 ▼    │
│  ┌──────────────────────────┐        │
│  │ ✓ 集团总部 · 技术总监     │        │
│  │   子公司B · 架构师(兼)    │        │
│  │   核心交易项目 · 技术负责  │        │
│  └──────────────────────────┘        │
└──────────────────────────────────────┘
```

切换身份影响：
- 数据权限范围（看哪个实体的数据）
- 审批链（走哪个部门的审批）
- 菜单权限（不同实体可能开放不同模块）

---

## 七、一期 MVP 简化建议

这个模型很全，但一期不需要全部实现：

| 阶段 | 包含 | 不包含 |
|------|------|--------|
| **一期** | `employee` + `employment` + `emp_position` | 多主体发薪、项目矩阵 |
| | 单个 legal_entity（默认公司） | 多个 legal_entity |
| | SOLID 汇报（单个上级） | DOTTED 汇报 |
| | 单部门 | 多部门可选 |
| **二期** | 多个 legal_entity（集团） | 项目矩阵 |
| | 多部门支持 | |
| | `hrbp_dept` 配置 | |
| **三期** | 项目组织 `project` + `emp_project` | |
| | 多主体发薪 `payroll_entity_id` | |
| | 矩阵汇报 `report_type` | |
| | 身份选择器 UI | |

> 但**数据模型建议一期就按这个设计建表**，字段先 nullable，不用就是空的。
> 这样二、三期扩展时不需要改表结构，只是把空字段填上。
