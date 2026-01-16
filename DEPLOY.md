# Agrisale 官网部署指南

本文档介绍如何将 Agrisale 官网部署到 Cloudflare Pages，并配置 Cloudflare R2 存储安装包。

## 目录

1. [Cloudflare Pages 部署](#cloudflare-pages-部署)
2. [Cloudflare R2 配置](#cloudflare-r2-配置)
3. [域名配置](#域名配置)
4. [版本更新流程](#版本更新流程)

---

## Cloudflare Pages 部署

### 方式一：通过 GitHub 自动部署（推荐）

1. **推送代码到 GitHub**
   ```bash
   cd /path/to/Agrisale-Web
   git init
   git add .
   git commit -m "Initial commit"
   git remote add origin https://github.com/Flocio/Agrisale_Web.git
   git branch -M main
   git push -u origin main
   ```

2. **登录 Cloudflare Dashboard**
   - 访问 [Cloudflare Dashboard](https://dash.cloudflare.com/)
   - 进入 "Workers & Pages"

3. **创建 Pages 项目**
   - 点击 "Create application"
   - 选择 "Pages"
   - 点击 "Connect to Git"
   - 选择 GitHub 账户并授权
   - 选择 `Flocio/Agrisale_Web` 仓库

4. **配置构建设置**
   - **Project name**: `agrisale-web`
   - **Production branch**: `main`
   - **Framework preset**: None
   - **Build command**: 留空（纯静态网站，无需构建）
   - **Build output directory**: `/`（根目录）

5. **部署**
   - 点击 "Save and Deploy"
   - 等待部署完成

### 方式二：直接上传

1. 在 Cloudflare Pages 中选择 "Upload assets"
2. 将整个项目目录拖拽上传

---

## Cloudflare R2 配置

### 1. 创建 R2 Bucket

1. **登录 Cloudflare Dashboard**
   - 进入 "R2 Object Storage"

2. **创建 Bucket**
   - 点击 "Create bucket"
   - **Bucket name**: `agrisale-releases`
   - **Location**: 选择离用户最近的区域（推荐 APAC）
   - 点击 "Create bucket"

### 2. 上传安装包

**目录结构：**
```
agrisale-releases/
├── agrisale/
│   └── v3.2.0/
│       ├── Agrisale-android-v3.2.0.apk
│       ├── Agrisale-ios-v3.2.0.ipa
│       ├── Agrisale-macos-v3.2.0.dmg
│       ├── Agrisale-macos-v3.2.0.zip
│       ├── Agrisale-windows-v3.2.0-installer.exe
│       └── Agrisale-windows-v3.2.0.zip
└── agrisalews/
    └── v1.5.2/
        ├── AgrisaleWS-android-v1.5.2.apk
        ├── AgrisaleWS-ios-v1.5.2.ipa
        ├── AgrisaleWS-macos-v1.5.2.dmg
        ├── AgrisaleWS-macos-v1.5.2.zip
        ├── AgrisaleWS-windows-v1.5.2-installer.exe
        └── AgrisaleWS-windows-v1.5.2.zip
```

**上传方式：**

**方式一：通过 Dashboard 上传**
1. 进入 `agrisale-releases` bucket
2. 点击 "Upload"
3. 创建文件夹并上传对应文件

**方式二：通过 Wrangler CLI 上传（推荐大量文件）**
```bash
# 安装 wrangler
npm install -g wrangler

# 登录
wrangler login

# 上传文件
wrangler r2 object put agrisale-releases/agrisale/v3.2.0/Agrisale-android-v3.2.0.apk --file=./Agrisale-android-v3.2.0.apk
wrangler r2 object put agrisale-releases/agrisale/v3.2.0/Agrisale-ios-v3.2.0.ipa --file=./Agrisale-ios-v3.2.0.ipa
# ... 其他文件
```

### 3. 配置公开访问

1. **进入 Bucket 设置**
   - 在 R2 Dashboard 中选择 `agrisale-releases`
   - 点击 "Settings"

2. **启用公开访问**
   - 找到 "Public access"
   - 点击 "Allow Access"
   - 确认启用

### 4. 绑定自定义域名

1. **在 R2 Bucket 设置中**
   - 找到 "Custom Domains"
   - 点击 "Connect Domain"

2. **配置域名**
   - 输入域名：`releases.drflo.org`
   - 点击 "Continue"

3. **DNS 配置（如果域名在 Cloudflare）**
   - Cloudflare 会自动添加 CNAME 记录
   - 如果域名不在 Cloudflare，需要手动添加：
     ```
     Type: CNAME
     Name: releases
     Content: <bucket-url>.r2.cloudflarestorage.com
     Proxy status: Proxied
     ```

4. **等待生效**
   - 域名配置通常在几分钟内生效
   - 可以访问 `https://releases.drflo.org` 测试

### 5. 配置 CORS（跨域访问）

1. **在 Bucket 设置中找到 "CORS Policy"**

2. **添加 CORS 规则**
   ```json
   [
     {
       "AllowedOrigins": [
         "https://agrisale.drflo.org",
         "https://*.pages.dev"
       ],
       "AllowedMethods": [
         "GET",
         "HEAD"
       ],
       "AllowedHeaders": [
         "*"
       ],
       "ExposeHeaders": [
         "Content-Length",
         "Content-Type"
       ],
       "MaxAgeSeconds": 3600
     }
   ]
   ```

3. **保存配置**

---

## 域名配置

### 配置 agrisale.drflo.org

1. **在 Cloudflare DNS 设置中**
   - 域名：`drflo.org`
   - 添加 CNAME 记录：
     ```
     Type: CNAME
     Name: agrisale
     Content: agrisale-web.pages.dev
     Proxy status: Proxied
     ```

2. **在 Pages 项目中添加自定义域名**
   - 进入 Pages 项目设置
   - "Custom domains" → "Set up a custom domain"
   - 输入：`agrisale.drflo.org`
   - 点击 "Activate domain"

---

## 版本更新流程

当发布新版本时，按以下步骤操作：

### 1. 上传新安装包到 R2

```bash
# 以 Agrisale v3.3.0 为例
wrangler r2 object put agrisale-releases/agrisale/v3.3.0/Agrisale-android-v3.3.0.apk --file=./Agrisale-android-v3.3.0.apk
wrangler r2 object put agrisale-releases/agrisale/v3.3.0/Agrisale-ios-v3.3.0.ipa --file=./Agrisale-ios-v3.3.0.ipa
wrangler r2 object put agrisale-releases/agrisale/v3.3.0/Agrisale-macos-v3.3.0.dmg --file=./Agrisale-macos-v3.3.0.dmg
wrangler r2 object put agrisale-releases/agrisale/v3.3.0/Agrisale-macos-v3.3.0.zip --file=./Agrisale-macos-v3.3.0.zip
wrangler r2 object put agrisale-releases/agrisale/v3.3.0/Agrisale-windows-v3.3.0-installer.exe --file=./Agrisale-windows-v3.3.0-installer.exe
wrangler r2 object put agrisale-releases/agrisale/v3.3.0/Agrisale-windows-v3.3.0.zip --file=./Agrisale-windows-v3.3.0.zip
```

### 2. 更新网站文件

**更新 `agrisale/index.html`：**
- 修改版本号显示：`v3.2.0` → `v3.3.0`
- 更新下载链接中的版本号
- 添加新版本的更新说明

**更新 `api/agrisale/latest.json`：**
```json
{
  "version": "3.3.0",
  "releaseDate": "2026-XX-XX",
  // ... 更新其他字段
}
```

### 3. 提交并推送

```bash
git add .
git commit -m "Release Agrisale v3.3.0"
git push
```

Cloudflare Pages 会自动部署更新。

### 4. 版本更新检查清单

- [ ] 安装包已上传到 R2（6 个文件）
- [ ] 下载页 HTML 已更新（版本号、下载链接、更新说明）
- [ ] API JSON 已更新
- [ ] 代码已推送到 GitHub
- [ ] 网站已自动部署
- [ ] 测试下载链接是否正常

---

## 常见问题

### Q: 下载链接 404？
A: 检查：
1. R2 文件是否已上传
2. 文件路径是否正确（注意大小写）
3. R2 公开访问是否已启用
4. 自定义域名是否配置正确

### Q: CORS 错误？
A: 检查 R2 的 CORS 配置是否包含了网站域名

### Q: 部署失败？
A: 检查：
1. GitHub 仓库权限
2. 构建日志中的错误信息

### Q: 域名无法访问？
A: 检查：
1. DNS 记录是否正确
2. SSL 证书是否已签发（可能需要几分钟）
3. Proxy 状态是否为 Proxied

---

## 快速命令参考

```bash
# 查看 R2 Bucket 内容
wrangler r2 object list agrisale-releases

# 删除文件
wrangler r2 object delete agrisale-releases/path/to/file

# 下载文件
wrangler r2 object get agrisale-releases/path/to/file --file=./local-file

# 部署 Pages（手动触发）
# 在 Cloudflare Dashboard 中点击 "Retry deployment"
```
