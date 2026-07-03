# Node.js on Render

Render 原生支持 Node.js。你可以将任何 Node.js 应用部署为 Web 服务、后台 Worker 或 Cron Job——Render 会自动检测你的项目、安装依赖并运行启动命令。

## 要求

- 仓库根目录（或你指定为根目录的子目录）中存在 `package.json` 文件。
- 推荐使用 Node.js 18 或更高版本。Render 默认使用最新的 LTS 版本；可通过 `.node-version` 或 `.nvmrc` 文件，或通过 `NODE_VERSION` 环境变量来固定特定版本。

## 快速开始

1. 将你的 Node.js 仓库推送到 GitHub 或 GitLab。
2. 在 Render 控制台中，选择 **新建 → Web 服务** 并连接你的仓库。
3. 确认检测到的构建和启动命令（见下文），或根据需要修改。
4. 点击 **创建 Web 服务**。

每次推送到已配置的分支时，Render 会自动构建并部署你的服务。

## 构建与启动命令

Render 会自动检测 Node.js 项目并设置合理的默认值。

| 设置 | 默认值 |
|---|---|
| **构建命令** | `npm install` |
| **启动命令** | `node index.js` |

如果你的项目使用不同的入口点或框架 CLI，请修改启动命令。常见示例：

| 框架 / 工具 | 启动命令 |
|---|---|
| Express（`server.js`）| `node server.js` |
| Fastify | `node app.js` |
| NestJS | `node dist/main.js` |
| Next.js（服务端）| `npm run start` |

你可以随时在 **设置 → 构建与部署** 中修改这两个命令。

## 固定 Node 版本

在仓库根目录创建 `.node-version` 或 `.nvmrc` 文件：

```
# .node-version
20.14.0
```

也可以在服务上设置 `NODE_VERSION` 环境变量。基于文件的方式更推荐，因为它将版本信息保存在版本控制中，并在所有环境中一致生效。

## 监听 PORT

Render 会为每个 Web 服务注入 `PORT` 环境变量。你的应用必须监听该端口：

```js
const express = require('express');
const app = express();

const port = process.env.PORT || 3000;
app.listen(port, () => {
  console.log(`服务器正在监听端口 ${port}`);
});
```

硬编码端口号将导致健康检查失败，并触发部署回滚。

## 环境变量

在服务的 **环境** 选项卡中设置环境变量，或通过 Render 环境组引用它们以跨服务共享值。在代码中通过 `process.env` 访问：

```js
const db = new Client({ connectionString: process.env.DATABASE_URL });
const apiKey = process.env.THIRD_PARTY_API_KEY;
```

永远不要将 secret 提交到仓库。所有凭据和 API 密钥都应在控制台中设置为环境变量。

## 健康检查

Render 向你配置的健康检查路径（默认：`/`）发送 HTTP GET 请求，以验证服务是否正在运行。应用准备好后立即响应 `2xx` 状态码：

```js
app.get('/healthz', (req, res) => {
  res.sendStatus(200);
});
```

在 **设置 → 健康与告警 → 健康检查路径** 中配置路径。一个检查关键依赖项（数据库连接、必需配置）的专用端点比根路径能提供更可靠的信号。

## TypeScript

对于 TypeScript 项目，在构建步骤中编译为 JavaScript，然后运行编译后的输出：

| 设置 | 示例值 |
|---|---|
| **构建命令** | `npm install && npm run build` |
| **启动命令** | `node dist/index.js` |

确保 `tsc` 或你的构建工具（esbuild、tsup 等）作为开发依赖列在 `package.json` 中，以便在构建期间可用。

## 静态文件

对于高流量站点，请避免从 Node.js Web 服务提供静态文件。建议将资产托管在 Render 静态站点或 CDN 上，让 Node.js 服务只处理 API 请求。如果确实需要从 Node.js 提供静态文件，请使用缓存中间件并设置适当的 `Cache-Control` 响应头。

## 使用 yarn 或 pnpm

要使用 `yarn` 或 `pnpm` 替代 `npm`，请更新构建命令：

```bash
# yarn
yarn install --frozen-lockfile

# pnpm
npm install -g pnpm && pnpm install --frozen-lockfile
```

提交你的 lock 文件（`yarn.lock` 或 `pnpm-lock.yaml`）以保持安装的可复现性。

## 单仓库（Monorepo）设置

如果你的服务是单仓库中的多个服务之一，请将服务的 **根目录** 设置为包含相关 `package.json` 的子目录。Render 将相对于该目录运行所有构建和启动命令。

如果你的单仓库使用工作区工具（Turborepo、Nx、Lerna），请将构建命令限定在正在部署的服务范围内：

```bash
# Turborepo：仅构建 'api' 包
npx turbo build --filter=api
```

## 常见问题

**构建失败，提示 `Cannot find module`**
该模块未列在 `package.json` 中，或 lock 文件不同步。在本地运行 `npm install`，提交更新后的 lock 文件，然后重新部署。

**部署后服务立即退出**
检查 `PORT` 环境变量是否被读取，以及服务器是否在打印任何就绪消息之前绑定到该端口。不绑定端口就退出的进程会被标记为失败。

**`npm run build` 不被识别**
`package.json` 中缺少 `build` 脚本。添加该脚本，或在控制台中将构建命令更新为你想运行的确切命令。

**TypeScript 在生产环境编译出错，但本地正常**
本地构建可能使用了不同版本的 Node 或 TypeScript。在 Render 服务上固定 `NODE_VERSION` 以匹配本地环境，并将 TypeScript 添加为开发依赖，确保 CI 和生产构建使用相同的版本。
