---
name: psp-dev
description: |
  Product Sales Planning (PSP) 全栈开发助手。适用于开发 Frappe + Vue 3 销售规划系统的新功能、API、DocType 和前端组件。

  即使没有显式提到 PSP，也应在以下场景触发此技能：
  - 开发、修改、调试 product_sales_planning 项目的任何代码（后端或前端）
  - 创建或修改 Frappe DocType、编写 API 接口、配置权限
  - 添加 Vue 页面/组件/composable、配置路由
  - 涉及 Handsontable、@antv/s2、@visactor/vtable 表格引擎的工作
  - 实现审批流程、飞书集成（OAuth/通知）、ClickHouse 查询
  - 商品计划、店铺管理、数据分析、透视表、固定报表
  - 导入导出、调度任务、计划对比、机制管理
  - 在 product_sales_planning 目录下工作的任何开发任务——因为这是一个有特殊架构模式的项目，通用开发建议可能不符合项目约定

  关键词：PSP、销售计划、Frappe、Vue3、frappe-ui、审批、权限、飞书、ClickHouse、Handsontable、s2、vtable、DocType、composable。
---

# PSP 全栈开发指南

本项目是基于 **Frappe + Vue 3** 的商品销售规划系统，用于零售品牌的月度计划管理、临时销售计划、新品计划等。

## 技术栈

| 层级 | 技术 | 说明 |
|------|------|------|
| 后端 | Python 3.10+ / Frappe | MariaDB ORM + REST API |
| 前端 | Vue 3.5+ / Vite 5 | Composition API + frappe-ui |
| UI | TailwindCSS 3.4 + frappe-ui | CSS 变量主题系统 |
| 表格 | Handsontable / @antv/s2 / @visactor/vtable | 三种表格引擎 |
| 图表 | ECharts | 数据可视化 |
| 数据仓库 | ClickHouse | 高性能分析查询 |
| 认证 | 飞书 OAuth | 企业单点登录 |
| 通知 | 飞书机器人 + 站内消息 | 双通道通知 |

## 目录结构

```
product_sales_planning/
├── product_sales_planning/          # Python 后端
│   ├── api/v1/                      # REST API（24 个端点文件）
│   ├── planning_system/doctype/     # DocType（25 个模块）
│   ├── services/                    # 业务逻辑层（8 个服务）
│   ├── utils/                       # 工具函数（13 个）
│   ├── overrides/                   # Frappe 框架覆写（OAuth/用户/Workspace）
│   ├── patches/                     # 数据库补丁（7 个）
│   ├── commands/                    # bench 自定义命令
│   ├── www/                         # Frappe 门户页面（5 个）
│   └── constants.py                 # 状态常量、错误码、分页限制
├── frontend/                        # Vue 3 前端
│   ├── src/pages/                   # 页面组件（6 个）
│   ├── src/components/              # 通用组件（17 个，含子目录）
│   ├── src/composables/             # 组合式函数（5 个）
│   ├── src/stores/                  # 响应式状态（permissionStore，Vue reactive 实现）
│   ├── src/api/                     # API 调用层
│   ├── src/config/                  # 权限配置 + 审批状态配置
│   ├── src/utils/                   # 工具函数（frappeRequest/cacheManager 等）
│   ├── src/layouts/                 # 布局组件（MainLayout）
│   └── src/router/                  # 路由 + 守卫
└── public/planning/                 # 构建输出
```

## 核心架构模式

### 1. 请求流

```
Browser (/planning/*)
  → Vite dev proxy (dev) / Nginx (prod)
  → Frappe www/planning.py (SPA shell)
  → Vue Router → MainLayout → Page → Composable → API
  → frappeRequest / frappe.call → /api/method/product_sales_planning.api.v1.*
  → MariaDB / ClickHouse
```

### 2. API 装饰器链

项目提供两个装饰器，按需组合使用。**注意：项目中不存在 `@api_endpoint` 组合装饰器，不要引用。**

```python
# 标准写法：@handle_exceptions 处理异常并返回统一格式
@frappe.whitelist()
@handle_exceptions
def my_read_api():
    pass

# 写操作：加上 @with_transaction 管理事务
@frappe.whitelist()
@handle_exceptions
@with_transaction
def my_write_api():
    pass

# 需要权限检查时：手动调用权限工具
from product_sales_planning.utils.api_decorators import handle_exceptions, with_transaction
from product_sales_planning.utils.validation_utils import validate_required_params
```

`@handle_exceptions` 会捕获 4 类 Frappe 异常并映射为 `error_response`：
- `ValidationError` → `VALIDATION_ERROR`
- `PermissionError` → `PERMISSION_DENIED`
- `DoesNotExistError` → `NOT_FOUND`
- 其他 → `INTERNAL_ERROR`

`@with_transaction` 自动管理 `frappe.db.begin()` / `commit()` / `rollback()`。

### 3. 统一响应格式

所有 API 必须使用 `success_response` 或 `error_response`，不要返回裸数据或 frappe 原始响应。

```python
from product_sales_planning.utils.response_utils import success_response, error_response
from product_sales_planning.constants import ErrorCode

return success_response(data={"items": [...]})
return success_response(data=result, message="操作成功")
return error_response(message="操作失败", error_code=ErrorCode.VALIDATION_ERROR)
```

响应结构：`success` → `{"status": "success", "message": "...", "data": ...}`，`error` → `{"status": "error", "message": "...", "error_code": "..."}`。两者均支持 `**kwargs` 扩展字段。

### 4. 前端请求方式

项目使用自定义 `frappeRequest`（`src/utils/frappeRequest.js`）替代 frappe-ui 默认 request，核心功能：
- CSRF token 三源获取（cookie → `window.csrf_token` → `window.frappe.csrf_token`）
- CSRF 错误自动刷新 token 并重试
- GET 参数用 `URLSearchParams`，POST 用 JSON body
- 自动前缀 `/api/method/`

Composable 中优先使用 `createResource`（frappe-ui）管理 API 资源，配合 `frappeRequest`：
```javascript
const resource = createResource({
    url: 'product_sales_planning.api.v1.my_api.get_data',
    makeParams: () => ({ task_id, store_id }),
    auto: false,
    transform: (res) => {
        const data = res?.message || res
        return data?.status === 'success' ? data.data : null
    }
})
```

### 5. 前端状态管理

**重要：`permissionStore` 不是 Pinia store，而是基于 Vue `reactive`/`readonly` 的响应式对象。**
- 使用时直接调用方法：`permissionStore.hasRole('System Manager')`，不需要加括号调用
- 三级数据源优先级：`window.boot` > `sessionStorage` 缓存 > API 请求
- 缓存 TTL 5 分钟，存储在 `sessionStorage`

### 6. 前端 Composable 模式

页面逻辑提取到 `composables/use*.js`，保持页面组件精简。项目现有 5 个 composable：
- `useApproval.js` — 审批流程（5 个 createResource）
- `useFixedReport.js` — 固定报表
- `useNotification.js` — 通知系统
- `usePermission.js` — 权限检查
- `useStoreDetail.js` — 店铺详情（最复杂，含数据加载/筛选/保存/导出/列设置）

### 7. 前端布局

`MainLayout.vue` 提供 Sidebar + TopBar + ErrorBoundary + 全局通知弹窗的框架结构。
- `DataAnalysis` 页面使用 `keep-alive` 缓存
- `Suspense` 提供加载状态 fallback
- 路由守卫 `createPermissionGuard()` 检查页面权限

### 8. 缓存失效

`doc_events` 注册在 `hooks.py` 中，当 `Commodity Schedule` 或 `Approval Instance` 变更时自动清除分析缓存。`Permission Config` 变更时清除权限缓存。新增 DocType 如需缓存失效行为，在 `utils/cache_hooks.py` 添加并注册到 `hooks.py → doc_events`。

## 详细开发模板

根据具体任务查阅对应参考文件（无需全部读取）：

- **创建 DocType / 编写 API / 服务层 / ClickHouse** → 读取 `references/backend.md`
- **创建页面 / 组件 / Composable / 表格引擎** → 读取 `references/frontend.md`
- **审批流程 / 权限系统 / 飞书集成 / 调度任务 / 导入导出** → 读取 `references/advanced.md`

## 项目特有陷阱

这些是 PSP 项目中容易踩坑的地方，开发时务必注意：

1. **Frappe API 参数类型**：`@frappe.whitelist()` 的参数都从请求序列化为字符串，必须手动 `int()` / `frappe.parse_json()` 转换。模板中已体现此模式，不要省略。

2. **Module 名大小写**：DocType JSON 中 `module` 字段必须为 `"planning system"`（全小写），这是 Frappe 的 module 命名约定，大写会导致找不到模块。

3. **API 路径格式**：所有 API 调用路径为 `product_sales_planning.api.v1.file_name.function_name`，注意是下划线文件名，不是驼峰。

4. **ignore_permissions 与权限检查**：API 装饰器已做权限检查，所以 `doc.save(ignore_permissions=True)` 是正确的——不要同时使用装饰器检查权限又让 ORM 再检查一次。

5. **前端构建输出**：`npm run build` 输出到 `public/planning/`，这些文件会被 Git 追踪。构建后需确认输出正确。

6. **Handsontable 许可**：项目使用 Handsontable 商业版，不要更换为社区版。

7. **`@api_endpoint` 不存在**：项目中没有 `@api_endpoint` 装饰器，只有 `@handle_exceptions` 和 `@with_transaction`。不要在代码中引用不存在的装饰器。

8. **前端 createResource 的 transform**：frappe-ui 的 `createResource` 返回的数据可能在 `res.message` 或直接在 `res` 中，transform 中需要兼容两种情况：`const data = res?.message || res`。

9. **计划数量显示**：数量为 0 时前端显示为空字符串，避免误导为已提报 0。参考 `useStoreDetail.js` 中的 `formatPlanQuantityForTable`。

## 核心常量速查

### 状态常量

| 类 | 常量 | 值 |
|------|------|------|
| `ApprovalStatus` | PENDING / APPROVED / REJECTED | 待审批 / 已通过 / 已驳回 |
| `SubmissionStatus` | NOT_STARTED / SUBMITTED | 未开始 / 已提交 |
| `TaskStatus` | NOT_STARTED / IN_PROGRESS / COMPLETED / CLOSED | 未开始 / 开启中 / 已完成 / 已关闭 |
| `ViewMode` | SINGLE / MULTI | single / multi |

### 错误码

| 常量 | 值 |
|------|------|
| `ErrorCode.VALIDATION_ERROR` | VALIDATION_ERROR |
| `ErrorCode.PERMISSION_DENIED` | PERMISSION_DENIED |
| `ErrorCode.NOT_FOUND` | NOT_FOUND |
| `ErrorCode.DUPLICATE` | DUPLICATE |
| `ErrorCode.INTERNAL_ERROR` | INTERNAL_ERROR |
| `ErrorCode.INVALID_PARAMETER` | INVALID_PARAMETER |
| `ErrorCode.TRANSACTION_FAILED` | TRANSACTION_FAILED |

### DocType 名称常量

| 常量 | 值 |
|------|------|
| `DocType.COMMODITY_SCHEDULE` | Commodity Schedule |
| `DocType.STORE_LIST` | Store List |
| `DocType.PRODUCT_LIST` | Product List |
| `DocType.SCHEDULE_TASKS` | Schedule Tasks |
| `DocType.TASKS_STORE` | Tasks Store |
| `DocType.APPROVAL_WORKFLOW` | Approval Workflow |

### 分页与限制

| 常量 | 值 |
|------|------|
| `MAX_PAGE_SIZE` | 1000 |
| `DEFAULT_PAGE_SIZE` | 20 |
| `MIN_PAGE_SIZE` | 1 |
| `MAX_BATCH_SIZE` | 1000 |
| `MAX_IMPORT_ROWS` | 10000 |
| `MAX_EXPORT_ROWS` | 50000 |
| `MAX_FILE_SIZE_MB` | 10 |
| `DEFAULT_PLAN_MONTHS` | 5 |
| `ALLOWED_FILE_EXTENSIONS` | .xlsx, .xls |
| `COMMODITY_EDITABLE_FIELDS` | ["quantity", "sub_date"] |

## 路由映射

| 路径 | 页面 | 说明 |
|------|------|------|
| `/` | PlanningDashboard | 计划看板 |
| `/approval-center` | ApprovalCenter | 审批中心 |
| `/mechanism` | MechanismManagement | 机制管理 |
| `/data-analysis` | DataAnalysis | 数据分析（含透视表，keep-alive 缓存） |
| `/store-detail/:storeId/:taskId` | StoreDetail | 店铺详情（beforeEnter 校验参数） |
| `/no-access` | NoAccess | 无权限 |

## 代码规范

### Python
- 缩进 Tab（4 宽度）、行长 110、双引号、导入排序 `ruff check --select I --fix`
- 命名：PascalCase 类、snake_case 函数、UPPER_SNAKE_CASE 常量

### Vue
- `<script setup>` Composition API、PascalCase 组件、camelCase props/emits、kebab-case CSS
- 深度样式用 `:deep()`、section 顺序：Props → Emits → Refs → Methods → Watchers & Lifecycle

### 国际化
- UI 文本和 API 错误消息：中文；代码注释：中文

## 常用命令

```bash
# 后端
bench --site <site> run-tests --app product_sales_planning
bench --site <site> migrate
ruff format product_sales_planning/ && ruff check --select I --fix product_sales_planning/

# 前端
cd frontend && npm run dev          # http://localhost:8080/planning
cd frontend && npm run build        # 输出到 public/planning/
cd frontend && npm run test         # vitest --run
```

## 开发流程

### 添加新功能模块

1. 创建 DocType → `planning_system/doctype/my_doctype/`
2. 创建 API → `api/v1/my_api.py`（使用装饰器链 + 统一响应）
3. 创建服务层 → `services/my_service.py`（复杂业务逻辑分离）
4. 创建前端页面 → `frontend/src/pages/MyPage.vue`
5. 创建 Composable → `frontend/src/composables/useMyFeature.js`
6. 配置路由 → `frontend/src/router.js`
7. 配置权限 → `frontend/src/config/permissions.js`
8. 编写测试 → `test_my_doctype.py` / `__tests__/*.test.js`

### 添加新 API 端点

1. 在 `api/v1/` 创建或修改文件
2. 使用 `@frappe.whitelist()` + `@handle_exceptions`（+ `@with_transaction` 按需）
3. 返回 `success_response` / `error_response`
4. 如需在 `hooks.py` 注册 `api_methods`

### 数据库变更

1. 修改 DocType JSON → `bench migrate` 自动同步
2. 复杂迁移 → 创建 `patches/` 补丁脚本
3. 在 `hooks.py` 的 `patches` 中注册
