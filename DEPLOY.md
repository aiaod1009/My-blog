# 博客自动化部署文档

> 使用 PM2 + GitHub Actions 实现 Next.js 博客的自动构建与部署

## 一、架构概览

```
┌─────────────────┐     git push     ┌──────────────────┐     SSH     ┌─────────────────────┐
│  本地开发环境     │ ──────────────→ │  GitHub 仓库      │ ──────────→ │  云服务器 (Ubuntu)   │
│  (Windows/VS Code)│                 │  GitHub Actions   │             │                     │
└─────────────────┘                  │  自动触发部署      │             │  Nginx (443)         │
                                      │                   │             │    ↓ 反代             │
                                      └──────────────────┘             │  Next.js (:3001)     │
                                                                       │  PM2 保活             │
                                                                       └─────────────────────┘
```

### 核心组件

| 组件 | 作用 |
|------|------|
| **GitHub** | 代码托管，触发自动部署 |
| **GitHub Actions** | 监听 `master` 分支推送，SSH 登录服务器执行部署脚本 |
| **PM2** | Node.js 进程管理，保活、崩溃重启、开机自启 |
| **Nginx** | SSL 终结、反向代理到本地 Next.js 服务 |
| **Next.js** | 博客应用本身，运行在 3001 端口 |

---

## 二、前提条件

- 一台 Ubuntu 云服务器（已配置域名 SSL）
- GitHub 仓库已关联本地项目
- 服务器已安装 Node.js（≥18）和 npm

---

## 三、服务器端配置

### 3.1 安装 PM2

```bash
npm install -g pm2
```

### 3.2 创建 PM2 配置文件

在项目根目录创建 `ecosystem.config.js`：

```js
module.exports = {
  apps: [
    {
      name: 'my-blog',
      script: 'node_modules/next/dist/bin/next',
      args: 'start -p 3001',
      cwd: '/home/ubuntu/My-blog',
      env: {
        NODE_ENV: 'production',
      },
      autorestart: true,
      instances: 1,
      error_file: 'logs/pm2-error.log',
      out_file: 'logs/pm2-out.log',
      log_date_format: 'YYYY-MM-DD HH:mm:ss',
    },
  ],
}
```

### 3.3 启动项目并设置开机自启

```bash
cd /home/ubuntu/My-blog

# 首次构建
npm run build

# 启动
pm2 start ecosystem.config.js

# 保存进程列表（重启后恢复）
pm2 save

# 配置开机自启（执行后会提示一行 sudo 命令，复制执行即可）
pm2 startup
```

### 3.4 Nginx 配置

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

修改后重载 Nginx：

```bash
sudo nginx -s reload
```

### 3.5 PM2 常用命令

```bash
pm2 list              # 查看所有进程
pm2 logs my-blog      # 查看日志
pm2 restart my-blog   # 重启
pm2 stop my-blog      # 停止
pm2 delete my-blog    # 删除进程
pm2 monit             # 实时监控面板
```

---

## 四、GitHub Actions 自动部署

### 4.1 创建工作流文件

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

**触发条件**：当 `master` 分支收到 `git push` 时自动执行。

**部署步骤**：
1. SSH 登录服务器
2. 拉取最新代码
3. 安装依赖
4. 构建项目
5. 重启 PM2 进程

### 4.2 配置 GitHub Secrets

打开 GitHub 仓库 → **Settings** → **Secrets and variables** → **Actions** → **New repository secret**。

需要添加 **3 个 Secret**：

| Name | Secret 值 | 说明 |
|------|-----------|------|
| `SERVER_HOST` | `aiaod.cn` | 服务器域名或 IP |
| `SERVER_USER` | `ubuntu` | SSH 登录用户名 |
| `SERVER_SSH_KEY` | `-----BEGIN OPENSSH PRIVATE KEY-----...` | 服务器 SSH 私钥全文 |

**获取服务器私钥**：

```bash
# 在服务器上执行
cat ~/.ssh/id_ed25519
```

复制全部内容（包括 `-----BEGIN` 和 `-----END` 行），粘贴到 `SERVER_SSH_KEY` 的值中。

### 4.3 验证部署

1. 推送代码到 GitHub：`git push`
2. 打开 GitHub 仓库 → **Actions** 标签页，查看工作流执行状态
3. 绿色 ✅ 表示部署成功
4. 访问 https://aiaod.cn 确认站点更新

> **注意**：GitHub Actions 的 Runner 部署在美国，首次触发后会通过 SSH 登录你的服务器，这是正常行为。SSH 密钥认证比密码更安全，无需担心。

---

## 五、日常开发工作流

### 5.1 修改博客内容

```bash
# 在本地修改代码

# 提交并推送（自动触发部署）
git add .
git commit -m "更新博客内容"
git push
```

### 5.2 仅修改配置（不触发部署）

如果只是想改 README 等不影响服务的文件，可以不在 commit message 中触发：

```bash
git commit -m "docs: update readme [skip ci]"
```

在 commit 中添加 `[skip ci]` 可以跳过 Actions 运行。

### 5.3 查看部署日志

- GitHub 仓库 → **Actions** 标签页 → 点击具体运行记录
- 服务器实时日志：`pm2 logs my-blog`
- Nginx 错误日志：`sudo tail -f /var/log/nginx/error.log`

---

## 六、故障排查

### 6.1 502 Bad Gateway

```
# 检查 Next.js 是否在运行
pm2 list

# 检查端口是否被监听
ss -tlnp | grep 3001

# 检查 Nginx 配置
sudo nginx -t
```

### 6.2 部署失败

去 GitHub Actions 查看具体报错，常见原因：

| 错误 | 原因 | 解决 |
|------|------|------|
| `permission denied` | SSH 密钥不匹配 | 重新配置 `SERVER_SSH_KEY` |
| `Host key verification failed` | 首次连接未知主机 | 在 workflow 添加 `fingerprint` 或先用 SSH 手动连一次 |
| `npm ERR!` | 依赖安装失败 | 检查 package.json 或 Node.js 版本 |

### 6.3 PM2 进程挂了

```bash
pm2 restart my-blog    # 手动重启
pm2 logs my-blog       # 查看崩溃原因
```

---

## 七、文件清单

```
my-blog/
├── .github/
│   └── workflows/
│       └── deploy.yml          # GitHub Actions 自动部署配置
├── ecosystem.config.js          # PM2 进程管理配置
├── package.json                 # 项目依赖与脚本
└── DEPLOY.md                    # 本部署文档
```

---

## 八、参考链接

- [PM2 官方文档](https://pm2.keymetrics.io/docs/usage/pm2-doc-single-page/)
- [GitHub Actions 文档](https://docs.github.com/zh/actions)
- [appleboy/ssh-action](https://github.com/appleboy/ssh-action)
- [Next.js 部署文档](https://nextjs.org/docs/pages/building-your-application/deploying)
