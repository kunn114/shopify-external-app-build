# Shopify Admin API Token 生成器（Embedded = false）

通过 OAuth 2.0 授权码流程，为 Shopify App 生成 **Admin API Access Token**，供后端服务调用 GraphQL / REST Admin API。

本项目为**纯静态页面**（无后端、无 App Bridge）。OAuth 授权在浏览器完成；**换取 Token 须在本地终端执行 CURL**（Shopify Token 接口不允许浏览器跨域调用）。

**不嵌入 Admin iframe**（Embedded = false）。从 **Admin → Apps 菜单** 进入时，Shopify 会在**新标签页**打开 App URL（例如 GitHub Pages），地址栏应为你的部署域名，而非 `admin.shopify.com`。

## 适用场景

- 内部工具：为指定店铺生成 Offline / Online Token，配置到后端环境变量
- 开发调试：Partner Dashboard 中 **关闭 Embed**，用 GitHub Pages 等静态托管快速授权
- 无服务端：接受在本地终端执行 CURL 完成最后一步换 Token

## App 配置要求

在 [Shopify Partner Dashboard](https://partners.shopify.com/)（或 [dev.shopify.com/dashboard](https://dev.shopify.com/dashboard)）中为你的 App 完成以下设置。

### 新版 Versions 配置

Redirect URL 在 **App → Versions** 中配置（需 **Release** 后生效）：

| 配置项 | 说明 |
|--------|------|
| **Embed app in Shopify admin** | **关闭**（`embedded: false`） |
| **App URL** | 本页 HTTPS 地址，例如 `https://kunn114.github.io/shopify-external-app-build/token-generator-not-embedded` |
| **Redirect URLs** | 与 App URL **完全相同** |
| **Admin API scopes** | 生成 Token 时填写的 Scopes 须与此处一致 |
| **Use legacy install flow** | 通常保持 `false` |

### Redirect URL 格式（重要）

- 须与页面顶部显示的 **Redirect URL** **逐字一致**
- **不要**带尾部斜杠 `/`
- **不要**写成 `/index.html`
- **不要**使用 `http://`（须 HTTPS）

正确示例：

```
https://kunn114.github.io/shopify-external-app-build/token-generator-not-embedded
```

错误示例（会导致 `redirect_uri is not whitelisted`）：

```
https://kunn114.github.io/shopify-external-app-build/token-generator-not-embedded/
```

## Token 类型

| 按钮 | Shopify 类型 | 说明 |
|------|--------------|------|
| **生成永久 Offline Token** | Offline（非过期） | 店铺级 Token，在 App 未卸载、Secret 未撤销前长期有效，适合后端定时任务、Webhook 等无用户在线场景。 |
| **生成非永久 Offline Token** | Offline（过期 + 可刷新） | 店铺级 Token，access_token 约 60 分钟过期，响应含 `refresh_token` 可续期；换取时在 CURL 中附加 `expiring=1`。 |
| **生成非永久 Online Token** | Online | 绑定当前授权员工，随会话失效（通常约 24 小时或登出后失效），适合需尊重员工权限的交互场景。 |

技术实现：

- **永久 Offline**：授权 URL 不传 `grant_options[]`；换 Token 的 CURL 不含 `expiring`。
- **非永久 Offline**：授权 URL 同样不传 `grant_options[]`；换 Token 的 CURL 附加 `"expiring":"1"`。
- **非永久 Online**：授权 URL 增加 `grant_options[]=per-user`；换 Token 的 CURL 不含 `expiring`。

## 使用步骤

1. 将本目录部署到静态站点（见下方 [GitHub Pages 部署](#github-pages-部署)），确保可通过 HTTPS 访问。
2. Partner Dashboard 中 **关闭** Embed，App URL 与 Redirect URL 均指向本页（**无尾部斜杠**），并 **Release** 版本。
3. 在开发店铺 **Admin → Apps** 中打开本 App（会**新标签页**打开）。
4. 打开页面，确认顶部 **Redirect URL** 与 Partner Dashboard 中配置的一致。
5. 填写：
   - **店铺域名**：如 `your-store.myshopify.com`；从 Admin 打开时 URL 通常带 `shop` 参数并自动填入。
   - **Client ID**：App 的 API Key（Partner Dashboard → API credentials）。
   - **Secret**：App 的 API Secret Key。
   - **Scopes**：与 App 配置的 Admin API scopes 一致，例如 `read_products,read_discounts`。
6. 点击 **生成永久 Offline Token**、**生成非永久 Offline Token** 或 **生成非永久 Online Token**，在 Shopify 授权页同意。
7. 回到本页，在 **CURL 命令** 区域选择 **Windows / macOS / Linux**（默认按浏览器环境自动选中），点击 **复制命令**。
8. 在本地终端粘贴并执行；响应 JSON 中的 `access_token` 即为 Token。

## OAuth 流程简述

```
用户填写 Client ID / Secret / Scopes / Shop
        ↓
点击生成按钮 → 跳转 Shopify 授权页
  https://{shop}.myshopify.com/admin/oauth/authorize?...
        ↓
用户同意 → 回调至 Redirect URL（带 code、state、hmac、shop）
        ↓
本页校验 state、HMAC → 展示 CURL 命令（含 code）
        ↓
用户在本地终端执行 CURL → POST /admin/oauth/access_token
        ↓
终端返回 JSON，读取 access_token
```

- **state**：防 CSRF，保存在 `sessionStorage`（键名 `tgne_oauth_state`）。
- **HMAC**：使用 Secret 校验回调参数完整性。
- 表单信息在授权前写入 `sessionStorage`（前缀 `tgne_*`），以便回调后生成 CURL。

授权页可能显示在 `admin.shopify.com/store/{shop}/oauth/authorize`，这是 Shopify 的正常行为；关键是 `redirect_uri` 参数与 Partner Dashboard 白名单一致。

## 为何使用 CURL

浏览器调用 `POST https://{shop}.myshopify.com/admin/oauth/access_token` 会被 **CORS** 拦截（控制台常见 `Failed to fetch`）。纯静态部署无法添加服务端代理，因此在授权回调校验通过后，在页面生成 CURL，由用户在本地终端完成 code → token 交换。

### CURL 示例（macOS / Linux）

授权成功后，页面会生成包含真实参数的命令，形态如下：

```bash
curl -s -X POST "https://your-store.myshopify.com/admin/oauth/access_token" \
  -H "Content-Type: application/json" \
  -d "{\"client_id\":\"YOUR_CLIENT_ID\",\"client_secret\":\"YOUR_SECRET\",\"code\":\"AUTHORIZATION_CODE\"}"
```

### Windows（PowerShell）

页面 **Windows** 标签使用 `curl.exe` 与 PowerShell 行续符 `` ` ``，例如：

```powershell
curl.exe -s -X POST "https://your-store.myshopify.com/admin/oauth/access_token" `
  -H "Content-Type: application/json" `
  -d "{\"client_id\":\"...\",\"client_secret\":\"...\",\"code\":\"...\"}"
```

### 成功响应示例

```json
{
  "access_token": "shpat_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx",
  "scope": "read_discounts,read_products"
}
```

非永久 Online Token 响应 additionally 包含 `expires_in`、`associated_user` 等字段；非永久 Offline Token 响应 additionally 包含 `expires_in`、`refresh_token`、`refresh_token_expires_in`。

> **授权码 `code` 只能使用一次**，且有时效。复制 CURL 后请尽快在终端执行；若失败需重新点击生成按钮走一遍授权。

## GitHub Pages 部署

本仓库典型部署地址：

```
https://<username>.github.io/<repo-name>/token-generator-not-embedded
```

步骤概要：

1. 将 `token-generator-not-embedded/` 目录推送到 GitHub 仓库。
2. 仓库 **Settings → Pages** 启用 GitHub Pages（分支 / 目录按你的仓库结构选择）。
3. Partner Dashboard 中 App URL、Redirect URL 均设为上述 HTTPS 地址（**无尾部 `/`**）。
4. Release App 版本后，在店铺 Admin → Apps 中打开测试。

页面会根据当前路径自动解析静态资源与 Redirect URI，无需硬编码域名。

## 本地预览

在项目根目录或本目录启动静态服务：

```bash
npx serve .
```

浏览器访问：`http://localhost:3000/token-generator-not-embedded/`（端口以实际为准）。

> 本地 `http://` 无法用于 Shopify OAuth 回调；正式换取 Token 须使用已在 Partner Dashboard 登记的 **HTTPS** 地址。

## 文件结构

```
token-generator-not-embedded/
├── index.html    # 页面、表单、CURL 面板
├── app.js        # OAuth 跳转、state/HMAC 校验、CURL 生成
├── styles.css    # 样式
└── README.md     # 本说明
```

## Embedded = false 行为说明

| 行为 | 说明 |
|------|------|
| 从 Admin → Apps 打开 | 浏览器**新标签页**打开 App URL |
| 地址栏 | 应为 `github.io` 或你的自定义域名 |
| iframe | 不应出现在 Admin iframe 内；若仍在 iframe，请确认 Partner Dashboard 已关闭 Embed |
| OAuth 发起 | 标准 GET 表单跳转至 `{shop}.myshopify.com/admin/oauth/authorize` |

## 安全提示

- **CURL 命令包含 Client Secret**：请勿分享截图、录屏或提交到公开 Issue；执行完毕后可关闭终端历史中的敏感行。
- **Client Secret 在浏览器中使用**：Secret 保存在本次会话的 `sessionStorage`（或内存回退），仅用于生成 CURL，不会发送到本仓库服务器。
- 该方式适合**内部工具 / 开发环境**。若页面对公网开放，建议改为：前端只发起授权，由**后端**保管 Secret 并完成 `code` 交换。
- 生成的 Access Token 具有对应 Scope 的完整权限，请妥善保管，勿提交到公开仓库。

## 常见问题

**Failed to fetch / CORS**  
预期行为。请使用页面展示的 CURL 在终端换取 Token，不要在浏览器内直接请求 Token 接口。

**redirect_uri is not whitelisted**  
Partner Dashboard 中 Redirect URL 须与页面顶部显示**逐字一致**；最常见错误是多了尾部 `/`。修改后需 **Release** 新版本。

**OAuth state 校验失败**  
授权完成前刷新了页面、关闭了标签页，或浏览器禁用了 `sessionStorage`。请在同一标签页内完成授权，或重新点击生成按钮。

**HMAC 校验失败**  
多为 Secret 填写错误，或回调 URL 参数被中间层改写。请核对 Secret 与部署地址。

**授权码已失效 / invalid_grant**  
`code` 只能使用一次。CURL 执行失败后需重新发起 OAuth 授权，不要使用旧的 CURL。

**Scope 错误**  
授权 URL 中的 scopes 必须是 App 已申请权限的子集，拼写与逗号分隔格式须与 Partner Dashboard 一致。

**从 Admin 打开后地址仍是 admin.shopify.com**  
Embedded 可能仍为开启，或浏览器缓存了旧版本。请确认 Versions 中 `embedded: false` 并已 Release，然后重新打开 App。

**页面在 Admin iframe 内**  
与 Embedded = false 不符。关闭 Partner Dashboard 中的 Embed，卸载后重新安装 App。

**CURL 在 Windows 报错**  
请使用页面 **Windows** 标签下的命令（`curl.exe` + PowerShell 续行符）。若系统无 curl，可安装 [Windows curl](https://curl.se/windows/) 或使用 Git Bash 下的 **macOS / Linux** 命令格式。
