# 高级功能开发指南

## 目录

1. [审批流程](#1-审批流程)
2. [权限系统](#2-权限系统)
3. [飞书集成](#3-飞书集成)
4. [调度任务](#4-调度任务)
5. [导入导出](#5-导入导出)
6. [数据分析](#6-数据分析)
7. [通知系统](#7-通知系统)
8. [Frappe 框架覆写](#8-frappe-框架覆写)
9. [其他工具](#9-其他工具)

## 1. 审批流程

### 审批状态机

```
未开始 → 待审批 → 已通过
              ↓         ↑
           已驳回 ──────┘（重新提交）
              ↓
           已撤回
```

| 状态 | 常量 | 说明 |
|------|------|------|
| 未开始 | `SubmissionStatus.NOT_STARTED` | 未提交 |
| 待审批 | `ApprovalStatus.PENDING` | 已提交，等待审批 |
| 已通过 | `ApprovalStatus.APPROVED` | 审批完成 |
| 已驳回 | `ApprovalStatus.REJECTED` | 审批驳回 |
| 已撤回 | — | 提交人撤回 |

前端可提交状态：`['未开始', '已撤回', '已驳回']`（定义在 `useApproval.js` 的 `SUBMITTABLE_STATUSES`）

### 审批相关 DocType

- `approval_instance` — 审批实例
- `approval_history` — 审批历史
- `approval_config` — 审批配置（步骤和审批人）

### 审批 API

```python
from product_sales_planning.services.approval_service import submit_for_approval
from product_sales_planning.api.v1.approval import approve, reject, withdraw

# 提交审批
result = submit_for_approval(
    task_id="TASK001",
    store_id="STORE001",
    submitter="user@example.com"
)

# 审批操作
approve(task_id, store_id, approver, comments)
reject(task_id, store_id, approver, comments, reject_to_step)
withdraw(task_id, store_id, submitter)
```

### 审批前端

`useApproval.js` composable 提供 5 个 `createResource`：
- `statusResource` — 获取审批状态
- `historyResource` — 获取审批历史
- `submitResource` — 提交审批
- `withdrawResource` — 撤回审批
- `approveResource` — 审批通过

状态映射（定义在 `useApproval.js`）：
- `STATUS_TEXT_MAP` — 状态 → 显示文本
- `STATUS_THEME_MAP` — 状态 → frappe-ui Badge 主题色

### 审批通知

审批服务自动通过 `send_approval_notification` 发送通知，类型有：
- `pending` — 待审批通知（发给审批人）
- `approved` — 审批通过通知（发给提交人）
- `rejected` — 驳回通知（发给提交人或上一级审批人）
- `withdrawn` — 撤回通知（发给当前审批人）

### 审批配置

通过 `approval_config` DocType 配置审批流程步骤和审批人，由 `approval_config_service.py` 管理业务逻辑。

## 2. 权限系统

### 三层权限架构

1. **Frappe 内置角色权限** — DocType 级别的 CRUD 控制
2. **DocType `has_permission()` 覆写** — 记录级权限（如 `StoreList`）
3. **自定义 Permission Config** — 精细化权限配置

### Permission Config 结构

```
Permission Config (权限配置)
├── Permission Config User (用户权限)
├── Permission Config Role (角色权限)
├── Permission Config Department (部门权限)
├── Permission Config Store (店铺权限)
├── Permission Config DocType (DocType权限)
└── Permission Config Platform (平台权限)
```

### 相关 DocType

- `permission_config` — 权限配置主表
- `permission_config_user` / `permission_config_role` / `permission_config_department` / `permission_config_store` / `permission_config_doctype` / `permission_config_platform` — 子表
- `frontend_page_permission` / `frontend_page_permission_role` — 前端页面权限

### 后端权限检查

```python
# API 层 — 使用装饰器或手动验证
from product_sales_planning.utils.api_decorators import handle_exceptions

@frappe.whitelist()
@handle_exceptions
def update_data(name):
    # 手动权限检查
    from product_sales_planning.services.permission_config_service import PermissionConfigService
    if not PermissionConfigService.check_permission(frappe.session.user, name, "write"):
        frappe.throw("无权限操作", frappe.PermissionError)
    ...

# DocType 层
class MyDocType(Document):
    def has_permission(self, ptype="read", user=None):
        if user == "Administrator":
            return True
        from product_sales_planning.services.permission_config_service import PermissionConfigService
        return PermissionConfigService.check_permission(user, self.name, ptype)
```

### 前端权限

- `permissionStore`（Vue `reactive`，**不是 Pinia**）— 用户角色和页面权限缓存
  - 三级数据源：`window.boot` > `sessionStorage` > API 请求
  - 缓存 TTL 5 分钟
  - 直接作为对象使用：`permissionStore.hasRole('System Manager')`
- `permissionGuard.js` — 路由守卫，拦截无权限访问
- `permissions.js` — 页面权限配置映射，含 `canAccessRoute()`、`hasRole()`、`getVisibleMenus()`

### 店铺列表特殊权限

`Store List` 有自定义权限查询条件（`hooks.py → permission_query_conditions`）和 `has_permission` 覆写，实现数据行级别的访问控制。

## 3. 飞书集成

### OAuth 登录

覆写 Frappe OAuth 流程，登录后重定向到 `/planning`：

```python
# overrides/oauth.py
# - 处理飞书 OAuth 回调
# - 自动创建/关联 Frappe 用户
# - 重定向到 /planning 而非 /app
# - 更新登录上下文（update_login_context）
```

飞书 Social Login Key 作为 fixture 预加载（`hooks.py → fixtures`）。

### 飞书通知

```python
from product_sales_planning.services.feishu_notification import FeishuNotification

# 发送飞书消息
FeishuNotification.send_message(
    user_id="user_key",
    title="审批通知",
    content="您有一条新的审批待处理"
)
```

### 用户导入

```python
# api/v1/import_feishu_users.py
# 从飞书组织架构同步用户到 Frappe
```

## 4. 调度任务

### 自动关闭过期任务

```python
# hooks.py scheduler_events
scheduler_events = {
    "hourly": ["product_sales_planning.planning_system.doctype.schedule_tasks.schedule_tasks.auto_close_expired_tasks"]
}
```

### 提醒服务

```python
from product_sales_planning.services.reminder_service import ReminderService

# 发送任务提醒
ReminderService.send_reminder(task_id="TASK001")
```

### 相关 DocType

- `schedule_tasks` — 调度任务
- `tasks_store` — 任务关联店铺
- `reminder_log` — 提醒日志

## 5. 导入导出

### API

```python
# api/v1/import_export.py
# - 导入 Excel 数据到 Commodity Schedule
# - 导出 Commodity Schedule 为 Excel
# - 支持 .xlsx / .xls 格式
# - 最大导入行数 10000，导出 50000
# - 最大文件大小 10MB
```

### 前端

- `ProductImportDialog.vue` — 导入对话框
- `ProductAddDialog.vue` — 添加商品对话框

## 6. 数据分析

### 双路径查询

| 路径 | 文件 | 数据源 | 适用场景 |
|------|------|--------|----------|
| MariaDB | `data_analysis.py` | Frappe DB | 小数据量、实时查询 |
| ClickHouse | `data_analysis_clickhouse.py` | ClickHouse | 大数据量、聚合分析 |

### 查询优化

`OptimizedQueryBuilder`（`utils/query_builder.py`）提供 SQL 查询优化：
- 使用 JOIN 替代 IN 子查询
- 使用 EXISTS 替代大量 OR 条件
- 数据库层面分页（LIMIT OFFSET）
- 合并多个独立查询为单个查询

核心表别名：`cs`(Commodity Schedule)、`ai`(Approval Instance)、`sl`(Store List)、`pl`(Product List)

支持维度：`detail`（明细）、`store`（店铺汇总）、`product`（商品汇总）

### 分析功能

- **透视分析** — `data_analysis_pivot.py` + 透视组件
- **固定报表** — `data_analysis_fixed_report.py` + `useFixedReport.js` + `FixedReportTable.vue`
- **计划对比** — `plan_comparison.py`
- **规则引擎** — `planningRuleEngine.js`（前端规则评估，支持跨任务对比）

### ClickHouse 工具

- `clickhouse_client.py` — 连接客户端（HTTP 协议，模块级单例）
- `clickhouse_sales.py` — 销售数据查询
- `clickhouse_materialized_views.py` — 物化视图管理

## 7. 通知系统

双通道通知：飞书机器人 + 站内消息。

```python
from product_sales_planning.services.notification_service import NotificationService

# 发送通知（自动双通道）
NotificationService.notify(
    user="user@example.com",
    title="审批通知",
    message="您有一条新的审批",
    feishu=True  # 同时发飞书
)
```

前端组件：
- `NotificationBell.vue` — 通知铃铛
- `NotificationModal.vue` — 通知弹窗
- `useNotification.js` — 通知 composable

## 8. Frappe 框架覆写

### OAuth 覆写

`overrides/oauth.py` — 飞书 OAuth 登录流程定制

### 用户钩子

`overrides/user_hooks.py` — 新用户创建时屏蔽所有模块（`before_insert`，通过 `doc_events` 注册）

### Workspace 覆写

`overrides/workspace.py` — 覆写 `save_page` 处理对象格式的 `new_widgets`

### 安装后钩子

```python
# hooks.py
after_install = "product_sales_planning.install.after_install"
after_migrate = ["product_sales_planning.install.after_migrate"]
```

### Boot Session 过滤

`hooks.py → boot_session` 注册 `filter_help_dropdown`，过滤掉帮助下拉中的 "Frappe Support" 项。

## 9. 其他工具

### 日期工具

`utils/date_utils.py` — 日期计算、计划月份推算
- `get_date_range_filter` — 日期范围筛选
- `get_month_first_day` — 获取月份第一天

### 查询构建器

`utils/query_builder.py` — `OptimizedQueryBuilder` 类，SQL 查询构建辅助

### 验证工具

`utils/validation_utils.py` — 输入校验工具函数
- `validate_required_params` — 必填参数验证
- `validate_positive_integer` — 正整数验证
- `validate_doctype_exists` — DocType 存在性验证

### 前端工具

- `frappeRequest.js` — 自定义请求函数，CSRF token 管理 + 自动重试
- `cacheManager.ts` — 前端缓存管理
- `pivotConfigStorage.js` — 透视配置持久化
- `planningRuleEngine.js` — 计划规则引擎（`PlanningRuleEvaluator` 类）
- `helpers.js` — 通用工具（debounce、formatTime）

### 测试与修复

- `utils/generate_test_data_directly.py` — 直接生成测试数据
- `commands/repair_data.py` — bench 数据修复命令
