# AgrisaleWS iOS OTA 安装

此目录下的 `manifest.plist` 用于 **iOS 无线安装（OTA）**：用户在 iPhone 上用 Safari 打开 `itms-services://?action=download-manifest&url=...` 链接即可安装，无需连接 Mac。

## 发布新版本时需更新

1. **manifest.plist**
   - `url`：改为新版本 IPA 的完整 HTTPS 地址（如 123 云盘直链）。
   - `bundle-short-version`：与 IPA 的版本号一致（如 2.4.0）。
   - `bundle-version`：与 IPA 的构建号一致（见 Flutter `pubspec.yaml` 或 Xcode）。

2. **官网页面**
   - 若下载页有写死「当前版本」或 OTA 说明里的版本号，一并改为新版本。

## 安装链接（示例）

- Cloudflare Pages（agrisale.drflo.org）：  
  `itms-services://?action=download-manifest&url=https://agrisale.drflo.org/manifest/agrisalews/manifest.plist`
- GitHub Pages：  
  `itms-services://?action=download-manifest&url=https://flocio.github.io/Agrisale-Web/manifest/agrisalews/manifest.plist`

实际访问时，下载页上的「打开并安装」按钮会根据当前站点自动生成对应链接。

## 前提条件

- IPA 为 **Ad Hoc** 签名，且安装设备的 **UDID** 已加入该描述文件。
- manifest 与 IPA 均通过 **HTTPS** 提供；本站通过 `_headers` 将 `.plist` 以 `application/xml` 输出。
