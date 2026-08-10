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

## 2. 部署

我这里熟悉 Vercel 部署，就以 Vercel 部署为例子。创建 Project => Import 这个项目

![](https://www.yysuni.com/blogs/readme/730266f17fab9717.png)

无需配置，直接点部署

![](https://www.yysuni.com/blogs/readme/95dee9a69154d0d0.png)

大约 60 秒会部署完成，有一个直接 vercel 域名，如：https://2025-blog-public.vercel.app/

到这里部署网站已经完成了，下一步创建 Github App

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

设置完成后，需要手动再部署一次，让环境变量生效。
* 可以直接 push 一次仓库代码会触发部署
* 也可以手动选择创建一次部署
![](https://www.yysuni.com/blogs/readme/59a802ed8d1c3a13.png)

## 4. 完成

现在，部署的这个网站就可以开始使用前端改内容了。比如更改一个分享内容。

**提示**，网站前端页面删改完提示成功之后，你需要等待后台的部署完成，再刷新页面才能完成服务器内容的更新哦。

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

## 9. 互助群

对于完全不是**程序员**的用户，确实会对于更新代码后，如何同步，如何**合并代码**手足无措。我创建了一个 **QQ群**（加群会简单点），或者 vx 群还是 tg 群会好一点可以 issue 里面说下就行。

QQ 群：[https://qm.qq.com/q/spdpenr4k2](https://qm.qq.com/q/spdpenr4k2)
> 不好意思，之前的那个qq群ID（1021438316），不知道为啥搜不到😂

应该主要是我自己亲自帮助你们遇到问题怎么办。（后续看看有没有好心人）

希望多多的非程序员加入 blogger 行列，web blog 还是很好玩的，属于自己的 blog 世界。

游戏资产不一定属于你的，你只有**使用权**，但这个 blog **网站、内容、仓库一定是属于你的**

#### 特殊的导航 Card

因为这个 Card 是全局都在的，所以放在了 `src/components` 目录

![](https://www.yysuni.com/blogs/readme/9780c38f886322fd.png)
