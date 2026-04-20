积分商城前端的认证体系围绕一个**轻量但职责清晰**的 Zustand store 展开。它不依赖任何中间件库或复杂的持久化插件——仅凭 `localStorage` 读写和 `create` API 即可完成 JWT 令牌存储、用户信息缓存、角色/权限判断三大核心任务。本文将从 store 结构设计出发，逐层剖析登录持久化机制、权限判断策略，以及 store 如何与路由守卫、侧边栏菜单、API 客户端协同运作，最终形成一套完整的认证闭环。

Sources: [auth-store.ts](frontend/src/stores/auth-store.ts#L1-L65), [auth.ts (types)](frontend/src/types/auth.ts#L1-L37)

## 状态模型：AuthState 接口设计

`AuthState` 接口定义了 store 的完整形态——三个**数据字段**承载认证状态，五个**方法字段**提供操作能力。数据字段包括 `token`（JWT 字符串或 null）、`user`（完整的用户信息对象）和 `isAuthenticated`（布尔派生值，由 token 是否存在决定）。方法字段则分为两类：**写入操作**（`login`、`setUser`、`logout`）负责状态变更并同步到 localStorage；**只读判断**（`hasRole`、`hasPermission`）封装了权限逻辑，供组件直接调用而无需关心数据结构细节。

```typescript
INTerface AuthState {
  token: string | null
  user: UserInfo | null
  isAuthenticated: BOOLEAN
  login: (token: string, user: UserInfo) => void
  setUser: (user: UserInfo) => void
  logout: () => void
  hasRole: (role: string) => BOOLEAN
  hasPermission: (code: string) => BOOLEAN
}
```

`UserInfo` 类型是权限判断的数据基础，它携带了 `roles`（角色数组，每项含 `code` 与 `name`）、`permissions`（权限编码字符串数组）以及一个关键标志位 `is_super_admin`（超级管理员标识）。此外还包含 `group_ids`、`groups`、`available_points` 等业务字段，使得 store 同时承担了"当前用户上下文"的职责——任何组件需要知道"当前用户是谁、有多少积分、属于哪个小组"时，都可以直接从 store 获取。

Sources: [auth-store.ts](frontend/src/stores/auth-store.ts#L4-L13), [auth.ts](frontend/src/types/auth.ts#L19-L30)

## 登录持久化：localStorage 双键策略

持久化机制采用**模块初始化时加载 + 写入时同步**的双阶段模式。store 定义了两个 localStorage 键名常量：`INTegral_mall_token` 和 `INTegral_mall_user`，分别存储 JWT 令牌和用户信息的 JSON 序列化字符串。

```
┌──────────────────────────────────────────────────────┐
│                   应用启动阶段                         │
│                                                      │
│  ┌─────────────┐    ┌─────────────────────────────┐  │
│  │ loadPersisted│───>│ localStorage.getItem(...)   │  │
│  │    ()        │    │  → INTegral_mall_token      │  │
│  └─────────────┘    │  → INTegral_mall_user (JSON) │  │
│         │           └─────────────────────────────┘  │
│         ▼                                           │
│  ┌─────────────────────────────────────────────┐    │
│  │ const persisted = loadPersisted()            │    │
│  │ create<AuthState>(() => ({                   │    │
│  │   token: persisted.token,        ◄── 恢复    │    │
│  │   user: persisted.user,          ◄── 恢复    │    │
│  │   isAuthenticated: !!persisted.token  ◄── 推导│    │
│  │ }))                                          │    │
│  └─────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────┘
```

`loadPersisted()` 函数在模块顶层执行（而非在 React 组件生命周期中），这意味着 store 的初始值在 JavaScript 模块首次被 import 时就已经确定。`try-catch` 保护了 JSON 解析可能出现的异常（例如用户手动篡改 localStorage 内容），确保任何解析失败都安全回退到 `{ token: null, user: null }`。这种**模块级初始化**策略的优势在于：store 创建即包含正确的初始状态，无需额外的 hydration 步骤，避免了 React 组件首次渲染时"闪一下未登录态"的问题。

Sources: [auth-store.ts](frontend/src/stores/auth-store.ts#L15-L36)

### 写入操作与 localStorage 同步

三个写入操作各自维护了 **Zustand 状态** 与 **localStorage** 的一致性：

| 操作 | Zustand 状态变更 | localStorage 变更 | 典型触发场景 |
|------|-----------------|-------------------|------------|
| `login(token, user)` | 设置 token、user、isAuthenticated=true | setItem 两个键 | 登录成功、注册成功 |
| `setUser(user)` | 仅更新 user | 仅 setItem 用户键 | 个人设置保存、profile 刷新 |
| `logout()` | 三字段全部清空/null/false | removeItem 两个键 | 手动登出、401 响应 |

值得注意的是 `setUser` 的设计意图——它不触碰 token 和认证状态，专门用于**用户信息的局部刷新**。在个人设置页面（Settings）中，用户修改姓名或上传头像后，前端调用 `userApi.updateProfile()` 获取最新 profile，通过 `mergeProfileIntoAuthUser()` 合并为完整的 `UserInfo`，然后调用 `setUser()` 更新 store 和 localStorage。这样既保持了 token 不变，又确保侧边栏底部显示的用户名和头像即时更新。

Sources: [auth-store.ts](frontend/src/stores/auth-store.ts#L38-L53), [settings/index.tsx](frontend/src/pages/settings/index.tsx#L22-L23), [profile-sync.ts](frontend/src/pages/settings/profile-sync.ts#L1-L15)

## 权限判断：双层体系与超级管理员穿透

积分商城实现了**角色 + 权限码**的双层授权模型，对应 store 中的两个判断方法：

### hasRole：角色判断

```typescript
hasRole: (role) => {
  const { user } = get()
  return user?.roles?.some((r) => r.code === role) ?? false
}
```

`hasRole` 检查用户的 `roles` 数组中是否包含指定 `code` 的角色。这是一个**精确匹配**——角色码完全对应才返回 true。当 user 为 null（未登录）时，可选链和空值合并确保安全返回 false。

### hasPermission：权限码判断 + 超级管理员穿透

```typescript
hasPermission: (code) => {
  const { user } = get()
  if (user?.is_super_admin) return true
  return user?.permissions?.includes(code) ?? false
}
```

`hasPermission` 的设计包含一个关键的**短路逻辑**：如果 `is_super_admin` 为 true，立即返回 true，不再检查具体权限码。这意味着超级管理员在权限判断层面拥有完全穿透能力，无需为其分配所有权限码。对于普通用户，则使用 `Array.includes()` 精确匹配权限编码字符串。

| 角色/状态 | hasRole('admin') | hasPermission('page:review') | hasPermission('any:unknown') |
|----------|------------------|------------------------------|------------------------------|
| 未登录用户 | false | false | false |
| participant 角色 | false | false（除非 permissions 含该码） | false |
| reviewer 角色 + page:review 权限 | false | true | false |
| super_admin | 取决于 roles 数据 | **true（穿透）** | **true（穿透）** |

Sources: [auth-store.ts](frontend/src/stores/auth-store.ts#L55-L64)

## useAuth Hook：面向组件的状态聚合层

直接使用 `useAuthStore` 进行选择性订阅固然可行，但项目提供了一个 `useAuth()` 自定义 Hook 作为更便捷的访问层。它聚合了 store 中最常用的状态和判断方法，并额外派生了两个业务属性：

```typescript
export function useAuth() {
  const user = useAuthStore((s) => s.user)
  const isAuthenticated = useAuthStore((s) => s.isAuthenticated)
  const hasRole = useAuthStore((s) => s.hasRole)
  const hasPermission = useAuthStore((s) => s.hasPermission)

  return {
    user,
    isAuthenticated,
    hasRole,
    hasPermission,
    isSuperAdmin: user?.is_super_admin ?? false,
    availablePoints: user?.available_points ?? 0,
  }
}
```

`isSuperAdmin` 和 `availablePoints` 是从 `user` 对象派生的便捷属性——前者避免组件重复写 `user?.is_super_admin ?? false`，后者让商品商城页、仪表盘等需要展示积分余额的组件无需直接访问 user 对象。这种**聚合 + 派生**的模式使组件代码更加简洁、意图更加明确。

在页面组件中的使用模式如下表所示：

| 页面 | 引用方式 | 使用的属性 | 用途 |
|------|---------|-----------|------|
| Dashboard | `useAuth()` | user, hasRole, hasPermission | 条件渲染审核统计、管理入口 |
| ProductMall | `useAuth()` | availablePoints | 展示当前可用积分 |
| AdminOrders | `useAuth()` | hasRole, isSuperAdmin | 区分 admin/merchant 视图 |
| ReviewPending | `useAuth()` | hasPermission, isSuperAdmin | 控制审核操作权限 |
| Settings | `useAuthStore()` | user, setUser | 读取并更新用户信息 |
| Login | `useAuthStore()` | login | 登录成功写入状态 |

Sources: [use-auth.ts](frontend/src/hooks/use-auth.ts#L1-L17), [dashboard/index.tsx](frontend/src/pages/dashboard/index.tsx#L14-L23), [product/index.tsx](frontend/src/pages/product/index.tsx#L9-L24), [admin/orders.tsx](frontend/src/pages/admin/orders.tsx#L9-L20)

## 路由守卫：四层鉴权与 store 的协作

路由守卫直接消费 `useAuthStore` 的选择器，构成了认证闭环中**"拦截"**这一环。守卫系统分为四层，每层对应不同的鉴权粒度：

```
┌─────────────────────────────────────────────────────────────┐
│                      路由请求进入                            │
│                           │                                 │
│                  ┌────────▼────────┐                        │
│                  │   GuestOnly     │ ──── 已登录 ──> /dashboard
│                  │                 │ ──── 未登录 ──> 放行     │
│                  └────────┬────────┘                        │
│                           │                                 │
│                ┌──────────▼──────────┐                      │
│                │    RequireAuth      │ ──── 未登录 ──> /login
│                │                     │ ──── 已登录 ──> 放行   │
│                └──────────┬──────────┘                      │
│                           │                                 │
│           ┌───────────────┼───────────────┐                 │
│           │               │               │                 │
│    ┌──────▼──────┐ ┌──────▼──────┐ ┌──────▼──────┐         │
│    │ RoleGuard   │ │ Permission  │ │  (无守卫)    │         │
│    │ roles:[]    │ │ Guard       │ │  公共页面    │         │
│    └──────┬──────┘ │ permission: │ │             │         │
│           │        └──────┬──────┘ └─────────────┘         │
│     ┌─────┴─────┐    ┌────┴────┐                          │
│     │ 角色匹配   │    │ 权限匹配 │                          │
│     │ → Outlet  │    │ → Outlet│                          │
│     │ 或 403    │    │ 或 403  │                          │
│     └───────────┘    └─────────┘                          │
└─────────────────────────────────────────────────────────────┘
```

**GuestOnly** 作用于 `/login` 和 `/register`，确保已登录用户无法访问登录页——如果 `isAuthenticated` 为 true，直接重定向到 `/dashboard`。**RequireAuth** 是所有需要认证的页面的外层守卫，未登录时重定向到 `/login`。**RoleGuard** 和 **PermissionGuard** 是内层精细守卫，分别调用 `user.roles` 和 `store.hasPermission()` 进行判断，不匹配时渲染 Ant Design 的 403 Result 组件。

在路由配置中，守卫的嵌套形成了清晰的权限层级。例如 `/review/pending` 路由同时需要 `RequireAuth`（外层）和 `PermissionGuard permission="page:review"`（内层）双重校验；`/admin/orders` 则需要 `RequireAuth` + `RoleGuard roles={['merchant', 'admin', 'super_admin']}`，允许三种角色访问。

Sources: [guards.tsx](frontend/src/router/guards.tsx#L1-L84), [router/index.tsx](frontend/src/router/index.tsx#L1-L100)

## API 客户端集成：JWT 注入与 401 自动登出

`useAuthStore` 不仅服务于 UI 层，还深度集成到了 Axios HTTP 客户端中，承担了**请求鉴权**和**会话失效处理**两个关键职责。

**请求拦截器**在每次请求发送前从 store 中读取 token，注入 `Authorization: Bearer <token>` 头。这里使用的是 `useAuthStore.getState().token`（而非 React Hook 形式），因为拦截器运行在 React 组件树之外，无法使用 Hook。Zustand 的 `getState()` API 完美适配了这种非 React 上下文的访问需求。

**响应拦截器**处理了两种场景：正常响应解包 `{code, message, data}` 结构；错误响应中，HTTP 401 状态码触发自动登出流程——调用 `useAuthStore.getState().logout()` 清除 store 和 localStorage，然后通过 `window.location.href = '/login'` 硬跳转到登录页。这种**被动登出**机制确保了当后端判定 token 过期或无效时，前端能立即清理状态并引导用户重新登录。

```
浏览器请求 → Axios 拦截器注入 JWT → 后端验证 →
  ├── 200 + code=0 → 正常解包返回 data
  ├── 200 + code≠0 → message.error 提示
  └── 401 → store.logout() + 跳转 /login
```

Sources: [client.ts](frontend/src/api/client.ts#L1-L65)

## 侧边栏权限过滤：动态菜单渲染

侧边栏组件（Sidebar）是权限判断在 UI 层最直观的体现。菜单数据以 `MenuGroup[]` 结构静态定义，每个菜单项可选配 `permission` 或 `roles` 条件。渲染时，组件从 store 获取 `hasPermission` 和 `hasRole` 方法，过滤掉用户无权看到的菜单项：

```typescript
const filtered = group.items.filter((item) => {
  const matchesPermission = item.permission ? hasPermission(item.permission) : false
  const matchesRole = item.roles ? item.roles.some((role) => hasRole(role)) : false
  return (!item.permission && !item.roles) || matchesPermission || matchesRole
})
```

过滤逻辑包含三种情况：**无条件菜单**（如仪表盘、积分申请）所有登录用户可见；**权限码菜单**（如"用户管理"需要 `page:admin:users`）通过 `hasPermission` 判断；**角色菜单**（如"订单管理"需要 merchant/admin/super_admin 角色之一）通过 `hasRole` 判断。这确保了普通参与者只看到基础菜单，而审核员、管理员、商家各自看到对应的专属功能区。

| 菜单分组 | 菜单项 | 守卫条件 | 参与者可见 | 审核员可见 | 管理员可见 |
|---------|--------|---------|----------|----------|----------|
| 默认 | 仪表盘 | 无条件 | ✓ | ✓ | ✓ |
| 默认 | 积分申请 | 无条件 | ✓ | ✓ | ✓ |
| 审核工作台 | 待审核 | `page:review` | ✗ | ✓ | ✓ |
| 管理后台 | 用户管理 | `page:admin:users` | ✗ | ✗ | ✓ |
| 管理后台 | 商品管理 | `page:admin:products` | ✗ | ✗ | ✓ |
| 管理后台 | 订单管理 | roles: merchant/admin/super_admin | ✗ | ✗ | ✓ |

Sources: [sidebar.tsx](frontend/src/layouts/main-layout/sidebar.tsx#L35-L106)

## 完整认证生命周期

将上述所有模块串联，可以描绘出认证状态从创建到销毁的完整生命周期：

1. **用户在登录页输入凭证** → `authApi.login()` 发起 POST 请求到 `/auth/login`
2. **后端返回 `{token, user}` 响应** → `LoginPage` 调用 `useAuthStore.getState().login(token, user)`
3. **login 方法同时更新 Zustand 状态和 localStorage** → 页面跳转至 `/dashboard`
4. **后续所有 API 请求自动携带 JWT** → Axios 请求拦截器从 store 读取 token
5. **路由守卫根据 store 状态控制访问** → RequireAuth、RoleGuard、PermissionGuard 各司其职
6. **侧边栏根据权限动态渲染菜单** → 用户只能看到有权限的功能入口
7. **个人设置修改后，setUser 同步更新 store 和 localStorage** → 侧边栏用户名/头像即时刷新
8. **当后端返回 401 时** → 响应拦截器调用 `logout()` 清除一切状态，跳转登录页
9. **用户主动登出** → `logout()` 清除 token、user、isAuthenticated，移除 localStorage，重定向到 `/login`

Sources: [login/index.tsx](frontend/src/pages/login/index.tsx#L14-L26), [auth-store.ts](frontend/src/stores/auth-store.ts#L38-L53), [client.ts](frontend/src/api/client.ts#L14-L42)

## 测试保障：单元测试与 E2E 状态快照

auth-store 的单元测试覆盖了三大功能维度：**登录/登出流程**（验证 Zustand 状态变更和 localStorage 持久化）、**hasRole 角色判断**（精确匹配与未登录边界）、**hasPermission 权限判断**（普通用户权限检查与超级管理员穿透）。测试使用 `vi.clearAllMocks()` 和 `localStorage.clear()` 在每个用例前重置状态，并通过 `useAuthStore.setState()` 直接操控 store 避免用例间污染。

在 E2E 层面，项目为 9 种角色（admin、chief、merchant、observer、participant、reviewer 等）准备了独立的 localStorage 状态快照文件（`frontend/e2e/auth-states/`），每个文件预填充了对应角色的 `INTegral_mall_token` 和 `INTegral_mall_user`。Playwright 测试通过加载这些快照来模拟不同角色的登录状态，无需每次都走登录流程——这大幅提升了 E2E 测试的执行效率。

Sources: [auth-store.test.ts](frontend/src/stores/__tests__/auth-store.test.ts#L1-L92), [admin.json](frontend/e2e/auth-states/admin.json#L1-L18), [participant.json](frontend/e2e/auth-states/participant.json#L1-L18)

## 设计决策与权衡

本系统的架构选择体现了几个值得注意的权衡点：

**手动 localStorage 持久化 vs zustand/middleware/persist**：项目选择了手动读写 localStorage 而非使用 Zustand 官方的 persist 中间件。这种选择的优势是代码极简、无额外依赖、行为完全透明——`loadPersisted` 只在模块加载时执行一次，`login/setUser/logout` 的 localStorage 操作一目了然。代价是需要手动维护每个写入方法的双写逻辑，且缺少了 persist 中间件提供的 partialize、version 迁移等高级能力。对于当前规模的认证状态，这个权衡是合理的。

**权限扁平列表 vs 权限树**：`permissions` 字段是一个简单的字符串数组（如 `['page:review', 'page:admin:users']`），而非嵌套的权限树结构。这使得 `hasPermission` 的实现极其高效（`Array.includes`），但要求权限编码本身具有足够的前缀语义（如 `page:` 前缀区分页面级权限，`admin:` 前缀标识管理操作）。编码规范的约束通过后端 RBAC 系统保证，前端无需关心编码的来源。

**模块级初始化 vs useEffect hydration**：`loadPersisted()` 在模块顶层执行而非在 React 组件的 `useEffect` 中，避免了首次渲染时的"未登录闪烁"。这在 SSR 场景中可能带来问题（localStorage 不存在），但积分商城是纯 CSR 应用，因此这个选择是安全的。

Sources: [auth-store.ts](frontend/src/stores/auth-store.ts#L18-L36), [auth.ts (types)](frontend/src/types/auth.ts#L19-L30)

## 延伸阅读

- **路由守卫的详细实现**：参见 [React 19 应用架构：路由、状态管理与权限守卫](15-react-19-ying-yong-jia-gou-lu-you-zhuang-tai-guan-li-yu-quan-xian-shou-wei)，了解四层守卫在路由配置中的嵌套策略
- **JWT 令牌的后端生成与验证**：参见 [JWT 认证中间件与上下文传递机制](12-jwt-ren-zheng-zhong-jian-jian-yu-shang-xia-wen-chuan-di-ji-zhi)，理解 token 的结构、过期策略和中间件鉴权
- **RBAC 权限编码的设计原则**：参见 [RBAC 权限系统：角色、权限编码与中间件鉴权](9-rbac-quan-xian-xi-tong-jiao-se-quan-xian-bian-ma-yu-zhong-jian-jian-quan)，理解 `page:review` 等权限码的定义规范
- **契约驱动开发中的类型安全**：参见 [前后端契约驱动开发：goctl 类型生成与漂移检查](16-qian-hou-duan-qi-yue-qu-dong-kai-fa-goctl-lei-xing-sheng-cheng-yu-piao-yi-jian-cha)，了解 `LoginReq`、`LoginResp` 等类型如何从后端 API 定义自动生成