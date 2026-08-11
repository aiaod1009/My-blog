# 2025 Blog

> 最新引导说明：https://www.yysuni.com/blog/readme

该项目使用 Github App 管理项目内容，请保管好后续创建的 **Private key**，不要上传到公开网上。

---

## 项目架构概览

本项目基于 **Next.js 15+ App Router**，是一个支持 GitHub 托管内容管理的个人博客系统。

### 目录结构

```
src/
├── app/               # 所有页面路由（Next.js App Router）
│   ├── (home)/        # 首页 — 卡片式仪表盘
│   ├── about/         # /about 关于我
│   ├── blog/          # /blog 博客列表 & /blog/[id] 详情
│   ├── bloggers/      # /bloggers 友情链接
│   ├── image-toolbox/ # /image-toolbox 图片转 WebP 工具
│   ├── music/         # /music ⚠️ 未完成
│   ├── projects/      # /projects 项目展示
│   ├── rss.xml/       # /rss.xml RSS 订阅
│   ├── share/         # /share 推荐资源
│   ├── svgs/          # /svgs SVG 图标库
│   ├── write/         # /write 写文章 & /write/[slug] 编辑
│   └── layout.tsx     # 根布局
├── components/        # 跨页面共享组件（Card, NavCard, BlogPreview, LikeButton 等）
├── config/            # 站点静态配置
├── hooks/             # 全局 Hooks（useAuth, useBlogIndex, useCenter, useSize 等）
├── layout/            # 布局骨架（Header, Footer, 动态气泡背景）
├── lib/               # 工具库（GitHub API, 认证, Markdown 渲染）
├── styles/            # 全局 CSS
├── svgs/              # SVG 图标资源
└── consts.ts          # 应用常量（GitHub 配置等）
```

### 路由一览

| 路由 | 类型 | 说明 |
|------|------|------|
| `/` | 首页 | 卡片式仪表盘，可拖拽编辑布局 |
| `/about` | 页面 | 可在线编辑的关于我 |
| `/blog` | 列表 | 博客文章列表（按年份分组） |
| `/blog/[id]` | 详情 | Markdown 文章渲染 + TOC + 点赞 |
| `/bloggers` | 列表 | 友情链接网格 |
| `/projects` | 列表 | 项目展示网格 |
| `/share` | 列表 | 推荐资源网格 |
| `/svgs` | 页面 | SVG 图标库（搜索 + 复制） |
| `/write` | 编辑器 | 写新文章 |
| `/write/[slug]` | 编辑器 | 编辑已有文章 |
| `/image-toolbox` | 工具 | PNG/JPG 转 WebP |
| `/rss.xml` | RSS | RSS 订阅源 |

### 架构特点

- **Next.js 15 App Router** — 基于文件系统的路由
- **GitHub 即数据库** — 所有内容通过 GitHub API 读写，无需独立后端
- **Zustand 状态管理** — 全局和页面级状态管理
- **SWR 数据获取** — 浏览器端数据请求和缓存
- **motion 动画** — 页面过渡和卡片动画
- **Shiki 语法高亮** — Markdown 代码块高亮

---

## 1. 安装

使用该项目可以先不做本地开发，直接部署然后配置环境变量。具体变量名请看下列大写变量

```ts
export const GITHUB_CONFIG = {
	OWNER: process.env.NEXT_PUBLIC_GITHUB_OWNER || 'yysuni',
	REPO: process.env.NEXT_PUBLIC_GITHUB_REPO || '2025-blog-public',
	BRANCH: process.env.NEXT_PUBLIC_GITHUB_BRANCH || 'main',
	APP_ID: process.env.NEXT_PUBLIC_GITHUB_APP_ID || '-'
} as const
```

也可以自己手动先调整安装，可自行 `pnpm i`

## 2. 部署到自己的服务器

本项目部署在自己的服务器上（不使用 Vercel），通过 **PM2 + Nginx + GitHub Actions** 实现自动化部署。整体流程：

```
网页上改内容 / 本地 git push
        │
        ▼
GitHub 仓库收到新 commit
        │
        ▼
GitHub Actions 自动触发（.github/workflows/deploy.yml）
        │  SSH 登录服务器执行：
        │  git pull → npm install → npm run build → pm2 restart my-blog
        ▼
服务器上的 Next.js 完成更新（Nginx 反代 3001 端口）
```

### 2.1 服务器环境准备

- 一台云服务器（如 Ubuntu），域名已解析并配置好 SSL 证书
- 已安装 Node.js（≥18）和 npm

```bash
npm install -g pm2
git clone 你的仓库地址 /home/ubuntu/My-blog
cd /home/ubuntu/My-blog
npm install
npm run build
```

### 2.2 PM2 保活运行

项目根目录已包含 `ecosystem.config.js`（Next.js 运行在 3001 端口）：

```bash
pm2 start ecosystem.config.js
pm2 save
pm2 startup    # 执行后复制提示的 sudo 命令运行，实现开机自启
```

### 2.3 Nginx 反向代理

```nginx
server {
    listen 443 ssl http2;
    server_name 你的域名;

    ssl_certificate     /etc/letsencrypt/live/你的域名/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/你的域名/privkey.pem;

    location / {
        proxy_pass http://127.0.0.1:3001;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

```bash
sudo nginx -s reload
```

### 2.4 GitHub Actions 自动部署

仓库已包含 `.github/workflows/deploy.yml`：每次 push 到 `master` 分支，都会通过 SSH 登录服务器自动执行 `git pull → npm install → npm run build → pm2 restart my-blog`。

需要在 GitHub 仓库 **Settings → Secrets and variables → Actions** 中配置三个 Secrets：

| Name | 值 | 说明 |
|------|-----|------|
| `SERVER_HOST` | 服务器域名或 IP | 如 `aiaod.cn` |
| `SERVER_USER` | SSH 登录用户名 | 如 `ubuntu` |
| `SERVER_SSH_KEY` | 服务器私钥全文 | 在服务器执行 `cat ~/.ssh/id_ed25519` 获取 |

配置完成后，任何 push（包括网页上通过 GitHub App 提交的 commit）都会自动触发部署，约 1-3 分钟后网站内容更新。可在仓库的 **Actions** 标签页查看部署进度。

> 更详细的服务器部署说明和踩坑记录见 [blog.md](blog.md)。

## 3. 创建 Github App 链接仓库

在 github 个人设置里面，找到最下面的 Developer Settings ，点击进入

![](https://www.yysuni.com/blogs/readme/0abb3b592cbedad6.png)

进入开发者页面，点击 **New Github App**

*GitHub App name* 和 *Homepage URL* , 输入什么都不影响。Webhook 也关闭，不需要。

![](https://www.yysuni.com/blogs/readme/71dcd9cf8ec967c0.png)

只需要注意设置一个仓库 write 权限，其它不用。

![](https://www.yysuni.com/blogs/readme/2be290016e56cd34.png)

点击创建，谁能安装这个仓库这个选择无所谓。直接创建。

![](https://www.yysuni.com/blogs/readme/aa002e6805ab2d65.png)


### 创建密钥

创建好 Github App 后会提示必须创建一个 **Private Key**，直接创建，会自动下载（不见了也不要紧，后面自己再创建再下载就行）。页面上有个 **App ID** 需要复制一下

再切换到安装页面

![](https://www.yysuni.com/blogs/readme/c122b1585bb7a46a.png)

这里一定要只**授权当前项目**。

![](https://www.yysuni.com/blogs/readme/2cf1cee3b04326f1.png)

点击安装，就完成了 Github App 管理该仓库的权限设置了。下一步就是让前端知道推送那个项目，就是最开始提到的环境变量。（如果你不会设置环境变量，直接改仓库文件 `src/consts.ts` 也行。因为是公开的，所以环境变量意义也不大）

直接输入这几个环境变量值就行，一般只用设置 OWNER 和 APP_ID。其它配置不用管，直接输入创建就行。

![](https://www.yysuni.com/blogs/readme/c5a049d737848abf.png)

设置完成后，需要让新配置在服务器上生效：
* 直接 push 一次仓库代码，GitHub Actions 会自动完成部署
* 也可以 SSH 到服务器手动执行 `git pull && npm run build && pm2 restart my-blog`

## 4. 完成

现在，部署的这个网站就可以开始使用前端改内容了。比如更改一个分享内容。

**提示**，网站前端页面删改完提示成功之后，GitHub Actions 会自动触发服务器部署，你需要等待部署完成（约 1-3 分钟，可在仓库 Actions 页面查看进度），再刷新页面才能看到服务器内容的更新哦。

## 5. 删除

使用这个项目应该第一件事需要删除我的 blog，单独删除，批量删除已完成。

## 6. 配置

大部分页面右上角都会有一个编辑按钮，意味着你可以使用 **private key** 进行配置部署。

### 6.1 网站配置

首页有一个不显眼的配置按钮，点击就能看到现在可以配置的内容。

![](https://www.yysuni.com/blogs/readme/cddb4710e08a5069.png)

## 7. 写 blog

写 blog 的图片管理，可能会有疑惑。图片管理推荐逻辑是先点击 **+ 号** 添加图片，（推荐先压缩好，尺寸推荐宽度不超过 1200）。然后将上传好的图片直接拖入文案编辑区，这就已经添加好了，点击右上角预览就可以看到效果。

## 8. 写给非前端

非前端配置内容，还是需要一个文件指引。下面写一些更细致的代码配置。

### 8.1 移除 Liquid Grass

进入 `src/layout/index.tsx` 文件，删除两行代码，然后提交代码到你的 github
```tsx
const LiquidGrass = dynamic(() => import('@/components/liquid-grass'), { ssr: false })
// 中间省略...
<LiquidGrass /> // 第 53 行
```

![](https://www.yysuni.com/blogs/readme/f70ff3fe3a77f193.png)

### 8.2 配置首页内容

首页的内容现在只能前端配置一部分，所以代码更改在 `src/app/(home)` 目录，这个目录代表首页所有文件。首页的具体文件为  `src/app/(home)/page.tsx`

 ![](https://www.yysuni.com/blogs/readme/011679cd9bf73602.png)

这里可以看到有很多 `Card` 文件，需要改那个首页 Card 内容就可以点入那个具体文件修改。

比如中间的内容，为 `HiCard`，点击 `hi-card.tsx` 文件，即可更改其内容。

![](https://www.yysuni.com/blogs/readme/20b0791d012163ee.png)


#### 特殊的导航 Card

因为这个 Card 是全局都在的，所以放在了 `src/components` 目录

![](https://www.yysuni.com/blogs/readme/9780c38f886322fd.png)
