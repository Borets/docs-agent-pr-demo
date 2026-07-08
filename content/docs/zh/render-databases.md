# PostgreSQL 数据库

Render 提供托管的 PostgreSQL 数据库，可在几秒内完成部署，并可直接连接到你的其他 Render 服务。本页介绍如何创建数据库、将服务连接到数据库以及在生产环境中管理数据库。

## 创建数据库

1. 在 Render 控制台中，选择 **新建 → PostgreSQL**。
2. 为数据库命名并选择区域。请使用与连接它的服务相同的区域，以避免跨区域延迟。
3. 选择一个计划。免费版数据库适用于开发和测试；它们将在 90 天后过期。
4. 点击 **创建数据库**。

Render 将完成数据库的配置，并在 **信息** 选项卡上显示连接详情。

## 连接字符串

每个数据库提供两个连接字符串：

| 字符串 | 使用场景 |
|---|---|
| **内部数据库 URL** | 在同一 Render 区域运行的服务。无出口费用，延迟更低。|
| **外部数据库 URL** | 来自 Render 外部的连接——你的本地机器、CI runner 或第三方服务。|

在 Render 内部的服务间连接始终使用内部 URL。内部 URL 无法从公网访问。

通过控制台中的 **连接** 按钮将数据库链接到服务时，Render 会自动将 `DATABASE_URL`（设置为内部 URL）作为环境变量注入到该服务中。

## 从服务连接

链接数据库后，你的服务可从 `DATABASE_URL` 读取连接字符串，无需硬编码凭据：

```js
// Node.js 示例，使用 pg 包
const { Pool } = require('pg');
const pool = new Pool({ connectionString: process.env.DATABASE_URL });
```

```python
# Python 示例，使用 psycopg2
import os, psycopg2
conn = psycopg2.connect(os.environ["DATABASE_URL"])
```

```go
// Go 示例，使用 database/sql
import (
    "database/sql"
    "os"
    _ "github.com/lib/pq"
)

db, err := sql.Open("postgres", os.Getenv("DATABASE_URL"))
```

在启动时验证连接，如果 `DATABASE_URL` 缺失或无法建立连接，则快速失败。没有正常数据库连接就启动的服务在实际流量下很可能会失败。

## SSL

Render PostgreSQL 要求所有外部连接使用 SSL。内部 URL 不需要额外的 SSL 配置。如果从 Render 外部连接，请在连接字符串后附加 `?sslmode=require`，或配置客户端库强制使用 SSL。

## 运行迁移

在新代码开始提供服务之前运行迁移。推荐方式是使用**预部署命令**：

1. 在控制台打开你的服务，进入 **设置 → 构建与部署**。
2. 将 **预部署命令** 设置为你的迁移运行器。示例：

   ```bash
   # Prisma
   npx prisma migrate deploy

   # golang-migrate
   ./migrate -database "$DATABASE_URL" -path ./migrations up

   # Alembic（Python）
   alembic upgrade head
   ```

3. 如果迁移失败，Render 将取消部署，你服务的前一版本继续使用未更改的 schema 运行。

尽可能将迁移设计为与代码前一版本向后兼容，这能确保零停机部署的安全性。

## 连接池

每个服务实例都会打开自己的连接池。在连接数限制较低的计划上，多个已扩展的实例可能会耗尽数据库的 `max_connections`。为避免这种情况：

- 在应用中设置保守的连接池大小。
- 将 [PgBouncer](https://www.pgbouncer.org/) 或类似的连接池工具作为独立服务部署。
- 升级到连接数限制更高的 Render PostgreSQL 计划。

## 备份

Render 会自动对付费计划的数据库进行每日备份。你也可以随时在数据库的 **备份** 选项卡中触发手动备份。备份根据你的计划的保留策略进行保留。

免费版数据库不包含自动备份。

## 直接访问数据库

要从本地机器连接到数据库或运行一次性查询，请使用外部连接字符串配合 `psql` 或任意 PostgreSQL 客户端：

```bash
psql "$EXTERNAL_DATABASE_URL"
```

你也可以通过数据库控制台页面中的 **Shell** 选项卡打开交互式 shell。

## 升级 PostgreSQL

Render 支持 PostgreSQL 主版本升级。升级前：

1. 在 **备份** 选项卡中进行手动备份。
2. 先在非生产数据库上测试升级。
3. 查阅 PostgreSQL 版本说明，了解影响你应用的重大变更。

升级需要短暂的停机时间。请在低流量窗口期安排升级。

## 常见问题

**出现 `too many connections` 错误**
你的应用打开的连接数超过了数据库计划允许的数量。减小每个实例的连接池大小，添加连接池工具，或升级你的数据库计划。

**本地连接时提示 `SSL connection is required`**
在外部连接字符串中添加 `?sslmode=require`，或在 shell 中设置 `PGSSLMODE=require`。

**预部署时迁移失败**
检查预部署环境中 `DATABASE_URL` 是否可用。通过控制台将数据库链接到服务后，该变量会自动注入。

**免费版数据库已过期**
免费版 PostgreSQL 数据库在 90 天后过期。请升级到付费计划，或创建新数据库并从备份中恢复数据。
