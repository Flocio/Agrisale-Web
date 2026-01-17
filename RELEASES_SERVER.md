# 树莓派下载服务器部署指南

本指南说明如何在树莓派（Ubuntu Server 24.04）上部署安装包下载服务，通过**独立的** Cloudflare Tunnel 暴露为 `releases.drflo.org`。

## 架构概览

```
agrisale.drflo.org    → Cloudflare Pages（官网）
agrisalews.drflo.org  → Cloudflare Tunnel (agrisalews-server) → 树莓派:9000（API）
releases.drflo.org    → Cloudflare Tunnel (releases-server)  → 树莓派:8000（下载服务）← 本指南
```

## 快速参考

| 项目 | 值 |
|------|-----|
| **Nginx 端口** | 8000 |
| **下载目录** | `~/releases/` |
| **配置文件** | `/etc/nginx/sites-available/releases` |
| **Tunnel 名称** | `releases-server` |
| **Tunnel 配置** | `/root/.cloudflared/config-releases.yml` |
| **systemd 服务** | `cloudflared-releases.service` |
| **域名** | `releases.drflo.org` |

---

## 第 1 步：创建目录结构

```bash
# 在树莓派上创建下载目录
mkdir -p ~/releases/agrisalews
mkdir -p ~/releases/agrisale

# 创建版本目录并上传安装包
mkdir -p ~/releases/agrisalews/v1.5.2
mkdir -p ~/releases/agrisale/v3.2.0

# 目录结构示例：
# ~/releases/
# ├── agrisalews/
# │   └── v1.5.2/
# │       ├── AgrisaleWS-android-v1.5.2.apk
# │       ├── AgrisaleWS-ios-v1.5.2.ipa
# │       ├── AgrisaleWS-macos-v1.5.2.dmg
# │       ├── AgrisaleWS-macos-v1.5.2.zip
# │       ├── AgrisaleWS-windows-v1.5.2-installer.exe
# │       └── AgrisaleWS-windows-v1.5.2.zip
# └── agrisale/
#     └── v3.2.0/
#         └── ...
```

---

## 第 2 步：安装和配置 Nginx

### 安装 Nginx

```bash
sudo apt update
sudo apt install nginx -y
```

### 创建配置文件

```bash
sudo nano /etc/nginx/sites-available/releases
```

写入以下内容：

```nginx
server {
    listen 8000;
    server_name localhost;
    
    # 下载目录
    root /home/你的用户名/releases;
    
    # 启用目录列表（可选，方便调试）
    autoindex on;
    autoindex_exact_size off;
    autoindex_localtime on;
    
    location / {
        try_files $uri $uri/ =404;
        
        # 允许跨域下载
        add_header Access-Control-Allow-Origin *;
        add_header Access-Control-Allow-Methods 'GET, OPTIONS';
        
        # 设置正确的 MIME 类型
        types {
            application/vnd.android.package-archive apk;
            application/octet-stream ipa dmg exe;
            application/zip zip;
        }
    }
    
    # 日志
    access_log /var/log/nginx/releases_access.log;
    error_log /var/log/nginx/releases_error.log;
}
```

**注意**：将 `你的用户名` 替换为实际的用户名（如 `pi` 或 `ubuntu`）。

### 启用配置

```bash
# 创建软链接
sudo ln -s /etc/nginx/sites-available/releases /etc/nginx/sites-enabled/

# 测试配置
sudo nginx -t

# 重新加载 Nginx
sudo systemctl reload nginx

# 设置开机自启
sudo systemctl enable nginx
```

### 验证服务

```bash
# 检查端口是否监听
sudo ss -tlnp | grep 8000

# 本地测试访问
curl http://localhost:8000/
```

---

## 第 3 步：创建独立的 Cloudflare Tunnel

### 创建新的 Tunnel

```bash
# 创建独立的 Tunnel 用于下载服务
cloudflared tunnel create releases-server
```

创建成功后，会显示：
```
Tunnel credentials written to /home/你的用户名/.cloudflared/<tunnel-id>.json
```

记下这个 `<tunnel-id>`，后面会用到。

### 配置 DNS 路由

```bash
# 将 releases.drflo.org 指向新创建的 Tunnel
cloudflared tunnel route dns releases-server releases.drflo.org
```

### 创建配置文件

```bash
# 创建 root 用户的 .cloudflared 目录（如果还没有）
sudo mkdir -p /root/.cloudflared

# 复制证书文件（如果还没有）
sudo cp ~/.cloudflared/cert.pem /root/.cloudflared/

# 复制新 Tunnel 的凭证文件
sudo cp ~/.cloudflared/<tunnel-id>.json /root/.cloudflared/

# 创建独立的配置文件
sudo nano /root/.cloudflared/config-releases.yml
```

写入以下内容（替换 `<tunnel-id>` 为实际的 Tunnel ID）：

```yaml
tunnel: <tunnel-id>
credentials-file: /root/.cloudflared/<tunnel-id>.json

ingress:
  - hostname: releases.drflo.org
    service: http://localhost:8000
  - service: http_status:404
```

### 创建独立的 systemd 服务

```bash
sudo nano /etc/systemd/system/cloudflared-releases.service
```

写入以下内容：

```ini
[Unit]
Description=Cloudflare Tunnel - Releases Server
After=network.target

[Service]
Type=simple
User=root
ExecStart=/usr/local/bin/cloudflared tunnel --config /root/.cloudflared/config-releases.yml run
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

### 启用并启动服务

```bash
# 重新加载 systemd 配置
sudo systemctl daemon-reload

# 启用开机自启
sudo systemctl enable cloudflared-releases

# 启动服务
sudo systemctl start cloudflared-releases

# 检查状态
sudo systemctl status cloudflared-releases

# 查看日志（如果有问题）
sudo journalctl -u cloudflared-releases -f
```

---

## 第 4 步：验证 DNS 配置

DNS 已经在第 3 步通过 `cloudflared tunnel route dns` 命令自动配置完成。

### 验证 DNS 记录

1. 登录 [Cloudflare Dashboard](https://dash.cloudflare.com)
2. 选择你的域名 `drflo.org`
3. 进入 **DNS** 设置
4. 应该能看到 `releases` CNAME 记录，指向 `<tunnel-id>.cfargotunnel.com`

### 如果之前绑定了 R2

先删除 `releases` 的 R2 绑定：
- R2 → 你的 Bucket → Settings → Custom domains → 删除 `releases.drflo.org`
- 然后删除 DNS 中的旧记录（如果有）

DNS 会被自动更新为指向新的 Tunnel。

---

## 第 5 步：上传安装包

### 使用 SCP 上传

```bash
# 从本地电脑上传到树莓派
scp /path/to/AgrisaleWS-android-v1.5.2.apk 用户名@树莓派IP:~/releases/agrisalews/v1.5.2/
scp /path/to/AgrisaleWS-ios-v1.5.2.ipa 用户名@树莓派IP:~/releases/agrisalews/v1.5.2/
# ... 其他文件
```

### 或使用 rsync（推荐）

```bash
rsync -avz --progress /path/to/安装包目录/ 用户名@树莓派IP:~/releases/agrisalews/v1.5.2/
```

---

## 第 6 步：更新 latest.json

更新 `api/agrisalews/latest.json` 中的下载链接：

```json
{
  "tag_name": "v1.5.2",
  "body": "...",
  "assets": [
    {
      "name": "AgrisaleWS-android-v1.5.2.apk",
      "browser_download_url": "https://releases.drflo.org/agrisalews/v1.5.2/AgrisaleWS-android-v1.5.2.apk"
    },
    {
      "name": "AgrisaleWS-ios-v1.5.2.ipa",
      "browser_download_url": "https://releases.drflo.org/agrisalews/v1.5.2/AgrisaleWS-ios-v1.5.2.ipa"
    },
    {
      "name": "AgrisaleWS-macos-v1.5.2.dmg",
      "browser_download_url": "https://releases.drflo.org/agrisalews/v1.5.2/AgrisaleWS-macos-v1.5.2.dmg"
    },
    {
      "name": "AgrisaleWS-macos-v1.5.2.zip",
      "browser_download_url": "https://releases.drflo.org/agrisalews/v1.5.2/AgrisaleWS-macos-v1.5.2.zip"
    },
    {
      "name": "AgrisaleWS-windows-v1.5.2-installer.exe",
      "browser_download_url": "https://releases.drflo.org/agrisalews/v1.5.2/AgrisaleWS-windows-v1.5.2-installer.exe"
    },
    {
      "name": "AgrisaleWS-windows-v1.5.2.zip",
      "browser_download_url": "https://releases.drflo.org/agrisalews/v1.5.2/AgrisaleWS-windows-v1.5.2.zip"
    }
  ]
}
```

---

## 第 7 步：验证部署

### 测试下载链接

```bash
# 测试是否可以访问
curl -I https://releases.drflo.org/agrisalews/v1.5.2/AgrisaleWS-android-v1.5.2.apk

# 检查 Cloudflare 节点（应该是亚洲节点如 HKG、SIN、NRT）
curl -I https://releases.drflo.org/agrisalews/v1.5.2/AgrisaleWS-android-v1.5.2.apk 2>/dev/null | grep cf-ray

# 测试下载速度
curl -o /dev/null -w "速度: %{speed_download} bytes/sec\n" \
  https://releases.drflo.org/agrisalews/v1.5.2/AgrisaleWS-android-v1.5.2.apk
```

### 在浏览器中测试

访问 `https://releases.drflo.org/` 应该能看到目录列表。

---

## 架构说明

### 独立 Tunnel 的好处

使用独立的 `releases-server` Tunnel，与 API 服务器的 `agrisalews-server` Tunnel 完全分离：

| 服务 | Tunnel | 端口 | systemd 服务 |
|------|--------|------|-------------|
| API 服务 | `agrisalews-server` | 9000 | `cloudflared-agrisalews.service` |
| 下载服务 | `releases-server` | 8000 | `cloudflared-releases.service` |

**优势**：
- ✅ 完全独立，互不干扰
- ✅ 可以独立启动、停止、重启
- ✅ 日志分离，方便排查问题
- ✅ 可以独立删除或升级

### 管理命令

```bash
# 查看所有 Tunnel
cloudflared tunnel list

# 查看下载服务状态
sudo systemctl status cloudflared-releases

# 重启下载服务
sudo systemctl restart cloudflared-releases

# 查看下载服务日志
sudo journalctl -u cloudflared-releases -f

# 停止下载服务（不影响 API）
sudo systemctl stop cloudflared-releases
```

---

## 常见问题

### Q: 403 Forbidden 错误

检查目录权限：
```bash
chmod 755 ~/releases
chmod -R 644 ~/releases/*
chmod 755 ~/releases/agrisalews ~/releases/agrisalews/v1.5.2
```

### Q: 502 Bad Gateway 错误

检查 Nginx 是否运行：
```bash
sudo systemctl status nginx
sudo journalctl -u nginx -f
```

### Q: Cloudflare Tunnel 连接问题

检查 Tunnel 状态：
```bash
sudo systemctl status cloudflared
cloudflared tunnel info 你的tunnel名称
```

---

## 发布新版本流程

1. 构建新版本安装包
2. 在树莓派上创建新版本目录：`mkdir -p ~/releases/agrisalews/v新版本号`
3. 上传安装包到新目录
4. 更新 `api/agrisalews/latest.json`
5. 更新官网下载页 `agrisalews/index.html`（如果需要）
6. 提交并推送 Agrisale-Web 仓库
