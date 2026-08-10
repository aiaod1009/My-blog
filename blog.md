# 博客自动化部署文档

> 使用 PM2 + GitHub Actions + GitHub App 实现 Next.js 博客的无后端自动构建与部署

---

## 一、整体架构

### 核心思想：怎么做到"不用后端"？

传统网站需要后端服务器 + 数据库来存储和修改内容。这个项目换了一种思路：

```
用户修改内容（两种方式）
       │
       ├── 方式 A：在网站上直接可视化编辑
       │      └── 通过 GitHub App 直接修改 GitHub 仓库里的文件
       │
       └── 方式 B：本地改代码后 git push
              └── 代码推送到 GitHub 仓库
      
                      │
                      ▼
              GitHub 仓库（代码 + 数据的"真相来源"）
                      │
                      ▼
          GitHub Actions（自动触发部署）
                      │
                      ▼
              服务器（运行 Next.js，用户访问）
              PM2 保活 + Nginx 反代
```

**GitHub 仓库同时扮演了"代码托管"和"数据库"两个角色。** 你在网站上修改内容，实际上是让 GitHub App 帮你把改动提交到仓库里——相当于可视化地操作代码文件，而不是通过传统数据库。

### 两种更新方式对比

| 方式 | 操作 | 适合 |
|------|------|------|
| **网站上直接编辑** | 打开配置面板，改颜色/写文章/传图片 | 快速改内容，无需电脑 |
| **本地改代码 push** | `git add` → `git commit` → `git push` | 改代码/加功能/批量操作 |

两种方式最终都会触发 GitHub Actions 自动部署到服务器，完全不需要手动 SSH。

### 数据流详解

```
┌─ 方式 A：网站可视化编辑 ──────────────────────────────────────┐
│  网站配置面板                                                 │
│    ↓ 使用 GitHub App（Private Key 签发 JWT）                   │
│    ↓ 通过 GitHub API 直接 commit 到仓库                        │
│  site-content.json / 文章 .md 等文件被修改                     │
│    ↓                                                            │
│  仓库收到新 commit → GitHub Actions 触发部署                    │
└──────────────────────────────────────────────────────────────┘

┌─ 方式 B：本地代码提交 ────────────────────────────────────────┐
│  本地改代码 → git add → git commit → git push                  │
│    ↓                                                            │
│  仓库收到新 commit → GitHub Actions 触发部署                    │
└──────────────────────────────────────────────────────────────┘
                               │
                               ▼
┌─ GitHub Actions 部署流程 ─────────────────────────────────────┐
│  SSH 登录服务器                                                │
│  cd /home/ubuntu/My-blog                                      │
│  git pull origin master          # 拉取最新代码                │
│  npm install                     # 安装依赖                    │
│  npm run build                   # 构建                        │
│  pm2 restart my-blog             # 重启服务                    │
└──────────────────────────────────────────────────────────────┘
                               │
                               ▼
┌─ 服务器架构 ──────────────────────────────────────────────────┐
│  用户访问 https://aiaod.cn                                     │
│    ↓                                                           │
│  Nginx（443端口，SSL 证书）                                     │
│    ↓ 反向代理                                                  │
│  Next.js（3001 端口，PM2 保活）                                 │
└──────────────────────────────────────────────────────────────┘
```

---

## 二、核心组件

| 组件 | 作用 |
|------|------|
| **GitHub 仓库** | 代码托管 + 数据存储（替代数据库） |
| **GitHub App** | 授权网站直接读写仓库文件（替代后端 API） |
| **GitHub Actions** | 监听 push，自动 SSH 到服务器部署 |
| **PM2** | Node.js 进程管理，保活、崩溃重启、开机自启 |
| **Nginx** | SSL 终结、反向代理到本地 Next.js 服务 |
| **Next.js** | 博客应用本身，运行在 3001 端口 |

---

## 三、GitHub App 配置（核心！替代后端的方案）

### 3.1 为什么要用 GitHub App？

这个项目**没有传统后端**，那在网站上改文章、改颜色是怎么保存的？

```
传统架构：                   本项目的架构：
浏览器 → API 服务器 → 数据库   浏览器 → GitHub API → GitHub 仓库
         （你需要自己写）               （GitHub 帮你做了）
```

GitHub App 的作用就是**授权网站直接读写你的 GitHub 仓库文件**，相当于 GitHub 替你当后端。

### 3.2 创建 GitHub App

1. 打开 https://github.com/settings/apps/new

2. 填写基本信息：
   - **GitHub App name**：随便写，如 `my-blog-app`
   - **Homepage URL**：随便写，如 `https://aiaod.cn`
   - **Webhook**：取消勾选 "Active"（不需要）

3. 设置权限（最关键！）：
   - **Repository permissions → Contents**：设为 **Read & write**

4. 点击 **Create GitHub App**

5. **记下 App ID**（页面顶部显示的数字）

6. 生成私钥：
   - 往下翻到 **Private keys**
   - 点击 **Generate a private key**
   - 自动下载一个 `.pem` 文件（**务必保管好，丢了就要重新生成**）

7. 安装 App 到仓库：
   - 左侧点击 **Install App**
   - 点击 **Install** → 选择 `Only select repositories`
   - 选中你的博客仓库（如 `aiaod1009/My-blog`）
   - 点击 **Install**

### 3.3 配置 App ID

在 `src/consts.ts` 中配置你的 App ID 和相关仓库信息：

```ts
export const GITHUB_CONFIG = {
    OWNER: '你的GitHub用户名',       // 如 aiaod1009
    REPO: '你的仓库名',              // 如 My-blog
    BRANCH: '你的默认分支',          // 如 master
    APP_ID: '你的App ID'            // 如 4502227
}
```

### 3.4 在网站上导入私钥

1. 打开博客首页 → 点击配置按钮 → 点 **保存**
2. 弹窗选择你的 `.pem` 文件
3. 网站会使用这个密钥去操作你的 GitHub 仓库

> **注意**：这个私钥是你操作 GitHub 仓库的凭证，相当于网站版的 SSH 密钥。

### 3.5 工作原理

```
网站要保存配置时：
  1. 用 App ID + Private Key → 生成 JWT（JSON Web Token）
  2. 用 JWT → 获取 Installation Token
  3. 用 Token → 调用 GitHub API 修改仓库文件
  4. GitHub 收到新 commit → 触发 Actions 自动部署
```

---

## 四、服务器端配置（仿 Vercel 自动化部署）

### 4.1 前提条件

- 一台 Ubuntu 云服务器
- 域名已解析到服务器并配置 SSL 证书
- 已安装 Node.js（≥18）和 npm

### 4.2 安装 PM2

```bash
npm install -g pm2
```

### 4.3 拉取代码

```bash
git clone 你的仓库地址 /home/ubuntu/My-blog
cd /home/ubuntu/My-blog
npm install
npm run build
```

### 4.4 创建 PM2 配置

项目根目录创建 `ecosystem.config.js`：

```js
module.exports = {
  apps: [
    {
      name: 'my-blog',
      script: 'node_modules/next/dist/bin/next',
      args: 'start -p 3001',
      cwd: '/home/ubuntu/My-blog',
      env: { NODE_ENV: 'production' },
      autorestart: true,
      instances: 1,
      error_file: 'logs/pm2-error.log',
      out_file: 'logs/pm2-out.log',
      log_date_format: 'YYYY-MM-DD HH:mm:ss',
    },
  ],
}
```

### 4.5 启动并设置开机自启

```bash
pm2 start ecosystem.config.js
pm2 save
pm2 startup    # 执行后复制提示的 sudo 命令运行
```

### 4.6 Nginx 配置

```nginx
server {
    listen 443 ssl http2;
    server_name aiaod.cn www.aiaod.cn;

    ssl_certificate /etc/letsencrypt/live/aiaod.cn/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/aiaod.cn/privkey.pem;

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

### 4.7 PM2 常用命令

```bash
pm2 list              # 查看所有进程
pm2 logs my-blog      # 查看日志
pm2 restart my-blog   # 重启
pm2 stop my-blog      # 停止
pm2 delete my-blog    # 删除进程
pm2 monit             # 实时监控面板
```

---

## 五、GitHub Actions 自动部署（仿 Vercel）

### 5.1 工作流原理

每次 push 代码到 GitHub，都会自动触发部署，就像 Vercel 一样。

### 5.2 创建工作流

在项目根目录创建 `.github/workflows/deploy.yml`：

```yaml
name: 自动部署到服务器

on:
  push:
    branches: [master]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: SSH 连接并部署
        uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.SERVER_HOST }}
          username: ${{ secrets.SERVER_USER }}
          key: ${{ secrets.SERVER_SSH_KEY }}
          script: |
            cd /home/ubuntu/My-blog
            git pull origin master
            npm install
            npm run build
            pm2 restart my-blog
```

### 5.3 配置 GitHub Secrets

打开 GitHub 仓库 → **Settings** → **Secrets and variables** → **Actions** → **New repository secret**

| Name | 值 | 说明 |
|------|-----|------|
| `SERVER_HOST` | `aiaod.cn` | 服务器域名或 IP |
| `SERVER_USER` | `ubuntu` | SSH 登录用户名 |
| `SERVER_SSH_KEY` | 服务器私钥全文 | 在服务器执行 `cat ~/.ssh/id_ed25519` 获取 |

### 5.4 验证自动部署

1. 推送代码到 GitHub：`git push`
2. 打开 GitHub 仓库 → **Actions** 标签页
3. 绿色 ✅ 表示部署成功
4. 访问网站确认更新

---

## 六、日常开发工作流

### 6.1 改内容（两种方式任选）

**方式 A：在网站上直接改**
```
打开网站 → 配置面板 → 改颜色/改标题/写文章 → 保存
```

**方式 B：本地改代码推送到 GitHub**
```bash
git add .
git commit -m "更新内容"
git push
# 等待 1-2 分钟自动部署完成
```

### 6.2 跳过自动部署

如果只是改文档，可以跳过部署：

```bash
git commit -m "docs: update [skip ci]"
```

### 6.3 查看部署状态

- GitHub 仓库 → **Actions** 标签页
- 服务器实时日志：`pm2 logs my-blog`
- Nginx 错误日志：`sudo tail -f /var/log/nginx/error.log`

---

## 七、踩坑记录

### 🕳️ 坑 1：VPN 导致 GitHub API 连接超时

**症状**：在网站上保存配置时失败，浏览器 Console 显示 `ERR_CONNECTION_TIMED_OUT`

**原因**：VPN 拦截了 `api.github.com` 的请求，导致连接超时

**解决**：**关掉 VPN**，让浏览器直连 GitHub API

> **为什么服务器上没问题？** 因为服务器有独立的网络出口，不受你本地 VPN 影响。

### 🕳️ 坑 2：分支名写错导致 API 404

**症状**：保存配置时 Console 显示 `GET https://api.github.com/repos/*/*/git/ref/heads/main 404`

**原因**：`src/consts.ts` 里 `BRANCH` 写的是 `'main'`，但仓库默认分支是 `master`

**解决**：将代码中的分支名改为实际的分支名：
```ts
BRANCH: 'master'
```

### 🕳️ 坑 3：TypeScript 不认识 CSS/图片导入

**症状**：编辑器中 `import '@/styles/globals.css'` 报红

**原因**：TypeScript 不认识 `.css` `.png` 等非代码文件

**解决**：在 `global.d.ts` 中添加声明：
```ts
declare module '*.css'
declare module '*.png'
declare module '*.jpg'
declare module '*.jpeg'
declare module '*.gif'
declare module '*.webp'
```

### 🕳️ 坑 4：.next 目录权限问题导致构建失败

**症状**：构建时报 `EACCES: permission denied`

**原因**：之前用 `sudo` 构建过，导致 `.next` 目录归 root 所有

**解决**：
```bash
sudo chown -R ubuntu:ubuntu /home/ubuntu/My-blog
rm -rf .next
npm run build
```

### 🕳️ 坑 5：图片上传 422

**症状**：上传头像时返回 422

**原因**：GitHub API 对文件大小或格式有限制

**解决**：压缩图片、使用标准格式（PNG/JPG）、避免中文文件名

### 🕳️ 坑 6：502 Bad Gateway

**症状**：网站突然 502

**原因**：Next.js 进程重启中，或者 PM2 挂了

**解决**：
```bash
pm2 list                          # 检查进程状态
pm2 restart my-blog               # 重启
pm2 logs my-blog --lines 50       # 查看日志找原因
```

---

## 八、文件清单

```
my-blog/
├── .github/
│   └── workflows/
│       └── deploy.yml            # GitHub Actions 自动部署配置
├── ecosystem.config.js            # PM2 进程管理配置
├── src/
│   ├── consts.ts                  # GitHub 仓库配置（OWNER/REPO/BRANCH/APP_ID）
│   └── lib/
│       ├── github-client.ts       # GitHub API 调用封装
│       └── auth.ts                # 认证 Token 获取
├── global.d.ts                    # TypeScript 模块声明
├── package.json
└── DEPLOY.md                      # 本部署文档
```

---

## 九、参考链接

- [PM2 官方文档](https://pm2.keymetrics.io/docs/usage/pm2-doc-single-page/)
- [GitHub Actions 文档](https://docs.github.com/zh/actions)
- [GitHub App 创建指南](https://docs.github.com/zh/apps/creating-github-apps)
- [appleboy/ssh-action](https://github.com/appleboy/ssh-action)
- [Next.js 部署文档](https://nextjs.org/docs/pages/building-your-application/deploying)

---

## 十、Q&A

### Q：为什么不用数据库？
A：这个项目走的是"GitHub 即数据库"的路线。所有内容都存为仓库里的文件，没有传统后端和数据库，省去了维护后端的成本。

### Q：Private Key 丢了怎么办？
A：去 GitHub App 设置页面重新生成一个，然后在网站上重新导入。

### Q：网站还能恢复之前基于 Vercel 的部署吗？
A：可以。GitHub 仓库里的代码是完整的，随时可以重新部署到 Vercel。当前服务器部署只是增加了一种部署方式。

### Q：自动部署大概延迟多久？
A：从 push 代码到部署完成，通常 1-3 分钟。可以在 GitHub Actions 页面查看实时进度。
