# 云熠星河 CMS 设计文档

## 概述

将当前纯静态粉丝应援 H5 页面改造为基于 Next.js 的全栈内容管理系统，支持在 VSCode 中开发、Vercel 部署，并内置管理后台实现动态内容管理。

## 技术栈

| 技术 | 用途 |
|------|------|
| Next.js 15 (App Router) | 全栈框架 |
| Prisma ORM | 数据库访问层 |
| Vercel Postgres (Neon) | PostgreSQL 数据库 |
| Auth.js (NextAuth v5) | 管理员认证 |
| shadcn/ui + Tailwind CSS | UI 组件库 |
| Vercel Blob | 图片存储 |
| Vercel KV (Redis) | 登录速率限制 |
| TypeScript | 类型安全 |

## 整体架构

```
┌─────────────────────────────────────────────────────┐
│                   用户浏览器                          │
│  ┌─────────────────┐   ┌─────────────────────────┐  │
│  │   公开页面        │   │    管理后台 /admin       │  │
│  │   (移动端优先)    │   │   (桌面端布局)           │  │
│  └────────┬────────┘   └───────────┬─────────────┘  │
└───────────┼─────────────────────────┼────────────────┘
            │                         │
            ▼                         ▼
┌─────────────────────────────────────────────────────┐
│              Next.js App Router (Server + Client)     │
│  ┌──────────┐  ┌──────────┐  ┌──────────────────┐  │
│  │ ISR 静态  │  │ Server   │  │ API Routes       │  │
│  │ 渲染      │  │ Actions  │  │ /api/admin/*     │  │
│  └──────┬───┘  └────┬─────┘  └────────┬─────────┘  │
└─────────┼───────────┼──────────────────┼─────────────┘
          ▼           ▼                  ▼
┌─────────────────────────────────────────────────────┐
│                   Prisma ORM                        │
│                     ▼                               │
│              Vercel Postgres                        │
└─────────────────────────────────────────────────────┘
```

## 目录结构

```
yunyi/
├── app/
│   ├── (public)/              # 公开页面路由组
│   │   ├── layout.tsx         # 公共布局（粒子背景、全局样式）
│   │   └── page.tsx           # 首页（所有内容区块）
│   ├── admin/
│   │   ├── layout.tsx         # 管理后台布局（侧边栏）
│   │   ├── login/page.tsx     # 登录页面
│   │   ├── page.tsx           # 仪表盘
│   │   ├── artists/page.tsx   # 艺人资料管理
│   │   ├── works/page.tsx     # 作品管理
│   │   ├── photos/page.tsx    # 图片管理
│   │   ├── accounts/page.tsx  # 社交账号管理
│   │   ├── groups/page.tsx    # 后援会组管理
│   │   └── settings/page.tsx  # 站点设置（封面、标题等）
│   └── api/
│       ├── artists/route.ts   # 艺人 CRUD
│       ├── works/route.ts     # 作品 CRUD
│       ├── photos/route.ts    # 图片 CRUD + 上传
│       ├── accounts/route.ts  # 社交账号 CRUD
│       ├── groups/route.ts    # 后援会组 CRUD
│       └── settings/route.ts  # 站点配置
├── components/
│   ├── ui/                    # shadcn/ui 组件
│   ├── public/                # 公开页面组件
│   │   ├── CoverHero.tsx      # 封面海报
│   │   ├── ArtistInfo.tsx     # 艺人资料卡
│   │   ├── PhotoGallery.tsx   # 图片画廊（带 lightbox）
│   │   ├── SocialLinks.tsx    # 社交账号链接
│   │   ├── WorksSection.tsx   # 作品展示
│   │   ├── FanGroups.tsx      # 后援会组
│   │   └── ParticleBg.tsx     # 粒子动画背景
│   └── admin/                 # 管理后台组件
│       ├── DataTable.tsx      # 通用数据表格
│       └── ImageUpload.tsx    # 图片上传（Vercel Blob）
├── lib/
│   ├── db.ts                  # Prisma 客户端
│   ├── auth.ts                # Auth.js 配置
│   └── utils.ts               # 工具函数
├── prisma/
│   ├── schema.prisma          # 数据库模型定义
│   └── seed.ts                # 种子数据（从现有 index.html 迁移）
├── public/
│   └── photos/                # 初始本地图片（种子数据导入后迁移至 Vercel Blob）
├── middleware.ts               # 路由中间件（认证保护）
└── package.json
```

## 数据模型

### Artist（艺人）

| 字段 | 类型 | 说明 |
|------|------|------|
| id | Int (PK) | 自增 ID |
| name | String | 姓名 |
| nickname | String? | 昵称 |
| origin | String? | 籍贯 |
| birthday | String? | 生日 |
| zodiac | String? | 星座 |
| height | String? | 身高 |
| university | String? | 毕业院校 |
| mbti | String? | MBTI |
| supportColor | String? | CP 应援色 |
| cpName | String? | CP 名 |
| fanName | String? | CP 粉丝名 |
| agency | String? | 经纪公司 |
| updatedAt | DateTime | 最后更新时间（并发控制） |

### SocialAccount（社交账号）

| 字段 | 类型 | 说明 |
|------|------|------|
| id | Int (PK) | 自增 ID |
| platform | String | 平台名（微博/抖音/小红书） |
| handle | String | 账号名 |
| url | String? | 链接 |
| artistId | Int (FK) | 关联艺人 |

### Work（作品）

| 字段 | 类型 | 说明 |
|------|------|------|
| id | Int (PK) | 自增 ID |
| type | String | 类型：tv/magazine/music |
| title | String | 作品名 |
| role | String? | 饰演角色 |
| status | String | 状态：airing/upcoming/published |
| url | String? | 链接 |
| artistId | Int (FK) | 关联艺人 |

### Photo（图片）

| 字段 | 类型 | 说明 |
|------|------|------|
| id | Int (PK) | 自增 ID |
| url | String | 图片 URL |
| alt | String? | 描述 |
| category | String | 分类：gallery/cover/magazine |

### FanGroup（后援会组）

| 字段 | 类型 | 说明 |
|------|------|------|
| id | Int (PK) | 自增 ID |
| name | String | 组名 |
| description | String? | 描述 |
| groupType | String | 类型：data/anti_black/admin/publicity |

### FanGroupLink（后援会链接）

| 字段 | 类型 | 说明 |
|------|------|------|
| id | Int (PK) | 自增 ID |
| label | String | 链接标签 |
| url | String | URL |
| groupId | Int (FK) | 关联分组 |

### SiteConfig（站点配置）

| 字段 | 类型 | 说明 |
|------|------|------|
| id | Int (PK) | 自增 ID |
| key | String (unique) | 配置键名 |
| value | String | 配置值 |

## 公开页面设计

### 移动优先策略
- 容器最大宽度 750px，桌面端居中显示
- 保持原有星空粒子动画效果
- 使用 Next.js Image 组件优化图片加载

### 新增功能
- 底部粘性导航栏：快速跳转各内容区块
- 图片 Lightbox：点击放大查看
- 链接一键复制：方便粉丝分享

### 数据获取
- Server Components + ISR（`revalidate: 60`）
- 公开页面无需认证
- 静态页面自动刷新，粉丝访问速度快

## 管理后台设计

### 认证方式
- Auth.js v5 + Credentials Provider
- 单管理员账号，密码 bcrypt 加密存储
- 初始管理员账号通过 `prisma/seed.ts` 种子脚本创建
- HTTP-only session cookies
- 登录速率限制：使用 Vercel KV (Redis) 存储登录尝试计数，5 次/分钟限制

### 路由保护
- `middleware.ts` 保护 `/admin/*` 路由
- 未认证访问重定向到登录页

### 后台功能与验收标准

| 模块 | 操作 | 验收标准 |
|------|------|----------|
| 仪表盘 | 内容概览（各模块数据条数统计） | 页面加载后显示各模块准确的数据条数 |
| 艺人资料 | 编辑/更新艺人信息 | 修改后保存成功，公开页面 60 秒内 ISR 刷新显示新内容 |
| 作品管理 | 新增/编辑/删除作品 | CRUD 操作均正常执行，删除需二次确认，列表实时更新 |
| 图片管理 | 上传/删除/拖拽排序 | 图片上传至 Vercel Blob 成功，支持预览、删除、拖拽调整顺序 |
| 社交账号 | 管理各平台链接 | 新增/编辑/删除账号，链接格式正确，公开页面实时展示 |
| 后援会组 | 管理分组及链接 | 分组及链接 CRUD 正常，删除分组时级联删除关联链接 |
| 站点设置 | 封面图、标题、副标题 | 修改后公开页面封面、标题、副标题立即更新 |

## 安全设计

- 管理后台路由 `/admin/*` 通过 middleware 保护
- 登录速率限制：Vercel KV (Redis) 存储登录尝试计数
- 所有用户输入服务端 XSS 过滤
- CSRF 保护（Next.js 内置）
- 数据库连接使用 SSL（`sslmode=require`）
- 环境变量通过 Vercel Environment Variables 管理

## 错误处理

| 场景 | 处理方式 |
|------|----------|
| 数据库连接失败 | 返回 503 错误页面，显示"服务暂时不可用" |
| API 请求失败 | 返回标准化 JSON 错误响应 `{ error, message }` |
| 图片上传失败 | 前端显示错误提示，支持重试 |
| 表单验证失败 | 字段级错误提示，阻止提交 |
| Session 过期 | 自动跳转登录页，保留当前页面 URL 作为回调 |
| 并发编辑 | 使用 `updatedAt` 版本号检测冲突，提示用户刷新 |

## 图片存储策略

使用 Vercel Blob 作为图片存储，支持运行时上传：
- Vercel Blob 免费套餐：500MB 存储 + 5GB 带宽/月
- 当前图片总量约 10MB，远低于限制
- 管理后台支持图片上传、删除
- 上传时自动生成 WebP 格式和多尺寸缩略图
- 种子数据阶段：将 `public/photos/` 中的初始图片迁移至 Blob

## 部署方案

- Vercel 自动部署（关联 Git 仓库）
- 公开页面使用 ISR，兼顾速度与实时性
- 环境变量在 Vercel 控制台配置
- 数据库备份：Vercel Postgres 自动备份

## 内容迁移

从现有 `index.html` 提取所有硬编码内容，编写 `prisma/seed.ts` 种子脚本：
- 2 位艺人基础信息
- 5 个社交账号
- 4 部作品
- 11 张图片（迁移至 Vercel Blob 后记录 URL）
- 5 个后援会组及其链接
- 站点配置（标题、副标题）
- 初始管理员账号

## 项目规范对齐

本项目为全新 Next.js 项目，无已有项目规范。采用以下社区标准：

- **组件库**：shadcn/ui（非 raw Radix UI）
- **样式方案**：Tailwind CSS utility-first
- **数据请求**：Prisma ORM 直接访问，不使用 raw fetch
- **API 规范**：Next.js App Router Server Actions + Route Handlers
- **类型安全**：全项目 TypeScript strict mode
- **代码风格**：ESLint + Prettier
