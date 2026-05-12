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
| Vercel Blob | 图片存储（可选） |
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
│   │   ├── page.tsx           # 仪表盘
│   │   ├── artists/page.tsx   # 艺人资料管理
│   │   ├── works/page.tsx     # 作品管理
│   │   ├── photos/page.tsx    # 图片管理
│   │   ├── accounts/page.tsx  # 社交账号管理
│   │   ├── groups/page.tsx    # 后援会组管理
│   │   └── settings/page.tsx  # 站点设置（封面、标题等）
│   └── api/
│       └── admin/[...resource]/route.ts  # 通用 CRUD
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
│       ├── ImageUpload.tsx    # 图片上传
│       └── FormBuilder.tsx    # 表单构建器
├── lib/
│   ├── db.ts                  # Prisma 客户端
│   ├── auth.ts                # Auth.js 配置
│   └── utils.ts               # 工具函数
├── prisma/
│   ├── schema.prisma          # 数据库模型定义
│   └── seed.ts                # 种子数据（从现有 index.html 迁移）
├── public/
│   └── photos/                # 本地图片（可选迁移至 Vercel Blob）
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
| coverPhotoUrl | String? | 封面图 URL |
| sortOrder | Int | 排序 |

### SocialAccount（社交账号）

| 字段 | 类型 | 说明 |
|------|------|------|
| id | Int (PK) | 自增 ID |
| platform | String | 平台名（微博/抖音/小红书） |
| handle | String | 账号名 |
| url | String? | 链接 |
| artistId | Int (FK) | 关联艺人 |
| sortOrder | Int | 排序 |

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
| sortOrder | Int | 排序 |

### Photo（图片）

| 字段 | 类型 | 说明 |
|------|------|------|
| id | Int (PK) | 自增 ID |
| url | String | 图片 URL |
| alt | String? | 描述 |
| category | String | 分类：gallery/cover/magazine/livestream |
| sortOrder | Int | 排序 |

### FanGroup（后援会组）

| 字段 | 类型 | 说明 |
|------|------|------|
| id | Int (PK) | 自增 ID |
| name | String | 组名 |
| description | String? | 描述 |
| groupType | String | 类型：data/anti_black/admin/publicity |
| sortOrder | Int | 排序 |

### FanGroupLink（后援会链接）

| 字段 | 类型 | 说明 |
|------|------|------|
| id | Int (PK) | 自增 ID |
| label | String | 链接标签 |
| url | String | URL |
| groupId | Int (FK) | 关联分组 |
| sortOrder | Int | 排序 |

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
- 内容更新时间戳

### 数据获取
- Server Components + ISR（`revalidate: 60`）
- 公开页面无需认证
- 静态页面自动刷新，粉丝访问速度快

## 管理后台设计

### 认证方式
- Auth.js v5 + Credentials Provider
- 单管理员账号，密码 bcrypt 加密存储
- HTTP-only session cookies
- 登录失败速率限制（5 次/分钟）

### 路由保护
- `middleware.ts` 保护 `/admin/*` 路由
- 未认证访问重定向到登录页

### 后台功能

| 模块 | 操作 |
|------|------|
| 仪表盘 | 内容概览、最近更新记录 |
| 艺人资料 | 编辑/更新艺人信息 |
| 作品管理 | 新增/编辑/删除作品，状态切换 |
| 图片管理 | 上传/删除/拖拽排序 |
| 社交账号 | 管理各平台链接 |
| 后援会组 | 管理分组及链接 |
| 站点设置 | 封面图、标题、副标题 |

## 安全设计

- 管理后台路由 `/admin/*` 通过 middleware 保护
- 登录失败速率限制（5 次/分钟）
- 所有用户输入服务端 XSS 过滤
- CSRF 保护（Next.js 内置）
- 数据库连接使用 SSL（`sslmode=require`）
- 环境变量通过 Vercel Environment Variables 管理

## 图片存储策略

初始阶段将图片保留在 `public/photos/` 目录。后续可选迁移至 Vercel Blob：
- Vercel Blob 免费套餐：500MB 存储 + 5GB 带宽/月
- 当前图片总量约 10MB，远低于限制
- 迁移后可通过管理后台上传图片

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
- 11 张图片记录
- 5 个后援会组及其链接
- 站点配置（标题、副标题、封面图）

## 项目规范对齐

本项目为全新 Next.js 项目，无已有项目规范。采用以下社区标准：

- **组件库**：shadcn/ui（非 raw Radix UI）
- **样式方案**：Tailwind CSS utility-first
- **数据请求**：Prisma ORM 直接访问，不使用 raw fetch
- **API 规范**：Next.js App Router Server Actions + Route Handlers
- **类型安全**：全项目 TypeScript strict mode
- **代码风格**：ESLint + Prettier
