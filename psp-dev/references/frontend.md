# 前端开发详细模板

## 目录

1. [页面组件模板](#1-页面组件模板)
2. [Composable 模板](#2-composable-模板)
3. [API 调用层](#3-api-调用层)
4. [表格引擎选择](#4-表格引擎选择)
5. [路由配置](#5-路由配置)
6. [权限检查](#6-权限检查)
7. [前端测试](#7-前端测试)

## 1. 页面组件模板

```vue
<!-- src/pages/MyPage.vue -->
<template>
    <div class="min-h-screen bg-page-gradient p-3 md:p-6">
        <div class="mx-auto max-w-content w-full space-y-6">
            <!-- 头部 -->
            <div class="rounded-lg border border-[var(--color-border-soft)] bg-[var(--color-bg-card)] p-4 shadow-card">
                <div class="flex items-center justify-between">
                    <h1 class="text-h3 m-0">{{ title }}</h1>
                    <Button variant="solid" theme="blue" size="sm" @click="handleAction">
                        <template #prefix><FeatherIcon name="plus" class="h-3.5 w-3.5" /></template>
                        添加
                    </Button>
                </div>
            </div>

            <!-- 内容 -->
            <div class="rounded-lg border border-[var(--color-border-soft)] bg-[var(--color-bg-card)] p-4">
                <!-- 内容区域 -->
            </div>
        </div>
    </div>
</template>

<script setup>
import { ref, computed, onMounted, watch } from 'vue'
import { useRouter, useRoute } from 'vue-router'
import { Button, FeatherIcon } from 'frappe-ui'

// Props
const props = defineProps({
    propA: { type: String, default: '' }
})

// Emits
const emit = defineEmits(['change', 'submit'])

// Refs
const title = ref('页面标题')
const loading = ref(false)
const data = ref([])

// Computed
const filteredData = computed(() => data.value.filter(...))

// Methods
const handleAction = async () => {
    loading.value = true
    try {
        const result = await frappe.call({
            method: 'product_sales_planning.api.v1.my_api.get_list',
            args: { ... }
        })
        data.value = result.message?.data || []
    } catch (error) {
        console.error('操作失败:', error)
    } finally {
        loading.value = false
    }
}

// Watchers & Lifecycle
watch(() => props.propA, (newVal) => { ... })
onMounted(() => { handleAction() })
</script>

<style scoped>
:deep(.custom-class) {
    /* 深度样式 */
}
</style>
```

## 2. Composable 模板

```javascript
// src/composables/useMyFeature.js
import { ref, computed } from 'vue'

export function useMyFeature(initialValue) {
    const data = ref(initialValue)
    const loading = ref(false)
    const error = ref(null)

    const hasData = computed(() => data.value && data.value.length > 0)

    const fetchData = async (params) => {
        loading.value = true
        error.value = null
        try {
            const result = await frappe.call({
                method: 'product_sales_planning.api.v1.my_api.get_data',
                args: params
            })
            data.value = result.message?.data || []
            return data.value
        } catch (e) {
            error.value = e
            throw e
        } finally {
            loading.value = false
        }
    }

    return { data, loading, error, hasData, fetchData }
}
```

项目现有 Composable（6个）：
- `useApproval.js` — 审批流程
- `useFixedReport.js` — 固定报表
- `useNotification.js` — 通知系统
- `usePermission.js` — 权限检查
- `usePivotAnalysis.js` — 透视分析
- `useStoreDetail.js` — 店铺详情

## 3. API 调用层

### 方式1：frappe.call（推荐用于标准 API）

```javascript
const result = await frappe.call({
    method: 'product_sales_planning.api.v1.my_api.get_list',
    args: {
        filters: JSON.stringify({ status: 'active' }),
        page: 1,
        page_size: 50
    }
})
const data = result.message?.data || []
```

### 方式2：frappeRequest（frappe-ui，用于自定义请求）

```javascript
import { frappeRequest } from 'frappe-ui'

const response = await frappeRequest({
    url: '/api/method/product_sales_planning.api.v1.my_api.update',
    method: 'POST',
    data: { name: 'xxx', value: 'yyy' }
})
```

### API 模块封装（推荐）

```javascript
// src/api/myApi.js
export const myApi = {
    getList: (params) => frappe.call({
        method: 'product_sales_planning.api.v1.my_api.get_list',
        args: params
    }),

    update: (data) => frappeRequest({
        url: '/api/method/product_sales_planning.api.v1.my_api.update',
        method: 'POST',
        data
    })
}
```

## 4. 表格引擎选择

项目使用三种表格引擎，根据场景选择：

### Handsontable — 通用可编辑表格

用于店铺详情页的商品数据编辑，支持单元格级别的编辑和验证。

```javascript
import { HotTable } from '@handsontable/vue3'
import 'handsontable/dist/handsontable.full.min.css'
```

### @antv/s2 — 透视分析表

用于数据分析页的透视表展示，支持多维度交叉分析。

```javascript
import { S2PivotTable } from '@/components/pivot/S2PivotTable.vue'
// 透视分析 composable
import { usePivotAnalysis } from '@/composables/usePivotAnalysis'
```

相关组件（`components/pivot/`）：
- `PivotLayoutBuilder.vue` — 透视布局构建器
- `PivotTable.vue` / `S2PivotTable.vue` — 透视表
- `PivotChart.vue` — 透视图表
- `DimensionChip.vue` / `ValueFieldChip.vue` — 维度/指标芯片
- `DimensionConfigPanel.vue` — 维度配置面板
- `PivotExportButton.vue` — 导出按钮

### @visactor/vtable — 高性能数据展示

用于大数据量场景的只读或轻编辑表格。

```javascript
import { VTable } from '@visactor/vtable'
```

## 5. 路由配置

```javascript
// src/router.js
const routes = [
    {
        path: '/my-page/:id?',
        name: 'MyPage',
        component: () => import('./pages/MyPage.vue'),
        meta: {
            requiresAuth: true,
            title: '我的页面'
        }
    }
]
```

路由守卫：
- 全局：`createPermissionGuard()` — 检查页面权限
- 路由级：`beforeEnter` — 验证参数（如 StoreDetail 的 storeId/taskId）

## 6. 权限检查

```javascript
import { usePermission } from '@/composables/usePermission'
import { permissionStore } from '@/stores/permissionStore'

// Composable 方式
const { hasRole, canAccessRoute, visibleMenus } = usePermission()
if (hasRole('System Manager')) { ... }
if (canAccessRoute('ApprovalCenter')) { ... }

// 直接使用 permissionStore 对象（注意：不是 Pinia，是 Vue reactive 对象，不需要函数调用）
if (permissionStore.hasRole('System Manager')) { ... }
if (permissionStore.canAccessRoute('ApprovalCenter')) { ... }
```

权限配置在 `src/config/permissions.js`，包含 `pagePermissions`、`getVisibleMenus()`、`canAccessRoute()` 等工具函数。

注意：`permissionStore` 是 Vue `reactive`/`readonly` 实现的响应式对象，不是 Pinia store，因此直接作为对象使用（`permissionStore.hasRole()`），不需要加括号调用。

## 7. 前端测试

### 单元测试

```javascript
// __tests__/MyComponent.test.js
import { describe, it, expect } from 'vitest'
import { mount } from '@vue/test-utils'
import MyComponent from './MyComponent.vue'

describe('MyComponent', () => {
    it('renders correctly', () => {
        const wrapper = mount(MyComponent, {
            props: { title: 'Test' }
        })
        expect(wrapper.text()).toContain('Test')
    })
})
```

### 属性测试（property-based）

```javascript
// __tests__/MyComponent.property.test.js
import { describe, it } from 'vitest'
import fc from 'fast-check'

describe('MyComponent', () => {
    it('handles all valid inputs', () => {
        fc.assert(fc.property(
            fc.string({ minLength: 0, maxLength: 100 }),
            (input) => {
                // 测试逻辑
            }
        ))
    })
})
```

运行命令：`cd frontend && npm run test src/path/to/test.js`
