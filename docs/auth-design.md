# 认证与权限设计（JWT + Spring Security）

> **技术选型**：Spring Security + JWT (jjwt) + RBAC
> **目标**：无状态认证、菜单/按钮级权限控制、数据权限隔离

---

## 一、整体架构

```
┌──────────┐       ┌──────────────────┐       ┌──────────────┐
│  前端     │──1──→│  登录接口          │──3──→  │  用户服务      │
│  Vue 3   │       │  POST /auth/login │       │  Employee    │
│          │←─2───│  返回 access_token │       └──────────────┘
│          │       │  + refresh_token  │             │
│          │       └──────────────────┘             │
│          │                                         ▼
│          │       ┌──────────────────┐       ┌──────────────┐
│          │──4──→│  业务接口          │──5──→│  JWT 过滤器    │
│          │       │  Header:          │       │  解析 token   │
│          │       │  Authorization:   │       │  提取用户信息  │
│          │       │  Bearer xxx       │       │  设置 Security-│
│          │       └──────────────────┘       │  Context      │
│          │                                   └──────┬───────┘
│          │                                           ▼
│          │                                   ┌──────────────┐
│          │                                   │  权限校验      │
│          │                                   │  @PreAuthorize│
│          │                                   │  + 数据权限   │
│          │                                   └──────────────┘
```

## 二、JWT 令牌设计

### 双令牌机制

| 令牌 | 存储位置 | 有效期 | 用途 |
|------|---------|--------|------|
| **access_token** | 前端内存 / 请求头 | **5-15 分钟** | 调用业务接口，不放权限列表 |
| **refresh_token** | httpOnly Cookie | 7 天 | 静默续签 access_token，续签时从 DB 重查最新权限 |

### Token 载荷

```json
{
  "sub": "1001",
  "emp_id": 1001,
  "name": "张三",
  "dept_id": 20,
  "type": "access",
  "iat": 1718000000,
  "exp": 1718000500     // 5 分钟后过期
}
```

> **设计要点**：
> - **access_token 不放角色和权限列表**，只放用户基本标识
> - 每次请求的权限校验通过 `@PreAuthorize` 实时查 DB（见下文）
> - `dept_path` 等数据权限字段也不放，数据权限通过 MyBatis-Plus 拦截器实时过滤
> - token 不存敏感信息（手机号、身份证等）
> - 短 TTL（5-15 分钟）减少泄漏风险

### 刷新流程

```
access_token 过期 → API 返回 401
  └→ 前端调用 POST /auth/refresh（携带 refresh_token Cookie）
      ├→ 成功：从 DB 重新查询权限 → 签发新 access_token
      └→ 失败：跳转登录页
```

> 刷新时后端从 `emp_role` + `role_permission` 重新加载权限，
> 所以新签发的 access_token 总是携带最新的权限。
> 权限变更最多延迟一个 access_token 的生命周期（5-15 分钟）。

## 三、Spring Security 配置

### 安全过滤器链

```
/public/**           → 无需认证（登录、注册、验证码等）
/auth/**             → 无需认证（登录、刷新）
/swagger-ui/**       → 开发环境可访问
/api/**              → 需要有效 JWT
/api/admin/**        → 需要 ADMIN 角色
```

### 核心组件

```java
// 1. JWT 认证过滤器
public class JwtAuthenticationFilter extends OncePerRequestFilter {
    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain chain) {
        // 从 Header 取 token
        // 解析 & 验证签名
        // 提取用户信息 → 设置 SecurityContext
        // chain.doFilter()
    }
}

// 2. 自定义认证令牌
public class JwtAuthenticationToken extends AbstractAuthenticationToken {
    private final EmployeePrincipal principal;
    // 已认证状态 = true
}

// 3. SecurityConfig
@Configuration
@EnableMethodSecurity  // 开启 @PreAuthorize
public class SecurityConfig {
    @Bean
    SecurityFilterChain filterChain(HttpSecurity http) {
        http
            .csrf(csrf -> csrf.disable())           // REST API 不需要
            .sessionManagement(sm -> sm.sessionCreationPolicy(STATELESS))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/public/**").permitAll()
                .requestMatchers("/auth/**").permitAll()
                .anyRequest().authenticated()
            )
            .addFilterBefore(jwtFilter, UsernamePasswordAuthenticationFilter.class);
        return http.build();
    }
}
```

## 四、RBAC 权限模型

### 设计思路

HR 系统的权限有几个特点：
- **角色极少且稳定**：总共 6 个角色，新增角色是低频事件
- **权限编码是契约**：如 `employee:read`、`salary:calculate`，在代码中直接引用，不是用户动态创建的数据
- **权限变更频率低**：几个月调一次，不需要运行时弹性的权限表

因此：**权限编码不进数据库，放在 Java 常量中。角色和权限的映射关系合并到 role 表的一个 TEXT[] 字段。**

### 数据模型

```
┌──────────┐     ┌────────────────┐     ┌──────────────────────┐
│   用户    │────→│  用户-角色关联  │←────│   角色（含权限列表）   │
│ employee │     │  emp_role      │     │   role               │
└──────────┘     └────────────────┘     │   permissions TEXT[] │
                                        └──────────────────────┘
```

```sql
-- 角色（权限合并为 TEXT[] 数组，数据范围作为角色固有属性）
CREATE TABLE role (
    id            BIGSERIAL    PRIMARY KEY,
    code          VARCHAR(50)  NOT NULL UNIQUE,  -- HR_MANAGER
    name          VARCHAR(100) NOT NULL,
    description   VARCHAR(500),
    permissions   TEXT[]       NOT NULL DEFAULT '{}',  -- 如 {"employee:read","employee:write"}
    data_scope    VARCHAR(20)  NOT NULL DEFAULT 'SELF' -- ALL / DEPT_TREE / DEPT / SELF
);

CREATE INDEX idx_role_perms ON role USING GIN (permissions);

-- 用户-角色
CREATE TABLE emp_role (
    emp_id      BIGINT NOT NULL REFERENCES employee(id),
    role_id     BIGINT NOT NULL REFERENCES role(id),
    PRIMARY KEY (emp_id, role_id)
);
```

**权限查询：**

```sql
-- 查某用户的所有权限
SELECT DISTINCT unnest(r.permissions)
FROM emp_role er
JOIN role r ON r.id = er.role_id
WHERE er.emp_id = 1001;

-- 结果：{"employee:read", "employee:write", "salary:read", ...}
```

**GIN 索引支持反向查询：**
```sql
-- 哪些角色包含 'employee:read' 权限？
SELECT code FROM role WHERE permissions @> ARRAY['employee:read'];
```

### 权限编码（Java 常量）

> 所有权限编码集中管理，不落数据库。

```java
public final class Permissions {

    // ── 员工模块 ──
    public static final String EMPLOYEE_READ   = "employee:read";
    public static final String EMPLOYEE_WRITE  = "employee:write";
    public static final String EMPLOYEE_DELETE = "employee:delete";
    public static final String EMPLOYEE_EXPORT = "employee:export";

    // ── 考勤模块 ──
    public static final String ATTENDANCE_READ    = "attendance:read";
    public static final String ATTENDANCE_WRITE   = "attendance:write";
    public static final String ATTENDANCE_APPROVE = "attendance:approve";

    // ── 薪资模块 ──
    public static final String SALARY_READ      = "salary:read";
    public static final String SALARY_CALCULATE = "salary:calculate";
    public static final String SALARY_PAY       = "salary:pay";

    // ── 审批模块 ──
    public static final String APPROVAL_VIEW      = "approval:view";
    public static final String APPROVAL_APPROVE   = "approval:approve";
    public static final String APPROVAL_INTERVENE = "approval:intervene";

    // ── 组织架构 ──
    public static final String ORG_READ  = "org:read";
    public static final String ORG_WRITE = "org:write";

    // ── 系统管理 ──
    public static final String SYS_USER   = "sys:user";
    public static final String SYS_ROLE   = "sys:role";
    public static final String SYS_CONFIG = "sys:config";
}
```

### 角色与权限的绑定（初始化代码）

> role 表数据通过代码初始化，建表时插入基础数据，后续通过 Flyway/Liquibase 管理变更。

```java
@Component
public class RolePermissionInitializer implements CommandLineRunner {

    private final RoleMapper roleMapper;

    @Override
    public void run(String... args) {
        // 插入角色（幂等：存在则跳过）
        insertRole("ADMIN",       "系统管理员",     Permissions.ALL);
        insertRole("HR_MANAGER",  "HR 管理员",      Permissions.HR_FULL);
        insertRole("HR_OPERATOR", "HR 专员",        Permissions.HR_OPERATE);
        insertRole("HR_PAYROLL",  "HR 薪资专员",    Permissions.HR_PAYROLL);
        insertRole("DEPT_HEAD",   "部门主管",       Permissions.DEPT_HEAD);
        insertRole("EMPLOYEE",    "普通员工",       Permissions.EMPLOYEE);
    }

    private void insertRole(String code, String name, String[] permissions) {
        if (roleMapper.selectByCode(code) == null) {
            roleMapper.insert(new Role().setCode(code).setName(name).setPermissions(permissions));
        }
    }
}
```

```java
// 每个角色的权限集合 + 数据范围（集中在 Permissions 类中维护）
public static final String[] ALL        = {"employee:read", "employee:write", ..., "sys:config"};
public static final String[] HR_FULL    = {"employee:read", "employee:write", ..., "attendance:approve"};
public static final String[] HR_OPERATE = {"employee:read", "employee:write", "employee:export"};
public static final String[] DEPT_HEAD  = {"employee:read", "attendance:read", "approval:view", "approval:approve"};
public static final String[] EMPLOYEE   = {"employee:read", "attendance:read", "approval:view"};
```

初始化也需要同步传入 `data_scope`：

```java
@Component
public class RolePermissionInitializer implements CommandLineRunner {
    @Override
    public void run(String... args) {
        insertRole("ADMIN",       "系统管理员",     Permissions.ALL,        DataScope.ALL);
        insertRole("HR_MANAGER",  "HR 管理员",      Permissions.HR_FULL,    DataScope.ALL);
        insertRole("HR_OPERATOR", "HR 专员",        Permissions.HR_OPERATE, DataScope.ALL);
        insertRole("HR_PAYROLL",  "HR 薪资专员",    Permissions.HR_PAYROLL, DataScope.ALL);
        insertRole("DEPT_HEAD",   "部门主管",       Permissions.DEPT_HEAD,  DataScope.DEPT_TREE);
        insertRole("HRBP",        "HRBP",           Permissions.HRBP,       DataScope.DEPT);
        insertRole("EMPLOYEE",    "普通员工",       Permissions.EMPLOYEE,   DataScope.SELF);
    }
}
```

> `data_scope` 是角色的固有属性：DEPT_HEAD 的角色定义就是管自己团队，不管谁担任这个角色。用户如果兼任多个角色，取所有角色中的 **最大范围**（ALL > DEPT_TREE > DEPT > SELF）。

> 权限变更时：修改 `Permissions.java` 中的数组 → 执行 `update role set permissions = ? where code = ?` → 生效。
> 不需要建表、不需要外键、不需要中间表。

### 建议的 HR 角色

| 角色 | 编码 | 说明 |
|------|------|------|
| 系统管理员 | `ADMIN` | 系统配置、用户管理、角色分配 |
| HR 管理员 | `HR_MANAGER` | 员工管理、薪资管理、考勤管理全部权限 |
| HR 专员 | `HR_OPERATOR` | 员工档案编辑、入职离职办理 |
| HR 薪资专员 | `HR_PAYROLL` | 薪资核算、工资单查看（仅限数据） |
| 部门主管 | `DEPT_HEAD` | 查看本部门员工、审批待办 |
| 普通员工 | `EMPLOYEE` | 个人信息查看、请假申请、工资单查看（自己） |

### 权限变更处理

> JWT 无状态认证的核心矛盾：**签出去的 token 在过期前无法撤销**。
> HR 系统对权限变更的时效性要求高（员工离职、调岗必须立即生效）。
>
> 策略：**不依赖 JWT 撤销，用短 TTL + 刷新时重查 + 出入职干预**。

#### 核心策略

| 手段 | 生效延迟 | 实现 |
|------|---------|------|
| **短 TTL** | access_token 5-15 分钟自动过期 | token 不放权限，载荷最小化 |
| **刷新时重查** | 下一次续签时立即生效 | `POST /auth/refresh` 从 DB 重新加载权限 |
| **删除 refresh_token** | 即时阻止续签 | 离职/调岗时 Redis 中删除 refresh_token |
| **`@PreAuthorize` 实时查 DB** | 即时 | 每次请求由 `AuthService` 查 `emp_role` + `role_permission` |

#### 典型场景处理

```java
// ========== 场景1：员工离职 ==========
public void terminateEmployee(Long empId) {
    // 1. 更新员工状态
    employeeMapper.updateStatus(empId, "TERMINATED");

    // 2. 删除所有 refresh_token → 前端用完后无法续签
    refreshTokenRepository.deleteByEmpId(empId);

    // 3. 原 access_token 最多存活 5-15 分钟
    //    到期后没有新 token 可用，自动跳转登录页
}

// ========== 场景2：角色变更（如主管→普通员工） ==========
public void changeUserRole(Long empId, Long newRoleId) {
    // 1. 更新角色关联
    empRoleMapper.update(empId, newRoleId);

    // 2. 权限变更后，@PreAuthorize 在下一次请求时就已生效
    //    因为权限查的是 DB，不是 token 里的缓存

    // 3. 可选：删除 refresh_token 强制前端续签
    //    不删也行，最多等一个 access_token 生命周期
    refreshTokenRepository.deleteByEmpId(empId);
}

// ========== 场景3：紧急冻结账号 ==========
public void freezeAccount(Long empId) {
    employeeMapper.updateStatus(empId, "FROZEN");
    refreshTokenRepository.deleteByEmpId(empId);
    // 5 分钟后所有 token 失效
}
```

#### 为什么不用黑名单？

```java
// 方案一：Redis 黑名单（不采用）
// 每次请求查 Redis → 多一次 IO
// 黑名单条数随 token 数量线性增长
// HR 系统场景少（每天几次权限变更），不值得引入黑名单复杂度

// 方案二：token_version 字段（备选）
// 如果要求"秒级失效"，在 employee 表加版本号字段
// 每次变更递增版本号，JWT 过滤器查 Redis 对比版本号
// 代价：每次请求多一次 Redis GET，复杂度稍增
```

**结论**：MVP 只需 `短 TTL + 删除 refresh_token`。这俩组合已经覆盖离职/调岗/冻结场景。

### 权限粒度

权限编码格式：`{模块}:{操作}`

| 模块 | 操作 | 编码示例 |
|------|------|---------|
| 员工 | 查看/新增/编辑/删除/导出 | `employee:read / :write / :delete / :export` |
| 考勤 | 查看/编辑/审批 | `attendance:read / :write / :approve` |
| 薪资 | 查看/核算/发放 | `salary:read / :calculate / :pay` |
| 审批 | 查看/审批/干预 | `approval:view / :approve / :intervene` |

## 五、数据权限（PostgreSQL 深度方案）

> HR 系统数据权限的本质：**谁能看到哪些员工的数据？** 所有数据（考勤、薪资、绩效、审批）最终都挂在员工和组织树上。
>
> PostgreSQL 在数据权限上有天然优势：LTREE 扩展、递归 CTE、RLS、JSONB，比 MySQL 灵活得多。

### 5.1 组织树设计（数据权限的基础）

组织树是所有数据权限的根基。有两种方案：

#### 方案 A：物化路径（推荐，适合 MyBatis-Plus）

```sql
CREATE TABLE department (
    id          BIGSERIAL    PRIMARY KEY,
    parent_id   BIGINT       REFERENCES department(id),
    name        VARCHAR(100) NOT NULL,
    path        VARCHAR(500) NOT NULL,   -- 物化路径，如 "/1/10/20/"
    level       SMALLINT     NOT NULL,   -- 层级深度，根=1
    sort_order  INT          DEFAULT 0
);

CREATE INDEX idx_dept_parent ON department(parent_id);
CREATE INDEX idx_dept_path   ON department(path);
```

- `path = /1/10/20/` 表示：根→部门1→部门10→部门20
- 查询部门 10 **及其所有子部门**：

```sql
SELECT id FROM department WHERE path LIKE '/1/10/%';
```

- 查询部门 20 的**所有祖先**（向上追溯）：

```sql
SELECT id FROM department WHERE '/1/10/20/' LIKE path || '%' AND id != 20;
```

**优势**：SQL 简单直观，MyBatis-Plus 的 `in` 查询直接拼字符串，不需数据库特殊功能。

#### 方案 B：PostgreSQL LTREE 扩展（最强树查询）

```sql
CREATE EXTENSION IF NOT EXISTS ltree;

CREATE TABLE department (
    id          BIGSERIAL    PRIMARY KEY,
    parent_id   BIGINT       REFERENCES department(id),
    name        VARCHAR(100) NOT NULL,
    path        LTREE        NOT NULL,       -- 如 "1.10.20"
    sort_order  INT          DEFAULT 0
);

CREATE INDEX idx_dept_path_ltree ON department USING GIST (path);
```

**LTREE 操作符：**

| 操作符 | 含义 | 示例 |
|--------|------|------|
| `@>` | 祖先包含后代 | `path @> '1.10'` → 1.10 的所有子孙 |
| `<@` | 后代的祖先 | `path <@ '1.10.20'` → 1.10.20 的所有祖先 |
| `~` | 正则匹配 | `path ~ '*.10.*'` → 路径中包含 10 的节点 |
| `||` | 拼接 | `'1.10' || '20'` → `1.10.20` |

```sql
-- 查找部门 10 及其所有子部门（一键）
SELECT id FROM department WHERE path <@ '1.10';

-- 查找部门 20 的所有祖先
SELECT id FROM department WHERE path @> '1.10.20' AND id != 20;

-- 查找直接子部门（最常用）
SELECT id FROM department WHERE path ~ '1.10.*{1}';
```

**优势**：查询性能最优（GiST 索引），语法最简洁，支持复杂的层级匹配。

**劣势**：需要理解 LTREE 语法，MyBatis-Plus 的 `in` 查询需要额外处理。

> **建议**：如果团队 PG 经验丰富 → **LTREE**；否则 → **物化路径**，功能够用且更通用。

### 5.2 员工表中的权限相关字段

```sql
CREATE TABLE employee (
    id              BIGSERIAL    PRIMARY KEY,
    name            VARCHAR(50)  NOT NULL,
    dept_id         BIGINT       NOT NULL REFERENCES department(id),
    dept_path       VARCHAR(500) NOT NULL,    -- 冗余字段，来自 department.path，免联表
    report_to       BIGINT       REFERENCES employee(id),  -- 直接上级
    report_to_path  VARCHAR(500),             -- 上级链，如 "/101/201/301/"
    emp_type        VARCHAR(20)  NOT NULL,     -- REGULAR / OUTSOURCE / INTERN / ...
    status          VARCHAR(20)  NOT NULL DEFAULT 'ACTIVE',
    ...
);
```

> `dept_path` 从部门表冗余到员工表，**查询时不用 JOIN**，数据权限过滤效率最高。

### 5.3 数据权限级别

> `data_scope` 是 **role 表的字段**，不是 emp_role 的扩展字段。因为数据范围是角色的固有属性 —— DEPT_HEAD 天然就该管自己团队，不管谁在这个位置。用户兼任多角色时取所有角色的 **最大范围**（ALL > DEPT_TREE > DEPT > SELF）。

```java
public enum DataScope {
    ALL,          // 全公司可见
    DEPT_TREE,    // 本部门及下级部门
    DEPT,         // 仅本部门
    SELF,         // 仅自己
    CUSTOM        // 自定义范围（扩展用）
}

public class EmployeePrincipal {
    // 取用户所有角色中的最大 data_scope
    public DataScope getEffectiveDataScope() {
        return userRoles.stream()
            .map(Role::getDataScope)
            .min(Comparator.comparingInt(DataScope::getLevel))  // ALL=0 > ... > SELF=3
            .orElse(DataScope.SELF);
    }
}
```

**每个角色对应的数据范围：**

| 角色 | 数据范围（role.data_scope） | 典型业务场景 |
|------|---------------------------|-------------|
| ADMIN | ALL | 系统配置，无数据限制 |
| HR_MANAGER | ALL | 看全公司员工档案 |
| HR_OPERATOR | ALL | 办理入职离职不受部门限制 |
| HR_PAYROLL | ALL | 全公司薪资（敏感操作有审计） |
| DEPT_HEAD | DEPT_TREE | 看自己团队员工、考勤、绩效 |
| HRBP | DEPT | 看所负责的部门 |
| EMPLOYEE | SELF | 仅看自己 |

### 5.4 核心实现：数据权限过滤器

#### MyBatis-Plus 拦截器方式（推荐）

这是中国 Java 生态中最成熟的方案 —— **拦截 Mapper 的 SQL，自动拼接数据权限条件**。

```java
@Component
public class DataPermissionInterceptor implements InnerInterceptor {

    @Override
    public void beforeQuery(Executor executor, MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql) {
        // 1. 从 SecurityContext 获取当前用户
        EmployeePrincipal current = SecurityUtils.getCurrentUser();
        if (current == null || current.hasDataScope(DataScope.ALL)) {
            return;  // ALL 范围无需拦截
        }

        // 2. 解析原始 SQL 的抽象语法树
        String originalSql = boundSql.getSql();
        String permissionSql = injectDataPermission(originalSql, current);
        
        // 3. 通过反射替换 BoundSql 中的 SQL
        ReflectUtil.setFieldValue(boundSql, "sql", permissionSql);
    }

    private String injectDataPermission(String sql, EmployeePrincipal user) {
        // 根据当前用户的数据范围，注入 WHERE 条件
        // 正则/JSqlParser 解析 SQL，找到 FROM table，追加 dept_path 过滤
    }
}
```

**注入后的 SQL 效果：**

```sql
-- 原始 SQL
SELECT * FROM employee WHERE status = 'ACTIVE' ORDER BY id

-- DEPT_TREE 范围自动注入后
SELECT * FROM employee 
WHERE status = 'ACTIVE' 
  AND dept_path LIKE '/1/10/%'  -- ← 自动加上，用户部门路径 = /1/10/
ORDER BY id

-- SELF 范围自动注入后
SELECT * FROM employee 
WHERE status = 'ACTIVE' 
  AND id = 1001                 -- ← 自动加上，只查自己
ORDER BY id
```

**表级别控制：** 不是所有表都需要数据权限过滤。通过注解标记：

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
public @interface DataScope {
    String tableAlias() default "";       -- 表别名，如 e
    String deptPathColumn() default "dept_path";   -- 部门路径字段名
    String empIdColumn() default "id";              -- 员工ID字段名
    boolean enable() default true;
}
```

```java
// 员工表 Mapper —— 自动过滤
@DataScope(tableAlias = "e")
@Mapper
public interface EmployeeMapper extends BaseMapper<Employee> {}

// 配置表，不需要过滤
@Mapper
public interface SystemConfigMapper extends BaseMapper<SystemConfig> {}
```

#### 手动控制方式（适用于复杂查询）

有些场景自动拦截不够用（如跨多表 JOIN），需要手动控制：

```java
@Service
public class SalaryServiceImpl implements SalaryService {

    @Override
    public PageResult<SalaryVO> pageQuery(SalaryQuery query) {
        // 获取当前用户的数据权限范围
        DataScope scope = SecurityUtils.getCurrentUser().getDataScope();
        
        // 手写 Lambda QueryWrapper，条件清晰
        LambdaQueryWrapper<Salary> wrapper = Wrappers.lambdaQuery();
        
        switch (scope) {
            case ALL:
                break;  // 无条件
            case DEPT_TREE:
                wrapper.in(Salary::getDeptId, getDeptTreeIds(user.getDeptId()));
                break;
            case DEPT:
                wrapper.eq(Salary::getDeptId, user.getDeptId());
                break;
            case SELF:
                wrapper.eq(Salary::getEmpId, user.getEmpId());
                break;
        }
        return salaryMapper.selectPage(new Page<>(query.getPage(), query.getSize()), wrapper);
    }
}
```

### 5.5 PostgreSQL Row-Level Security（RLS）

> 💡 RLS 是 PostgreSQL 独有的特性，**从数据库层面强制数据行权限**，即使绕过应用直接查数据库也受控。

```sql
-- 1. 启用 RLS
ALTER TABLE employee ENABLE ROW LEVEL SECURITY;

-- 2. 创建 RLS 策略（示例：员工只能看自己的数据）
CREATE POLICY emp_self_policy ON employee
    FOR ALL
    USING (id = current_setting('app.current_emp_id')::BIGINT);

-- 3. 部门主管看自己和下属的数据
CREATE POLICY emp_dept_tree_policy ON employee
    FOR SELECT
    USING (
        current_setting('app.data_scope') = 'DEPT_TREE'
        AND dept_path LIKE (
            SELECT path || '%' FROM department 
            WHERE id = current_setting('app.dept_id')::BIGINT
        )
    );
```

**Java 端设置**：每次请求时，在 JWT 过滤器中设置 PostgreSQL session 变量：

```java
public class JwtAuthenticationFilter extends OncePerRequestFilter {
    @Override
    protected void doFilterInternal(...) {
        // JWT 解析后，将用户上下文注入到 PG session
        JdbcTemplate jdbc = ...;
        jdbc.execute("SET app.current_emp_id = " + principal.getEmpId());
        jdbc.execute("SET app.data_scope = '" + principal.getDataScope() + "'");
        jdbc.execute("SET app.dept_id = " + principal.getDeptId());
        
        chain.doFilter(request, response);
    }
}
```

> **RLS 的优势与局限**：
> - ✅ 安全级别最高，数据库层兜底
> - ✅ 防止"忘了加权限过滤"的 Bug
> - ❌ 调试复杂，策略多了排查困难
> - ❌ 和 MyBatis-Plus 的分页/乐观锁配合需要额外测试
>
> **建议**：二期再引入 RLS 作为安全兜底层，一期先用应用层拦截器。

### 5.6 完整流程示例

以 **"部门主管查看团队员工列表"** 为例：

```
1. 用户登录
   → JWT 中携带 dept_path = "/1/10/", data_scope = "DEPT_TREE"

2. 前端请求 GET /api/employees?page=1&size=20

3. JWT 过滤器解析 token，设置 SecurityContext
   → principal.deptPath = "/1/10/"
   → principal.dataScope = "DEPT_TREE"

4. EmployeeMapper.selectPage() 执行
   → DataPermissionInterceptor 拦截 SQL
   → 注入 WHERE dept_path LIKE '/1/10/%'
   → 最终 SQL:
       SELECT * FROM employee 
       WHERE status = 'ACTIVE' 
         AND dept_path LIKE '/1/10/%'    ← 自动注入
       ORDER BY id
       LIMIT 20 OFFSET 0

5. 结果：主管看到自己部门及所有下级部门的员工
```

### 5.7 各业务模块的数据权限矩阵

| 模块 | 接口 | ADMIN | HR_MANAGER | DEPT_HEAD | HRBP | EMPLOYEE |
|------|------|-------|-----------|-----------|------|----------|
| 员工列表 | GET /employees | ALL | ALL | DEPT_TREE | DEPT | SELF |
| 员工详情 | GET /employees/{id} | ALL | ALL | DEPT_TREE | DEPT | SELF |
| 新增员工 | POST /employees | ALL | ALL | ❌ | ❌ | ❌ |
| 薪资列表 | GET /salary | ALL | ALL | DEPT_TREE(仅查看) | ❌ | SELF |
| 薪资核算 | POST /salary/calculate | ALL | ❌ | ❌ | ❌ | ❌ |
| 考勤报表 | GET /attendance/report | ALL | ALL | DEPT_TREE | DEPT | SELF |
| 审批待办 | GET /approval/todos | ALL | ALL | DEPT_TREE | DEPT | SELF |

### 5.8 数据权限 + MyBatis-Plus 最佳实践

```yaml
# application.yml — MyBatis-Plus 配置
mybatis-plus:
  configuration:
    # 驼峰转下划线
    map-underscore-to-camel-case: true
  global-config:
    db-config:
      # PostgreSQL 主键策略
      id-type: ASSIGN_ID
  # 拦截器注入
  config-location: classpath:mybatis-config.xml
```

```java
@Configuration
public class MyBatisPlusConfig {
    
    @Bean
    public MybatisPlusInterceptor mybatisPlusInterceptor() {
        MybatisPlusInterceptor interceptor = new MybatisPlusInterceptor();
        
        // 分页插件（PostgreSQL 方言）
        PaginationInnerInterceptor pagination = new PaginationInnerInterceptor(DbType.POSTGRES_SQL);
        interceptor.addInnerInterceptor(pagination);
        
        // 数据权限插件（自定义，见上）
        interceptor.addInnerInterceptor(new DataPermissionInterceptor());
        
        // 乐观锁插件（用于薪资并发修改）
        interceptor.addInnerInterceptor(new OptimisticLockerInnerInterceptor());
        
        return interceptor;
    }
}
```

### 5.9 一期 vs 二期

| 阶段 | 数据权限方案 | 原因 |
|------|-------------|------|
| **一期** | **手动拼接 `Wrappers.lambdaQuery()` + `eq`/`in` 条件** | 最简单，每行代码逻辑清晰，不出奇奇怪怪的 SQL 注入 Bug |
| **二期** | **MyBatis-Plus DataPermissionInterceptor** | 模块多了以后减少重复代码，统一管理 |
| **三期** | **PostgreSQL RLS 兜底** | 安全加固，防止绕过 |

> **核心建议**：一期的数据权限**不要上拦截器**。每个 Mapper 方法显式传入当前用户的数据范围，比"魔法拦截"更可控。HR 系统的数据权限错误（如一个人看到全公司薪资）是严重生产事故，宁可代码多几行，不要靠黑魔法。

## 六、一期 MVP 范围

**必做：**
- [x] JWT 双令牌（access + refresh）
- [ ] 登录 / 退出接口
- [ ] JWT 过滤器 + SecurityConfig
- [ ] 角色 + 权限基础表结构
- [ ] `@PreAuthorize` 接口级权限
- [ ] 密码加密（BCryptPasswordEncoder）

**二期再做：**
- [ ] 数据权限拦截器
- [ ] 权限管理界面（角色分配）
- [ ] 操作审计日志
- [ ] OAuth2 / SSO 集成
- [ ] 菜单权限动态生成
