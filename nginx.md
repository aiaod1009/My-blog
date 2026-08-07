# 1. 主域名 + www 子域名 → 博客（反代 Vercel）
server {
    listen 443 ssl http2;
    server_name aiaod.cn www.aiaod.cn;

    ssl_certificate /etc/letsencrypt/live/aiaod.cn/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/aiaod.cn/privkey.pem;

    location / {
        proxy_pass http://127.0.0.1:3001;
        proxy_ssl_server_name on;
    }
}

# 2. chiaroscuro 子域名 → Vue 网站
server {
    listen 443 ssl http2;
    server_name chiaroscuro.aiaod.cn;

    ssl_certificate /etc/letsencrypt/live/aiaod.cn/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/aiaod.cn/privkey.pem;

    root /home/ubuntu/chiaroscuro/frontend/dist;
    index index.html;

    location / {
        try_files $uri $uri/ /index.html;
    }

    location /api {
        proxy_pass http://127.0.0.1:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}

# 3. 自动将 HTTP 重定向到 HTTPS（通配所有子域名）
server {
    listen 80;
    server_name aiaod.cn www.aiaod.cn chiaroscuro.aiaod.cn;
    return 301 https://$host$request_uri;
}

