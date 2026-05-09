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

项目 Composable 使用 `createResource`（frappe-ui）管理 API 资源，配合自定义 `frappeRequest`。所有 API 资源均设 `auto: false` 手动触发。

```javascript
// src/composables/useMyFeature.js
import { ref, computed } from 'vue'
import { createResource } from 'frappe-ui'

// API 基础路径
const API_BASE = 'product_sales_planning.api.v1.my_api'

export function useMyFeature(storeId, taskId) {
    const baseParams = { task_id: taskId, store_id: storeId }

    // ==================== API 资源 ====================

    const listResource = createResource({
        url: `${API_BASE}.get_list`,
        makeParams: () => baseParams,
        auto: false,
        transform: (res) => {
            // 兼容 res.message 或直接 res 两种返回格式
            const data = res?.message || res
            return data?.status === 'success' ? data.data : null
        }
    })

    const updateResource = createResource({
        url: `${API_BASE}.update`,
        auto: false
    })

    // ==================== 状态 ====================

    const loading = computed(() => listResource.loading || updateResource.loading)
    const data = computed(() => listResource.data || [])

    // ==================== 方法 ====================

    const fetchData = async () => {
        await listResource.fetch()
    }

    const updateData = async (params) => {
        await updateResource.submit(params)
        await fetchData() // 刷新列表
    }

    return {
        // 状态
        loading,
        data,
        // 资源（供模板使用 .loading / .error）
        listResource,
        updateResource,
        // 方法
        fetchData,
        updateData
    }
}
```

项目现有 Composable（5 个）：
- `useApproval.js` — 审批流程（5 个 createResource：status/history/submit/withdraw/approve）
- `useFixedReport.js` — 固定报表
- `useNotification.js` — 通知系统
- `usePermission.js` — 权限检查
- `useStoreDetail.js` — 店铺详情（最复杂，含数据加载/筛选/保存/导出/列设置/选择状态）

## 3. API 调用层

### 方式1：createResource（推荐，frappe-ui 响应式资源）

```javascript
import { createResource } from 'frappe-ui'

const resource = createResource({
    url: 'product_sales_planning.api.v1.my_api.get_data',
    makeParams: () => ({ store_id: 'STORE001' }),
    auto: false,
    transform: (res) => {
        const data = res?.message || res
        return data?.status === 'success' ? data.data : null
    }
})

// 手动触发
await resource.fetch()
// 或提交参数
await resource.submit({ name: 'xxx', value: 'yyy' })
```

### 方式2：frappe.call（简单场景，一次性调用）

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

### 方式3：自定义 frappeRequest（需要精细控制请求时）

```javascript
import frappeRequest from '@/utils/frappeRequest'

const response = await frappeRequest({
    url: 'product_sales_planning.api.v1.my_api.update',
    method: 'POST',
    params: { name: 'xxx', value: 'yyy' }
})
```

### frappeRequest 核心机制

- CSRF token 三源获取：cookie → `window.csrf_token` → `window.frappe.csrf_token`
- CSRF 错误自动刷新 token 并重试一次
- 自动前缀 `/api/method/`（不以 `/` 或 `http` 开头的 URL）
- GET 请求参数用 `URLSearchParams`，POST 用 JSON body
- 在 `main.js` 中通过 `setConfig('resourceFetcher', frappeRequest)` 全局替换 frappe-ui 默认请求

### API 模块封装

```javascript
// src/api/myApi.js
export const myApi = {
    getList: (params) => frappe.call({
        method: 'product_sales_planning.api.v1.my_api.get_list',
        args: params
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
// 透视分析 composable
import { usePivotAnalysis } from '@/composables/usePivotAnalysis'
```

相关组件（`components/pivot/`）：PivotLayoutBuilder、PivotTable、S2PivotTable、PivotChart、DimensionChip、ValueFieldChip、DimensionConfigPanel、PivotExportButton

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
- `afterEach` — 设置 document.title 为 `{meta.title} - 计划填报系统`

布局结构（`MainLayout.vue`）：
- Sidebar（可折叠）+ TopBar + ErrorBoundary
- DataAnalysis 页面使用 `keep-alive` 缓存
- Suspense 提供加载 fallback
- 全局 NotificationModal 通知弹窗

## 6. 权限检查

```javascript
import { usePermission } from '@/composables/usePermission'
import { permissionStore } from '@/stores/permissionStore'

// Composable 方式
const { hasRole, canAccessRoute, visibleMenus } = usePermission()
if (hasRole('System Manager')) { ... }
if (canAccessRoute('ApprovalCenter')) { ... }

// 直接使用 permissionStore（注意：是 Vue reactive 对象，不是 Pinia store）
if (permissionStore.hasRole('System Manager')) { ... }
if (permissionStore.canAccessRoute('ApprovalCenter')) { ... }
```

**重要**：`permissionStore` 是 Vue `reactive`/`readonly` 实现的响应式对象，不是 Pinia store。直接作为对象使用（`permissionStore.hasRole()`），不需要加括号调用。

权限配置在 `src/config/permissions.js`，包含 `pagePermissions`、`getVisibleMenus()`、`canAccessRoute()` 等工具函数。

数据源优先级：`window.boot`（由 `planning.py` 注入）> `sessionStorage` 缓存（TTL 5 分钟）> API 请求。

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

### 测试文件位置

- `pages/__tests__/DataAnalysis.test.js`
- `components/store-detail/__tests__/FilterPanel.test.js`
- `components/store-detail/__tests__/ApprovalDialog.property.test.js`
- `components/fixed-report/__tests__/FixedReportTable.test.js`
- `composables/__tests__/useFixedReport.test.js`
- `router/guards/__tests__/permissionGuard.property.test.js`
- `utils/__tests__/cacheManager.test.ts`
