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
│   ├── api/v1/                      # REST API (26个文件)
│   ├── planning_system/doctype/     # DocType (25个模块)
│   ├── services/                    # 业务逻辑层 (8个服务)
│   ├── utils/                       # 工具函数 (14个)
│   ├── overrides/                   # Frappe 框架覆写 (OAuth/用户/Workspace)
│   ├── patches/                     # 数据库补丁 (6个)
│   ├── commands/                    # bench 自定义命令
│   ├── www/                         # Frappe 门户页面 (5个)
│   └── constants.py                 # 状态常量、错误码、分页限制
├── frontend/                        # Vue 3 前端
│   ├── src/pages/                   # 页面组件 (7个)
│   ├── src/components/              # 通用组件 (30+)
│   ├── src/composables/             # 组合式函数 (6个)
│   ├── src/stores/                  # 响应式状态 (permissionStore, Vue reactive)
│   ├── src/api/                     # API 调用层 (2个)
│   ├── src/config/                  # 权限配置
│   └── src/router/                  # 路由 + 守卫
└── public/planning/                 # 构建输出
```

## 核心架构模式

### 1. 请求流

```
Browser (/planning/*)
  → Vite dev proxy (dev) / Nginx (prod)
  → Frappe www/planning.py (SPA shell)
  → Vue Router → Page → Composable → API
  → frappe.call() → /api/method/product_sales_planning.api.v1.*
  → MariaDB / ClickHouse
```

### 2. API 装饰器链

优先使用 `@api_endpoint` 组合装饰器，它封装了权限检查、异常处理和事务管理。仅在需要精细控制时才拆分为 `@handle_exceptions` + `@require_permission`。

```python
# 推荐写法
@frappe.whitelist()
@api_endpoint(doctype="DocType Name", perm_type="write", require_transaction=True)
def my_api():
    pass

# 需要精细控制时拆分
@frappe.whitelist()
@handle_exceptions
@require_permission("DocType Name", "read")
def my_api():
    pass
```

### 3. 统一响应格式

所有 API 必须使用 `success_response` 或 `error_response`，不要返回裸数据或 frappe 原始响应。

```python
from product_sales_planning.utils.response_utils import success_response, error_response
from product_sales_planning.constants import ErrorCode

return success_response(data={"items": [...]})
return error_response(message="操作失败", error_code=ErrorCode.VALIDATION_ERROR)
```

### 4. 缓存失效

`doc_events` 注册在 `hooks.py` 中，当 `Commodity Schedule` 或 `Approval Instance` 变更时自动清除分析缓存。`Permission Config` 变更时清除权限缓存。新增 DocType 如需缓存失效行为，在 `utils/cache_hooks.py` 添加并注册到 `hooks.py → doc_events`。

### 5. 前端 Composable 模式

页面逻辑提取到 `composables/use*.js`，保持页面组件精简。API 调用封装到 `api/` 层。这很重要——项目已有 6 个 composable，新功能应遵循相同模式。

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

## 核心常量速查

| 常量 | 值 |
|------|------|
| `ApprovalStatus` | 待审批 / 已通过 / 已驳回 |
| `SubmissionStatus` | 未开始 / 已提交 |
| `TaskStatus` | 未开始 / 开启中 / 已完成 / 已关闭 |
| `MAX_PAGE_SIZE` | 1000 |
| `DEFAULT_PAGE_SIZE` | 20 |
| `MAX_IMPORT_ROWS` | 10000 |
| `MAX_EXPORT_ROWS` | 50000 |
| `DEFAULT_PLAN_MONTHS` | 5 |
| `ALLOWED_FILE_EXTENSIONS` | .xlsx, .xls |

## 路由映射

| 路径 | 页面 | 说明 |
|------|------|------|
| `/` | PlanningDashboard | 计划看板 |
| `/approval-center` | ApprovalCenter | 审批中心 |
| `/mechanism` | MechanismManagement | 机制管理 |
| `/data-analysis` | DataAnalysis | 数据分析（含透视表） |
| `/data-view` | DataView → redirect | 数据查看 |
| `/store-detail/:storeId/:taskId` | StoreDetail | 店铺详情 |
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
2. 使用 `@frappe.whitelist()` + 装饰器链
3. 返回 `success_response` / `error_response`
4. 如需在 `hooks.py` 注册 `api_methods`

### 数据库变更

1. 修改 DocType JSON → `bench migrate` 自动同步
2. 复杂迁移 → 创建 `patches/` 补丁脚本
3. 在 `hooks.py` 的 `patches` 中注册
