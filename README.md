# Shopify External App Release

可在 [GitHub Pages](https://pages.github.com/) 等静态站点托管的 Shopify 工具集合。

## 工具

### [Shopify Admin API Token 生成器](./token-generator-not-embedded/)

纯静态 OAuth 工具（**Embedded = false**），用于为 Shopify App 生成 Admin API Access Token。

- OAuth 授权在浏览器完成
- 换取 Token 需在本地终端执行 CURL（无后端，规避 CORS）
- 支持 Offline / Online Token
- Secret 在 sessionStorage 中加密存储

| 链接 | 说明 |
|------|------|
| [打开工具](./token-generator-not-embedded/) | GitHub Pages 入口（相对路径） |
| [使用说明](./token-generator-not-embedded/README.md) | 配置、部署与常见问题 |
| [仓库主页](./index.html) | 简要导航页 |

## GitHub Pages 部署

将本仓库启用 Pages 后，典型访问地址为：

```
https://<username>.github.io/<repo-name>/
```

工具页路径：

```
https://<username>.github.io/<repo-name>/token-generator-not-embedded/
```

Partner Dashboard 中 **App URL** 与 **Redirect URL** 须指向上述工具页（HTTPS、无尾部 `/`）。详见 [token-generator-not-embedded/README.md](./token-generator-not-embedded/README.md)。

## 本地开发与构建

本项目使用 [Vite](https://vite.dev/) 构建。构建产物输出到 `dist/`，可推送到独立仓库（如 `shopify-external-app-build`）作为 GitHub Pages 站点。

```bash
npm install
npm run dev      # 本地开发预览
npm run build    # 压缩 + 混淆，生成 dist/
npm run preview  # 预览 dist/ 构建结果
```

构建说明：

- **多页入口**：根目录 `index.html` 与 `token-generator-not-embedded/index.html`
- **JS**：Terser 压缩 + `javascript-obfuscator` 混淆（仅 token-generator 相关脚本）
- **CSS / HTML**：Vite 默认压缩
- **静态资源**：`images/` 等由 Vite 自动处理；README 复制到 `dist/`
- **base**：`./` 相对路径，适配 GitHub Pages 子路径部署

### 部署到 shopify-external-app-build

```bash
npm run build
# 将 dist/ 内容推送到 shopify-external-app-build 仓库根目录或 gh-pages 分支
```
