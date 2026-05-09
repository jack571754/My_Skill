# 后端开发详细模板

## 目录

1. [DocType 开发](#1-doctype-开发)
2. [API 开发](#2-api-开发)
3. [服务层开发](#3-服务层开发)
4. [ClickHouse 查询](#4-clickhouse-查询)
5. [缓存服务](#5-缓存服务)
6. [后端测试](#6-后端测试)

## 1. DocType 开发

### DocType 文件结构

```
doctype/my_doctype/
├── __init__.py
├── my_doctype.py        # 主要逻辑
├── my_doctype.json      # DocType 配置
├── my_doctype.js        # Frappe 前端脚本（可选）
└── test_my_doctype.py   # 测试文件
```

### DocType JSON 模板

```json
{
  "doctype": "DocType",
  "name": "My DocType",
  "module": "planning system",
  "autoname": "field:code",
  "title_field": "name1",
  "fields": [
    {
      "fieldname": "code",
      "fieldtype": "Data",
      "label": "编码",
      "unique": 1,
      "bold": 1
    },
    {
      "fieldname": "name1",
      "fieldtype": "Data",
      "label": "名称",
      "in_list_view": 1
    },
    {
      "fieldname": "status",
      "fieldtype": "Select",
      "label": "状态",
      "options": "\n未开始\n开启中\n已完成\n已关闭",
      "default": "未开始"
    }
  ],
  "permissions": [
    {
      "role": "System Manager",
      "read": 1, "write": 1, "create": 1, "delete": 1
    }
  ]
}
```

### DocType Python 类

```python
from frappe.model.document import Document

class MyDocType(Document):
    def validate(self):
        """验证钩子 - 保存前数据校验"""
        pass

    def before_save(self):
        """保存前钩子 - 设置默认值、派生字段"""
        pass

    def on_update(self):
        """更新后钩子 - 清缓存、触发通知"""
        pass

    def has_permission(self, ptype="read", user=None):
        """自定义权限检查（覆盖 Frappe 默认行为）"""
        if user == "Administrator":
            return True
        from product_sales_planning.services.permission_config_service import PermissionConfigService
        return PermissionConfigService.check_permission(user, self.name, ptype)
```

## 2. API 开发

### 装饰器使用

项目提供两个装饰器，**不存在 `@api_endpoint` 组合装饰器**：

```python
from product_sales_planning.utils.api_decorators import handle_exceptions, with_transaction
```

- `@handle_exceptions` — 统一异常处理，捕获 ValidationError/PermissionError/DoesNotExistError/通用异常，返回 `error_response`
- `@with_transaction` — 事务管理，自动 `begin`/`commit`/`rollback`

### 基础查询 API

```python
# api/v1/my_api.py
"""
My API - XXX相关API接口
"""
import frappe
from product_sales_planning.utils.response_utils import success_response, error_response
from product_sales_planning.utils.api_decorators import handle_exceptions
from product_sales_planning.constants import MAX_PAGE_SIZE, DEFAULT_PAGE_SIZE, ErrorCode

@frappe.whitelist()
@handle_exceptions
def get_list(filters=None, page=1, page_size=DEFAULT_PAGE_SIZE):
    """获取列表数据"""
    filters = frappe.parse_json(filters) if filters else {}
    page = int(page)
    page_size = min(int(page_size), MAX_PAGE_SIZE)

    data = frappe.get_all(
        "My DocType",
        filters=filters,
        fields=["name", "code", "name1", "status"],
        order_by="modified desc",
        start=(page - 1) * page_size,
        page_length=page_size
    )

    total = frappe.db.count("My DocType", filters)

    return success_response(data={
        "data": data,
        "total": total,
        "page": page,
        "page_size": page_size
    })
```

### 写操作 API（带事务）

```python
@frappe.whitelist()
@handle_exceptions
@with_transaction
def update_data(name, **kwargs):
    """更新数据"""
    doc = frappe.get_doc("My DocType", name)

    for key, value in kwargs.items():
        if hasattr(doc, key) and key not in ("doctype", "name"):
            setattr(doc, key, value)

    doc.save(ignore_permissions=True)
    return success_response(data=doc.as_dict())
```

### 批量操作 API

```python
@frappe.whitelist()
@handle_exceptions
@with_transaction
def batch_create(items):
    """批量创建"""
    items = frappe.parse_json(items)
    results = []
    errors = []

    for item in items:
        try:
            doc = frappe.new_doc("My DocType")
            doc.update(item)
            doc.insert(ignore_permissions=True)
            results.append(doc.name)
        except Exception as e:
            errors.append({"item": item, "error": str(e)})

    return success_response(data={
        "created": results,
        "errors": errors,
        "total": len(items),
        "success_count": len(results)
    })
```

### 带参数验证的 API

```python
from product_sales_planning.utils.validation_utils import validate_required_params, validate_positive_integer

@frappe.whitelist()
@handle_exceptions
def get_detail(name):
    """获取详情"""
    validate_required_params({"name": name}, ["name"])

    doc = frappe.get_doc("My DocType", name)
    return success_response(data=doc.as_dict())
```

## 3. 服务层开发

服务层有两种风格——类风格和模块函数风格，根据复杂度选择：

### 类风格（适合复杂业务，如 CommodityScheduleService）

```python
# services/my_service.py
"""XXX服务层 - 将业务逻辑从 API 层分离"""
import frappe
from frappe import _

class MyService:
    @staticmethod
    def get_data(param1, param2):
        """获取数据"""
        if not param1:
            frappe.throw(_("参数不能为空"))

        data = frappe.get_all("My DocType", filters={...})
        return {"data": data, "total": len(data)}

    @staticmethod
    def batch_operation(items):
        """批量操作（带事务）"""
        try:
            success_count = 0
            errors = []

            for item in items:
                try:
                    doc = frappe.new_doc("My DocType")
                    doc.update(item)
                    doc.insert(ignore_permissions=True)
                    success_count += 1
                except Exception as e:
                    errors.append(f"项目失败: {str(e)}")

            frappe.db.commit()
            return {"status": "success", "count": success_count, "errors": errors}
        except Exception as e:
            frappe.db.rollback()
            frappe.log_error("批量操作失败", str(e))
            raise
```

### 模块函数风格（适合通知、配置等扁平逻辑，如 approval_service.py）

```python
# services/my_notification_service.py
"""通知服务"""
import frappe

def send_notification(user, title, message, notification_type="info"):
    """发送通知"""
    # 直接使用模块级函数
    pass
```

### 服务层常用工具

```python
from product_sales_planning.utils.date_utils import get_date_range_filter, get_month_first_day
from product_sales_planning.utils.validation_utils import validate_required_params, validate_positive_integer, validate_doctype_exists
from product_sales_planning.constants import DEFAULT_PLAN_MONTHS
```

## 4. ClickHouse 查询

### 连接配置

ClickHouse 通过 HTTP 协议连接，配置优先级：`frappe.conf` > `DEFAULT_CLICKHOUSE_CONFIG`。

```python
from product_sales_planning.utils.clickhouse_client import get_client

# 获取客户端实例（模块级单例）
client = get_client()
```

### 安全的表名引用

```python
from product_sales_planning.utils.clickhouse_client import format_clickhouse_table_ref, get_clickhouse_table_ref

# 安全引用表名（自动添加反引号）
table = format_clickhouse_table_ref("smartbimpp.sales_data")  # → `smartbimpp`.`sales_data`

# 按配置键优先级查找表名
table = get_clickhouse_table_ref(["clickhouse_sales_table", "default_sales_table"], "sales_data")
```

### 查询示例

```python
# 执行查询返回列表
result = client.query("""
    SELECT
        store_id,
        product_code,
        SUM(quantity) as total_qty
    FROM `smartbimpp`.`sales_data`
    WHERE date >= '2025-01-01'
    GROUP BY store_id, product_code
    ORDER BY total_qty DESC
    LIMIT 100
""")

# 执行查询返回 DataFrame
df = client.query_df("SELECT * FROM `smartbimpp`.`sales_data` LIMIT 100")
```

项目有两个分析路径：
- `data_analysis.py` — 直接查询 MariaDB
- `data_analysis_clickhouse.py` — 查询 ClickHouse（高性能分析）

其他 ClickHouse 工具：
- `clickhouse_sales.py` — 销售数据查询
- `clickhouse_materialized_views.py` — 物化视图管理

## 5. 缓存服务

`AnalysisCacheService` 是面向分析场景的专用 Redis 缓存，通过 `frappe.cache()` 连接，不是通用 key-value 缓存。

### 缓存 TTL

| 类型 | TTL | 说明 |
|------|-----|------|
| 筛选选项 | 10 分钟 | `FILTER_OPTIONS_TTL = 600` |
| 统计数据 | 5 分钟 | `STATS_TTL = 300` |
| 分析数据 | 3 分钟 | `DATA_TTL = 180` |

### 使用方式

```python
from product_sales_planning.utils.cache_service import get_cache_service

cache = get_cache_service()

# 设置/获取分析数据
cache.set_analysis_data(task_id, store_id, data)
data = cache.get_analysis_data(task_id, store_id)

# 失效操作
cache.invalidate_by_store(store_id)
cache.invalidate_by_task(task_id)
cache.invalidate_user_cache(user)
cache.invalidate_all()
```

### 降级策略

所有缓存操作都使用 `_safe_*` 方法包装，Redis 不可用时返回 `None`/`False`，不抛异常。调用方应将 `None` 视为缓存未命中。

### 缓存键结构

- 筛选选项：`psp_analysis:filter_options:{user_id}`
- 分析数据：`psp_analysis:analysis:{user_id}:{dimension}:{filters_hash}:{page}:{page_size}`
- 统计数据：`psp_analysis:stats:{user_id}:{filters_hash}`

缓存失效钩子在 `utils/cache_hooks.py`，通过 `hooks.py → doc_events` 自动触发。

## 6. 后端测试

```python
from frappe.tests.utils import FrappeTestCase

class TestMyDocType(FrappeTestCase):
    def setUp(self):
        self.test_data = {
            "doctype": "My DocType",
            "code": "TEST001",
            "name1": "测试数据"
        }

    def test_create(self):
        doc = frappe.get_doc(self.test_data)
        doc.insert()
        self.assertEqual(doc.name, "TEST001")

    def tearDown(self):
        frappe.delete_doc("My DocType", "TEST001")
```

运行命令：`bench --site <site> run-tests --app product_sales_planning --module product_sales_planning.planning_system.doctype.my_doctype`
