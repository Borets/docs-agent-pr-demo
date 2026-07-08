# 静态站点

Render 可以直接从你的 Git 仓库构建和托管静态站点。静态站点在所有计划上均免费，并在每次推送时自动部署。

## 什么是静态站点

静态站点是指构建输出为包含 HTML、CSS、JavaScript 及资产文件目录的任何项目。常见的框架包括：

- Next.js（静态导出）
- Gatsby
- Vite / Create React App
- Hugo、Jekyll、Eleventy
- Astro（静态模式）
- 纯 HTML

如果你的项目需要在请求时通过服务器渲染响应，请将其作为 **Web 服务** 部署。

## 创建静态站点

1. 将你的项目推送到 GitHub 或 GitLab。
2. 在 Render 控制台中，选择 **新建 → 静态站点** 并连接你的仓库。
3. 设置 **构建命令** 和 **发布目录**（见下方默认值）。
4. 点击 **创建静态站点**。

Render 构建站点并将其发布到 `*.onrender.com` URL。之后每次推送到已配置的分支都会自动触发新的构建和部署。

## 构建命令与发布目录

Render 需要两项信息来构建和托管你的站点：

| 设置 | 说明 |
|---|---|
| **构建命令** | 生成静态输出的命令。|
| **发布目录** | 构建完成后 Render 托管的目录。|

常见框架的默认值：

| 框架 | 构建命令 | 发布目录 |
|---|---|---|
| Create React App | `npm run build` | `build` |
| Vite | `npm run build` | `dist` |
| Next.js（静态导出）| `npm run build` | `out` |
| Gatsby | `gatsby build` | `public` |
| Hugo | `hugo` | `public` |
| Eleventy | `npx @11ty/eleventy` | `_site` |

你可以随时在 **设置 → 构建与部署** 中修改这两项设置。

## 自定义域名

在 **设置 → 自定义域名** 选项卡中添加自定义域名：

1. 输入你的域名并点击 **保存**。
2. 在你的 DNS 提供商处添加一条指向你 `*.onrender.com` 地址的 `CNAME` 记录。
3. Render 自动配置并续期 TLS 证书。

裸域名（不带 `www` 的 `example.com`）需要 `ALIAS` 或 `ANAME` 记录。并非所有 DNS 提供商都支持这些记录类型；如果该选项不可用，请咨询你的提供商。

## 重定向与重写

对于单页应用（SPA），你需要将所有路径重写到 `index.html`，以便客户端路由能正常工作。在你的**发布目录**根目录中创建一个 `_redirects` 文件：

```
/* /index.html 200
```

你也可以定义明确的重定向：

```
/old-path /new-path 301
/blog/* /posts/:splat 302
```

规则从上到下依次评估，第一条匹配的规则生效。

## 响应头

在发布目录中使用 `_headers` 文件设置自定义 HTTP 响应头：

```
/*
  X-Frame-Options: DENY
  X-Content-Type-Options: nosniff
  Content-Security-Policy: default-src 'self'

/assets/*
  Cache-Control: public, max-age=31536000, immutable
```

## 环境变量

构建时环境变量在构建过程中可用。在 **设置 → 环境** 中设置它们。

请注意，静态站点中的环境变量**仅在构建时有效**，它们会被烘焙到构建产物中。不要将 secret 或凭据放入静态站点的环境变量中——这些值可能在编译输出中可见。

对于要求前缀的框架（例如 Vite 的 `VITE_` 或 Create React App 的 `REACT_APP_`），在设置变量名时请添加相应前缀。

## Pull Request 预览

在 **设置 → Pull Request 预览** 中启用 **Pull Request 预览**。Render 为每个开放的 Pull Request 构建并部署独立的预览站点，预览链接会直接发布到 PR 中。

当 PR 被关闭或合并时，预览站点将被销毁。

## 构建缓存

Render 在构建之间缓存 `node_modules` 及其他包管理器目录。大多数依赖安装直接从缓存中提供，因此构建只需运行你框架实际的编译步骤。

要清除缓存并强制完整重新安装，请从控制台选择 **手动部署 → 清除构建缓存并部署**。

## 常见问题

**站点能加载但客户端路由返回 404**
你的发布目录缺少包含 `/* /index.html 200` 重写规则的 `_redirects` 文件。添加该文件并重新部署。

**构建失败，提示 `command not found`**
构建命令引用了未安装的 CLI 工具。将其添加为 `package.json` 中的开发依赖，或在构建命令中显式安装它。

**环境变量在运行时不可用**
静态站点的环境变量仅在构建时注入。如果编译输出中缺少某个变量，请确认它在上次构建前已设置，然后重新部署以获取新值。

**自定义域名显示证书错误**
`CNAME` 记录传播后（通常在几分钟到一小时内），Render 会自动配置 TLS。如果一小时后错误仍然存在，请确认 DNS 记录指向控制台中显示的正确 `*.onrender.com` 地址。
