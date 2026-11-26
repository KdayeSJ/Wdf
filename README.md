# 闪击の业务 v2.0

游戏周边业务下单系统 — 暗黑奢华科技风，白色液态玻璃 UI，丝滑动效。

## 技术栈

- **前端**: React 19 + TypeScript + Vite + Tailwind CSS 4 + shadcn/ui + Framer Motion
- **后端**: NestJS 10 + TypeScript + Drizzle ORM
- **数据库**: PostgreSQL
- **认证**: JWT（`X-App-Auth-Token`）+ 自建账号体系
- **通知**: 方糖（Server酱）推送 + 站内通知
- **部署**: 妙搭全栈应用平台

## 目录结构

```
├── client/                  # React 前端
│   ├── src/
│   │   ├── pages/           # 页面组件（首页/登录/注册/个人中心/后台）
│   │   ├── components/      # 通用组件 + 业务组件 + UI 库
│   │   ├── api/             # 前端 API 调用层
│   │   ├── hooks/           # 自定义 hooks
│   │   ├── utils/           # 工具函数
│   │   ├── app.tsx          # 路由定义
│   │   └── index.tsx        # 入口
│   ├── index.html
│   └── public/              # 静态资源
├── server/                  # NestJS 后端
│   ├── main.ts              # 入口
│   ├── app.module.ts        # 根模块
│   ├── database/
│   │   └── schema.ts        # Drizzle schema（自动生成）
│   └── modules/             # 业务模块
│       ├── auth/            # 认证（注册/登录/个人资料/方糖绑定）
│       ├── order/           # 订单创建 + 优惠券校验
│       ├── user-order/      # 用户侧订单查询
│       ├── admin-order/     # 管理员订单管理 + 强改价
│       ├── admin-user/      # 管理员用户管理 + 导出
│       ├── coupon/          # 优惠券（发放/作废/我的券）
│       ├── broadcast/       # 全站广播
│       ├── lottery/         # 抽奖活动
│       ├── notification/    # 站内通知 + 方糖推送
│       └── view/            # 模板渲染（平台内置，勿改）
├── shared/
│   └── api.interface.ts     # 前后端共享类型定义
├── package.json
├── tsconfig.app.json        # 前端 tsconfig
├── tsconfig.node.json       # 后端 tsconfig
├── vite.config.ts           # Vite 配置
├── nest-cli.json            # Nest CLI 配置
├── tailwind.config.ts       # Tailwind 配置
├── postcss.config.js
└── README.md
```

## 本地运行步骤

### 前置要求

- Node.js >= 22.0.0
- npm >= 10.0.0
- PostgreSQL >= 14

### 1. 安装依赖

```bash
npm install
```

### 2. 配置环境变量

复制 `.env.example` 为 `.env`，填入以下变量：

```env
# 数据库连接（必填）
DATABASE_URL=postgresql://username:password@localhost:5432/shanji_db

# JWT 密钥（必填，建议 32 位以上随机字符串）
JWT_SECRET=your_jwt_secret_key_here

# 日志配置
LOG_DIR=./logs
LOG_REQUEST_BODY=true
LOG_RESPONSE_BODY=true
```

### 3. 初始化数据库表结构

在 PostgreSQL 中创建数据库后，执行建表 SQL。参考 `server/database/schema.ts` 中的表定义，或使用 Drizzle Kit 生成迁移：

```bash
# 安装 drizzle-kit（如未安装）
npm install -D drizzle-kit

# 生成并执行迁移
npx drizzle-kit generate
npx drizzle-kit push
```

> **注意**: 实际生产环境使用妙搭平台的数据库能力，`schema.ts` 由平台 DDL 执行后自动生成。本地自建需手动建表。

### 4. 启动开发服务器

```bash
# 同时启动前后端（推荐）
npm run dev

# 或分别启动
npm run dev:server   # 后端默认 3000 端口
npm run dev:client   # 前端默认 5173 端口
```

访问 `http://localhost:5173` 查看前端页面。

### 5. 构建生产版本

```bash
npm run build
```

构建产物位于：
- 后端：`dist/server/`
- 前端：`dist/client/`

## 默认管理员账号

系统无内置默认管理员，需手动注册后通过 SQL 提升权限：

```sql
-- 1. 先在前端注册一个账号（如 admin）
-- 2. 执行 SQL 提升为超级管理员
UPDATE users_accounts 
SET role = 'super_admin' 
WHERE username = '你的用户名';
```

### 角色体系

| 角色 | 权限 |
|------|------|
| `user` | 普通用户：下单、查订单、抽奖、用券 |
| `admin_normal` | 普通管理员：订单管理、强改价（触发通知） |
| `admin_mid` | 中级管理员：以上 + 优惠券发放、广播管理、抽奖管理 |
| `super_admin` | 超级管理员：以上 + 用户管理、管理员管理、数据导出 |

## 主要 API 接口清单

### 认证模块 `POST /api/auth/*`

| 方法 | 路径 | 说明 | 鉴权 |
|------|------|------|------|
| POST | `/api/auth/register` | 用户注册 | - |
| POST | `/api/auth/login` | 用户/管理员登录 | - |
| GET | `/api/auth/me` | 获取当前用户信息 | ✓ |
| PATCH | `/api/auth/profile` | 更新个人资料 | ✓ |
| PATCH | `/api/auth/password` | 修改密码 | ✓ |
| POST | `/api/auth/logout` | 退出登录 | ✓ |
| PATCH | `/api/auth/sendkey` | 绑定/解绑方糖 SendKey | ✓ |
| POST | `/api/auth/sendkey/test` | 发送方糖测试消息 | ✓ |

### 订单模块

| 方法 | 路径 | 说明 | 鉴权 |
|------|------|------|------|
| POST | `/api/order` | 创建订单（支持优惠券） | 可选 |
| POST | `/api/order/check-coupon` | 校验优惠券可用性 | ✓ |
| GET | `/api/user/orders` | 我的订单列表 | ✓ |
| GET | `/api/user/orders/:id` | 我的订单详情 | ✓ |

### 管理员订单

| 方法 | 路径 | 说明 | 角色 |
|------|------|------|------|
| GET | `/api/admin/orders` | 订单列表（分页+筛选） | admin+ |
| GET | `/api/admin/orders/:id` | 订单详情 | admin+ |
| PATCH | `/api/admin/orders/:id/status` | 更新订单状态 | admin+ |
| PATCH | `/api/admin/orders/:id/internal-note` | 更新内部备注 | admin+ |
| PATCH | `/api/admin/orders/:id/price` | 强改订单价格 | admin+ |
| DELETE | `/api/admin/orders/:id` | 删除订单 | admin+ |
| PATCH | `/api/admin/orders/:id/reset` | 重置订单状态 | admin+ |
| GET | `/api/admin/orders/:id/audit-logs` | 订单审计日志 | admin+ |

### 优惠券模块

| 方法 | 路径 | 说明 | 角色 |
|------|------|------|------|
| GET | `/api/coupons/mine` | 我的优惠券 | user |
| GET | `/api/admin/coupons` | 优惠券列表 | admin_mid+ |
| POST | `/api/admin/coupons/issue` | 发放优惠券 | admin_mid+ |
| PATCH | `/api/admin/coupons/:id/void` | 作废优惠券 | admin_mid+ |

### 广播模块

| 方法 | 路径 | 说明 | 角色 |
|------|------|------|------|
| GET | `/api/broadcasts/latest` | 最新激活广播 | - |
| GET | `/api/broadcasts` | 广播列表（需登录） | ✓ |
| GET | `/api/admin/broadcasts` | 后台广播列表 | admin_mid+ |
| POST | `/api/admin/broadcasts` | 发布广播 | admin_mid+ |
| PATCH | `/api/admin/broadcasts/:id` | 编辑广播 | admin_mid+ |
| DELETE | `/api/admin/broadcasts/:id` | 删除广播 | admin_mid+ |

### 抽奖模块

| 方法 | 路径 | 说明 | 角色 |
|------|------|------|------|
| GET | `/api/lots` | 抽奖活动列表 | - |
| POST | `/api/lots/:id/draw` | 参与抽奖 | user |
| GET | `/api/draws/mine` | 我的抽奖记录 | user |
| GET | `/api/admin/lots` | 后台活动列表 | admin_mid+ |
| POST | `/api/admin/lots` | 创建抽奖活动 | admin_mid+ |
| POST | `/api/admin/lots/:id/end` | 结束活动 | admin_mid+ |
| GET | `/api/admin/lots/:id/draws` | 活动中奖记录 | admin_mid+ |

### 通知模块

| 方法 | 路径 | 说明 | 鉴权 |
|------|------|------|------|
| GET | `/api/notifications` | 通知列表 | ✓ |
| PATCH | `/api/notifications/:id/read` | 标为已读 | ✓ |
| PATCH | `/api/notifications/read-all` | 全部已读 | ✓ |

### 管理员用户管理

| 方法 | 路径 | 说明 | 角色 |
|------|------|------|------|
| GET | `/api/admin/users` | 用户列表 | super_admin |
| GET | `/api/admin/users/search` | 搜索用户 | super_admin |
| PATCH | `/api/admin/users/:id/status` | 启用/禁用用户 | super_admin |
| DELETE | `/api/admin/users/:id` | 删除用户 | super_admin |
| GET | `/api/admin/admins` | 管理员列表 | super_admin |
| POST | `/api/admin/admins` | 新增管理员 | super_admin |
| PATCH | `/api/admin/admins/:id/role` | 修改管理员角色 | super_admin |
| DELETE | `/api/admin/admins/:id` | 删除管理员 | super_admin |
| GET | `/api/admin/export/users` | 导出用户数据 | super_admin |
| GET | `/api/admin/export/orders` | 导出订单数据 | super_admin |

## 数据库表

| 表名 | 说明 |
|------|------|
| `users_accounts` | 用户账号（含角色、方糖 sendkey） |
| `orders` | 订单（含原价/实付/改价记录/优惠券） |
| `coupons` | 优惠券 |
| `broadcasts` | 全站广播 |
| `lots` | 抽奖活动 |
| `draws` | 抽奖记录 |
| `notifications` | 站内通知 |
| `audit_logs` | 操作审计日志 |

详细字段定义见 `server/database/schema.ts`。

## 特色功能

- **白色液态玻璃 UI**：毛玻璃 + 渐变光效，暗黑奢华科技风
- **丝滑动效**：Framer Motion 驱动的页面过渡与交互动画
- **三级管理员体系**：普通 / 中级 / 超级管理员，权限分级
- **方糖通知**：新订单、改价事件自动推送到微信
- **强改价**：管理员可修改订单价格，记录审计日志
- **优惠券系统**：折扣券 / 免单券 / 指定商品券
- **抽奖系统**：5 种奖品，每人每活动一次，中奖自动发券
- **全站广播**：首页横幅展示，支持 info/warning/promo 三级
- **移动端适配**：全页面响应式布局
- **数据导出**：用户/订单数据 TXT 导出

## 方糖（Server酱）配置

1. 注册 [方糖](https://sct.ftqq.com/) 账号获取 SendKey
2. 登录后进入个人中心 → 方糖设置 → 绑定 SendKey
3. 点击"发送测试消息"验证配置

管理员绑定后，新订单创建、订单改价等事件会自动推送微信通知。

## 版本信息

- 当前版本：**v2.0**
- 品牌名称：闪击の业务
