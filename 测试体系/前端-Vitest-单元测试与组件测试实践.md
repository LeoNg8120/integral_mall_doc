积分商城前端采用 **Vitest + Testing Library** 作为单元测试与组件测试的核心技术栈，覆盖了 API 映射层、Zustand Store、React Hook、公共组件、工具函数和页面业务逻辑共 **6 个测试层级、14 个测试文件、68 个测试用例**。本文将从测试架构设计出发，逐层解析每个测试领域的实现模式、Mock 策略和断言技巧，帮助你掌握项目的前端测试范式并快速编写新测试。

Sources: [vitest.config.ts](frontend/vitest.config.ts#L1-L24), [package.json](frontend/package.json#L1-L50)

## 测试架构总览

项目的 Vitest 测试配置在 [`vitest.config.ts`](frontend/vitest.config.ts) 中定义，独立于 Vite 的构建配置（[`vite.config.ts`](frontend/vite.config.ts)），形成"构建与测试配置分离"的清晰边界。核心配置要素如下表所示：

| 配置项 | 值 | 设计意图 |
|--------|------|----------|
| `environment` | `jsdom` | 模拟浏览器 DOM 环境，支持组件渲染测试 |
| `globals` | `true` | 全局注入 `describe`/`it`/`expect`，无需逐文件导入 |
| `setupFiles` | `./src/test/setup.ts` | 加载 `@testing-library/jest-dom` 的自定义匹配器 |
| `include` | `src/**/*.test.{ts,tsx}` | 标准测试文件命名约定，测试就近放置 |
| `coverage.provider` | `v8` | 基于 V8 引擎的原生覆盖率，性能优于 Istanbul |
| `coverage.reporter` | `text, html, lcov` | 终端输出 + HTML 报告 + CI 集成格式 |
| `coverage.reportsDirectory` | `.artifacts/vitest/coverage` | 输出至项目根目录的 `.artifacts` 中，不污染前端源码 |

测试文件的物理组织遵循**就近放置原则**——每个被测模块所在目录下创建 `__tests__` 子文件夹，确保测试与源码的同级可见性。当前测试覆盖的 6 个层级及其文件分布如下：

```
frontend/src/
├── api/__tests__/           ← API 层：5 个文件（请求参数 + 响应映射）
│   ├── application.test.ts
│   ├── notification.test.ts
│   ├── order.test.ts
│   ├── points.test.ts
│   └── review.test.ts
├── components/__tests__/    ← 组件层：1 个文件（DOM 渲染 + 交互）
│   └── status-badge.test.tsx
├── hooks/__tests__/         ← Hook 层：1 个文件（状态管理逻辑）
│   └── use-pagination.test.tsx
├── stores/__tests__/        ← Store 层：2 个文件（状态流转 + 异步行为）
│   ├── auth-store.test.ts
│   └── notification-store.test.ts
├── utils/__tests__/         ← 工具函数层：2 个文件（纯函数边界条件）
│   ├── application.test.ts
│   └── time.test.ts
└── pages/
    ├── admin/__tests__/     ← 页面业务逻辑层：1 个文件
    │   └── user-role-utils.test.ts
    ├── review/__tests__/    ← 页面业务逻辑层：1 个文件
    │   └── review-level.test.ts
    └── settings/__tests__/  ← 页面业务逻辑层：1 个文件
        └── profile-sync.test.ts
```

Sources: [vitest.config.ts](frontend/vitest.config.ts#L1-L24), [setup.ts](frontend/src/test/setup.ts#L1-L2)

## 测试金字塔：从纯函数到异步 Store 的分层策略

项目的前端测试遵循经典测试金字塔原则——底层的纯函数和工具类测试用例最多、运行最快，顶层的异步 Store 集成测试用例较少但验证范围更广。下表展示了各层级的测试特征与用例数量：

| 测试层级 | 测试文件数 | 用例数 | 被测对象特征 | Mock 依赖 | 运行环境 |
|----------|-----------|--------|-------------|-----------|---------|
| 工具函数 | 2 | 9 | 纯函数、无副作用 | 无 | 同步断言 |
| 页面业务逻辑 | 3 | 8 | 纯函数、数据转换 | 无 | 同步断言 |
| Hook | 1 | 7 | React 状态管理 | `renderHook` | jsdom + `act()` |
| 组件 | 1 | 15 | DOM 渲染、属性映射 | `render` / `screen` | jsdom |
| API 层 | 5 | 14 | HTTP 请求参数 + 响应字段映射 | `vi.mock('@/api/client')` | 异步断言 |
| Store 层 | 2 | 15 | Zustand 状态流转 + 异步 Action | `vi.mock('@/api/client')` | jsdom + `act()` |

**纯函数层**（工具函数 + 页面业务逻辑）是完全无副作用的测试，不依赖任何 Mock，执行速度最快。它们验证的是数据转换的正确性——如时间格式化、状态码中文映射、角色编码到 ID 的转换。这些测试构成了金字塔的基座，为上层提供了可信赖的数据处理保障。

**Hook 层与组件层**需要 jsdom 环境来模拟浏览器 API。Hook 测试使用 `@testing-library/react` 的 `renderHook` 来挂载自定义 Hook 并通过 `act()` 包裹状态变更；组件测试则使用 `render` + `screen` 组合来验证 DOM 输出。这两层关注的是 React 的渲染逻辑和状态管理，而非网络请求。

**API 层**测试的核心目标是验证**后端响应字段到前端类型模型的映射正确性**——项目采用"前后端契约驱动"模式，API 层承担了从蛇形命名（`final_points`）到驼峰命名（`points`）的字段转换责任，测试确保这种转换不丢失数据。此层通过 `vi.mock('@/api/client')` 将 HTTP 客户端完全替换为 Mock，使测试专注于映射逻辑。

**Store 层**是最接近集成测试的层级，验证 Zustand Store 的 Action 是否正确地更新状态、调用 API、处理异步流程。同样 Mock 掉 HTTP 客户端后，测试聚焦于"用户操作 → 状态变更"的因果链。

Sources: [time.test.ts](frontend/src/utils/__tests__/time.test.ts#L1-L38), [review-level.test.ts](frontend/src/pages/review/__tests__/review-level.test.ts#L1-L38), [use-pagination.test.tsx](frontend/src/hooks/__tests__/use-pagination.test.tsx#L1-L70), [status-badge.test.tsx](frontend/src/components/__tests__/status-badge.test.tsx#L1-L57), [application.test.ts](frontend/src/api/__tests__/application.test.ts#L1-L97), [notification-store.test.ts](frontend/src/stores/__tests__/notification-store.test.ts#L1-L153)

## API 层测试：契约映射的守门人

API 层是本项目测试覆盖最密集的模块之一，5 个测试文件分别验证 `applicationApi`、`orderApi`、`pointsApi`、`notificationApi`、`reviewApi` 的请求参数构造和响应字段映射。所有 API 测试共享同一套 Mock 模式。

### Mock 模式：替换 HTTP 客户端

项目的 API 模块通过统一的 [`client.ts`](frontend/src/api/client.ts) 发起 HTTP 请求（基于 Axios 封装了 `get`/`post`/`put`/`del` 四个方法）。测试时，**只需 Mock 这一薄层**即可隔离整个网络栈：

```typescript
// 每个 API 测试文件顶部的标准 Mock 声明
vi.mock('@/api/client', () => ({
  get: vi.fn(),
  post: vi.fn(),
  put: vi.fn(),
  del: vi.fn(),
}))
```

这种"模块级 Mock"策略的优势在于：测试不关心 Axios 拦截器、JWT 注入、错误解包等底层逻辑，而是直接控制 `get`/`post` 的返回值来验证 API 层的字段映射。Mock 后通过 `vi.mocked()` 获取类型安全的引用：

```typescript
import { get, post } from '@/api/client'

const mockGet = vi.mocked(get)  // 保留完整类型信息
const mockPost = vi.mocked(post)
```

Sources: [client.ts](frontend/src/api/client.ts#L46-L64), [notification.test.ts](frontend/src/api/__tests__/notification.test.ts#L1-L9)

### 字段映射测试：前后端契约的验证核心

API 层最关键的测试目标是验证**后端蛇形命名到前端驼峰命名的字段映射**。以 `orderApi` 为例，后端返回的 `points_cost` 需要映射为前端的 `points_spent`，嵌套的 `product.id` 需要展平为 `product_id`：

```typescript
// 测试数据模拟后端真实响应结构
vi.mocked(get).mockResolvedValueOnce({
  list: [{
    id: 1,
    order_no: 'ORD-20260416-001',
    points_cost: 200,
    product: { id: 5, name: '测试商品', image: '/uploads/product.png' },
    // ...
  }],
  total: 1,
})

// 断言映射后的前端结构
expect(resp.list[0]).toMatchObject({
  product_id: 5,          // product.id → product_id
  product_name: '测试商品', // product.name → product_name
  points_spent: 200,       // points_cost → points_spent
})
```

`applicationApi` 的测试更进一步，验证了 `rule_type` → `type`、`description` → `reason`、`final_points` → `points` 等多个字段的语义映射，以及详情页对 `ai_score.score` 的提取。这些测试的本质是**活的契约文档**——当后端字段重命名时，测试会立即失败并指出具体的不匹配。

Sources: [order.test.ts](frontend/src/api/__tests__/order.test.ts#L18-L55), [application.test.ts](frontend/src/api/__tests__/application.test.ts#L15-L96)

### 请求参数测试：确保正确的 API 调用

除了响应映射，API 测试还验证请求参数的正确传递。`reviewApi` 的测试检查 `level` 参数是否被正确附加到请求中，`notificationApi` 的测试验证 `page`、`page_size`、`type` 参数是否按预期构造：

```typescript
// 验证请求参数
await notificationApi.getList(1, 20, 'review')
expect(get).toHaveBeenCalledWith('/notifications', {
  page: 1, page_size: 20, type: 'review'
})
```

这种"验证调用参数"的断言模式（`toHaveBeenCalledWith`）与"验证返回值"的断言模式（`expect(resp).toMatchObject(...)`）互补，构成了 API 层的完整测试覆盖。

Sources: [review.test.ts](frontend/src/api/__tests__/review.test.ts#L16-L45), [notification.test.ts](frontend/src/api/__tests__/notification.test.ts#L16-L61)

## Store 层测试：状态流转与异步行为验证

Zustand Store 的测试需要同时处理**同步状态读写**和**异步 API 调用后的状态更新**，是整个测试体系中复杂度最高的层级。

### 测试隔离：beforeEach 重置策略

每个 Store 测试的 `beforeEach` 必须完成两件事：清除所有 Mock 调用记录，以及将 Store 状态重置为初始值。`useAuthStore` 的重置通过 `setState` 直接覆盖：

```typescript
beforeEach(() => {
  vi.clearAllMocks()
  localStorage.clear()
  useAuthStore.setState({
    token: null,
    user: null,
    isAuthenticated: false,
  })
})
```

而 `useNotificationStore` 的重置同样使用 `setState` 覆盖所有状态字段。这种显式重置策略确保每个测试用例都从干净的状态开始，不受前序用例的影响。

Sources: [auth-store.test.ts](frontend/src/stores/__tests__/auth-store.test.ts#L5-L13), [notification-store.test.ts](frontend/src/stores/__tests__/notification-store.test.ts#L22-L32)

### 同步状态测试：auth-store 的角色与权限判断

`useAuthStore` 的 `login`/`logout`/`hasRole`/`hasPermission` 方法都是同步操作，测试模式相对直观。关键的测试场景包括：

- **持久化验证**：`login` 后检查 `localStorage` 是否写入了 `INTegral_mall_token` 和 `INTegral_mall_user`
- **角色判断**：验证 `hasRole('participant')` 在用户拥有该角色时返回 `true`，不拥有时返回 `false`
- **权限短路**：验证 `is_super_admin: true` 的用户调用 `hasPermission('any:permission')` 返回 `true`——这是超级管理员的"全部权限"语义

测试中构造了一个标准的 `mockUser` 对象作为测试夹具，包含 `participant` + `reviewer` 双角色和 `points:view` + `application:create` 双权限，覆盖了最常见的权限判断路径。

Sources: [auth-store.test.ts](frontend/src/stores/__tests__/auth-store.test.ts#L15-L91)

### 异步行为测试：notification-store 的加载与交互

`useNotificationStore` 的 Action（如 `fetchNotifications`、`markAsRead`）是异步函数，测试时需要用 `@testing-library/react` 的 `act()` 包裹以正确处理 React 状态更新：

```typescript
await act(async () => {
  await useNotificationStore.getState().fetchNotifications(1)
})
```

**Loading 状态的中间态测试**是一个值得注意的模式。为了验证请求期间 `loading` 为 `true`，测试使用 `mockImplementationOnce` 返回一个延迟 50ms 的 Promise，在不 `await` 完成前断言 `loading` 状态：

```typescript
const promise = act(async () => {
  await useNotificationStore.getState().fetchNotifications(1)
})
expect(useNotificationStore.getState().loading).toBe(true)  // 中间态
await promise
expect(useNotificationStore.getState().loading).toBe(false) // 终态
```

**`markAsRead` 的"置灰不删除"语义**是另一个精细的测试场景。测试验证标记已读后通知列表长度不变（3 条仍然是 3 条），只有 `is_read` 字段从 `false` 变为 `true`，并且未受影响的通知保持原样。这确保了"软更新"逻辑不会意外删除数据。

Sources: [notification-store.test.ts](frontend/src/stores/__tests__/notification-store.test.ts#L34-L152)

## 组件测试：DOM 渲染与属性映射

### StatusBadge 的参数化测试

[`StatusBadge`](frontend/src/components/status-badge.tsx) 组件是项目中唯一的 UI 组件测试，使用 `test.each` 进行参数化测试来验证状态码到中文文本的映射。10 个状态映射用例通过一个二维数组驱动：

```typescript
const testCases: Array<[string, string]> = [
  ['pending', '待审核'],
  ['ai_reviewing', 'AI 审核中'],
  ['approved', '已通过'],
  ['rejected', '已驳回'],
  // ... 共 10 个
]

test.each(testCases)('"%s" 应显示为 "%s"', (status, expectedText) => {
  render(<StatusBadge status={status} />)
  expect(screen.getByText(expectedText)).toBeInTheDocument()
})
```

`test.each` 的优势在于：当新增一个状态码时，只需在数组中添加一行即可生成新的测试用例，无需编写完整的 `it()` + `render()` + `expect()` 样板代码。此外，`@testing-library/jest-dom` 提供的 `toBeInTheDocument()` 匹配器使得 DOM 断言语义清晰——它来自 [`setup.ts`](frontend/src/test/setup.ts) 中的全局注册。

### 边界条件与覆盖属性

组件测试还覆盖了三个重要的边界条件：

| 场景 | 输入 | 期望行为 | 设计意图 |
|------|------|---------|---------|
| 空字符串回退 | `status=""` | 显示"启用" | 容错：空值不等同于错误 |
| null 值回退 | `status={null}` | 显示"启用" | 防御性编程 |
| 未知状态码 | `status="custom_unknown"` | 显示原始值 | 不吞没未知状态 |
| text 属性覆盖 | `text="审核通过"` | 覆盖默认"已通过" | 支持自定义文案 |

对 `null` 值的测试使用了 `@ts-expect-error` 注释来故意传入 TypeScript 不允许的类型，验证运行时的防御性处理。

Sources: [status-badge.test.tsx](frontend/src/components/__tests__/status-badge.test.tsx#L1-L57)

## Hook 测试：renderHook 与 act 模式

[`usePagination`](frontend/src/hooks/use-pagination.tsx) Hook 的测试展示了 `@testing-library/react` 的 `renderHook` API 在自定义 Hook 测试中的标准用法。核心模式是：

```typescript
const { result } = renderHook(() => usePagination())
expect(result.current.page).toBe(1)

act(() => {
  result.current.setPage(3)
})
expect(result.current.page).toBe(3)
expect(result.current.offset).toBe(20)  // (3-1) * 10
```

**关键测试点**包括：默认参数验证（`pageSize=10`）、自定义初始值（`usePagination(20)`）、派生属性 `offset` 的计算正确性（`(page - 1) * pageSize`），以及各个 setter 方法是否正确触发状态更新。`act()` 的使用是必须的——它确保 React 的批量状态更新在断言前已经完成。

Sources: [use-pagination.test.tsx](frontend/src/hooks/__tests__/use-pagination.test.tsx#L1-L70)

## 工具函数与页面业务逻辑测试

### 纯函数测试：零 Mock、零依赖

工具函数和页面业务逻辑函数都是纯函数，测试最为简洁——无需 Mock、无需 jsdom 渲染、无需 `act()` 包裹，只需构造输入并断言输出。

**[`formatDateTime`](frontend/src/utils/time.ts)** 测试覆盖了 5 种输入变体：ISO 格式时间、无时区后缀时间、`undefined`、`null`、空字符串和无效日期字符串。后四种都应返回 `"-"` 作为占位符。对于时区转换测试，断言使用 `dayjs` 库重新计算期望值而非硬编码，确保测试在不同时区的 CI 环境中也能通过。

**[`getApplicationTypeLabel`](frontend/src/utils/application.ts)** 测试验证了 `bonus`→"增加"、`subtract`→"扣减"、`transfer`→"转移" 的映射，以及未知类型和空输入的回退行为。

Sources: [time.test.ts](frontend/src/utils/__tests__/time.test.ts#L1-L38), [application.test.ts](frontend/src/utils/__tests__/application.test.ts#L1-L18)

### 页面级业务逻辑测试

三个页面级测试文件验证了各自页面的核心业务逻辑函数：

- **[`resolveRoleIds`](frontend/src/utils/user-role.ts)**（用户管理页）：验证角色编码到 ID 的完整映射，以及"角色目录不完整时返回 `null` 避免误清空"的安全策略。这个 `null` 返回值是一个关键的**防误操作设计**——当接口数据未加载完成时，宁可不做操作也不清空用户的角色分配。

- **[`review-level.ts`](frontend/src/pages/review/review-level.ts)** 的四个导出函数（审核页）：验证超级管理员获得双审核队列、`normalizeReviewLevel` 的回退逻辑、从申请状态码推断审核级别、以及审核级别的中文标签映射。

- **[`mergeProfileIntoAuthUser`](frontend/src/pages/settings/profile-sync.ts)**（个人设置页）：验证用户资料响应如何刷新认证态中的展示字段（email、name、avatar_url、roles、groups、available_points），确保合并逻辑不会遗漏任何字段。

Sources: [user-role-utils.test.ts](frontend/src/pages/admin/__tests__/user-role-utils.test.ts#L1-L37), [review-level.test.ts](frontend/src/pages/review/__tests__/review-level.test.ts#L1-L38), [profile-sync.test.ts](frontend/src/pages/settings/__tests__/profile-sync.test.ts#L1-L40)

## 运行与 CI 集成

### 本地运行命令

项目提供了三个层级的测试运行入口：

| 命令 | 用途 | 适用场景 |
|------|------|---------|
| `pnpm test:unit` | 单次运行所有单元测试 | CI 流水线、提交前验证 |
| `pnpm test:unit:watch` | 监听模式，文件变更自动重跑 | 开发时实时反馈 |
| `pnpm test:unit:coverage` | 运行测试并生成覆盖率报告 | 定期检查覆盖盲区 |

通过 Makefile 顶层入口 `make test-frontend` 也可以运行，底层调用 [`scripts/test-frontend.sh`](scripts/test-frontend.sh)，该脚本会将 Vitest 输出同时写入 `.artifacts/reports/frontend-vitest-latest.txt` 报告文件，方便 CI 环境下的测试报告归档。

### 覆盖率配置

覆盖率使用 V8 引擎原生 provider（`@vitest/coverage-v8`），报告输出到项目根目录的 `.artifacts/vitest/coverage/`，支持三种格式：终端文本输出（开发快速查看）、HTML 可视化报告（详细定位未覆盖行）、LCOV 格式（CI/CD 集成使用）。`.artifacts` 目录的设计遵循"构建产物不入库"原则，避免测试报告污染 Git 仓库。

Sources: [test-frontend.sh](scripts/test-frontend.sh#L1-L40), [package.json](frontend/package.json#L6-L16), [vitest.config.ts](frontend/vitest.config.ts#L17-L22)

## 编写新测试的模式指南

### 按被测对象类型选择模板

根据你要测试的模块类型，选择对应的测试模板和依赖：

**纯函数 / 工具类**（最简单）：
```typescript
import { describe, expect, it } from 'vitest'
import { yourFunction } from '../your-module'

describe('yourFunction', () => {
  it('handles normal input', () => {
    expect(yourFunction('input')).toBe('expected')
  })

  it('handles edge case', () => {
    expect(yourFunction(undefined)).toBe('-')
  })
})
```

**API 模块**（需要 Mock `client`）：
```typescript
import { beforeEach, describe, expect, it, vi } from 'vitest'
import { yourApi } from '@/api/your-api'
import { get } from '@/api/client'

vi.mock('@/api/client', () => ({
  get: vi.fn(),
  post: vi.fn(),
}))

describe('yourApi', () => {
  beforeEach(() => {
    vi.mocked(get).mockReset()
  })

  it('maps response fields correctly', async () => {
    vi.mocked(get).mockResolvedValueOnce({ backend_field: 'value' })
    const result = await yourApi.getList()
    expect(result).toMatchObject({ frontendField: 'value' })
  })
})
```

**Zustand Store**（需要 Mock + `act`）：
```typescript
import { act } from '@testing-library/react'
import { useYourStore } from '../your-store'
import * as client from '@/api/client'

vi.mock('@/api/client', () => ({
  get: vi.fn(),
  post: vi.fn(),
}))

describe('your-store', () => {
  beforeEach(() => {
    vi.clearAllMocks()
    useYourStore.setState({ /* 初始状态 */ })
  })

  it('updates state after async action', async () => {
    vi.mocked(client.get).mockResolvedValueOnce({ data: 'value' })
    await act(async () => {
      await useYourStore.getState().yourAction()
    })
    expect(useYourStore.getState().yourField).toBe('expected')
  })
})
```

### 测试命名的双语约定

项目中存在两种命名风格：部分测试使用中文断言描述（如 `'login 应设置 token 和用户信息'`），部分使用英文（如 `'maps list response fields'`）。这反映了测试编写时的渐进演进。对于新测试，建议保持与同目录已有测试一致的风格；对于全新模块的测试，推荐使用中文描述以保持与项目文档体系的一致性。

Sources: [auth-store.test.ts](frontend/src/stores/__tests__/auth-store.test.ts#L29-L37), [application.test.ts](frontend/src/api/__tests__/application.test.ts#L1-L14), [notification-store.test.ts](frontend/src/stores/__tests__/notification-store.test.ts#L1-L10)

## 延伸阅读

- 如需了解后端测试策略（Go 单元测试的 Mock 模式与覆盖范围），参阅 [后端单元测试策略：Mock 辅助与覆盖范围](22-hou-duan-dan-yuan-ce-shi-ce-lue-mock-fu-zhu-yu-fu-gai-fan-wei)。
- 如需了解前端 E2E 测试（147 用例的 Playwright 测试组织），参阅 [Playwright E2E 测试：147 用例的组织与执行](23-playwright-e2e-ce-shi-147-yong-li-de-zu-zhi-yu-zhi-xing)。
- 如需了解 API 层测试验证的契约映射的源头——前后端类型生成机制，参阅 [前后端契约驱动开发：goctl 类型生成与漂移检查](16-qian-hou-duan-qi-yue-qu-dong-kai-fa-goctl-lei-xing-sheng-cheng-yu-piao-yi-jian-cha)。
- 如需了解 Zustand Store 的完整设计（被测对象的架构上下文），参阅 [Zustand 认证状态管理：登录持久化与权限判断](17-zustand-ren-zheng-zhuang-tai-guan-li-deng-lu-chi-jiu-hua-yu-quan-xian-pan-duan)。