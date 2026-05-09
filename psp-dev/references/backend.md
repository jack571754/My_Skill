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

### 基础查询 API

```python
# api/v1/my_api.py
"""
My API - XXX相关API接口
"""
import frappe
from product_sales_planning.utils.response_utils import success_response, error_response
from product_sales_planning.utils.api_decorators import (
    handle_exceptions,
    require_permission,
    api_endpoint
)
from product_sales_planning.constants import MAX_PAGE_SIZE, DEFAULT_PAGE_SIZE

@frappe.whitelist()
@api_endpoint(doctype="My DocType", perm_type="read")
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

### 写操作 API（带权限和事务）

```python
@frappe.whitelist()
@api_endpoint(doctype="My DocType", perm_type="write", require_transaction=True)
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
@api_endpoint(doctype="My DocType", perm_type="create", require_transaction=True)
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

## 3. 服务层开发

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

## 4. ClickHouse 查询

```python
from product_sales_planning.utils.clickhouse_client import execute_query, execute_query_df

# 执行查询返回列表
result = execute_query("""
    SELECT
        store_id,
        product_code,
        SUM(quantity) as total_qty
    FROM sales_data
    WHERE date >= '2025-01-01'
    GROUP BY store_id, product_code
    ORDER BY total_qty DESC
    LIMIT 100
""")

# 执行查询返回 DataFrame
df = execute_query_df("SELECT * FROM sales_data LIMIT 100")

# 测试连接
from product_sales_planning.utils.clickhouse_client import test_connection
test_connection()
```

项目有两个分析路径：
- `data_analysis.py` — 直接查询 MariaDB
- `data_analysis_clickhouse.py` — 查询 ClickHouse（高性能分析）

其他 ClickHouse 工具：
- `clickhouse_sales.py` — 销售数据查询
- `clickhouse_materialized_views.py` — 物化视图管理

## 5. 缓存服务

缓存服务是面向分析场景的专用缓存，不是通用 key-value 缓存。通过 `get_cache_service()` 获取单例实例。

```python
from product_sales_planning.utils.cache_service import get_cache_service

cache = get_cache_service()

# 设置/获取分析数据
cache.set_analysis_data(task_id, store_id, data)
data = cache.get_analysis_data(task_id, store_id)

# 设置/获取筛选选项
cache.set_filter_options(task_id, options)
options = cache.get_filter_options(task_id)

# 设置/获取统计信息
cache.set_stats_data(task_id, stats)
stats = cache.get_stats_data(task_id)

# 失效操作
cache.invalidate_by_store(store_id)
cache.invalidate_by_task(task_id)
cache.invalidate_user_cache(user)
cache.invalidate_all()
```

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
