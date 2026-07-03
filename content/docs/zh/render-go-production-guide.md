# Go on Render：生产指南

本指南介绍在 Render 上运行 Go Web 服务的完整生命周期——从连接数据库、处理优雅关闭，到保持部署高效。阅读本文前，请先完成 [Go 基础配置](render-go.md)。

## 为 Render 设计服务结构

Render 以无状态进程的形式运行你的服务。提前做好以下几项结构决策，可以让后续一切更加顺畅：

- **从环境变量读取所有配置。** Go 的 `os.Getenv` 按预期工作。使用启动辅助函数验证必需的变量，在变量缺失时快速失败；这样可以在部署时而非运行时暴露配置错误。
- **监听 `PORT`。** Render 为每个 Web 服务设置此变量。硬编码端口号会导致健康检查失败并触发部署回滚。
- **在 `SIGTERM` 时干净退出。** Render 在停止容器之前会发送 `SIGTERM`。不处理该信号的服务可能会丢弃正在进行的请求。

## 优雅关闭

用关闭处理器包装你的 `http.Server`，以便 Render 在停止进程前能够排空请求：

```go
package main

import (
    "context"
    "log"
    "net/http"
    "os"
    "os/signal"
    "syscall"
    "time"
)

func main() {
    port := os.Getenv("PORT")
    if port == "" {
        port = "8080"
    }

    mux := http.NewServeMux()
    mux.HandleFunc("/healthz", func(w http.ResponseWriter, r *http.Request) {
        w.WriteHeader(http.StatusOK)
    })
    // 在此注册你的其他路由

    srv := &http.Server{
        Addr:    ":" + port,
        Handler: mux,
    }

    // 在 goroutine 中启动服务器。
    go func() {
        log.Printf("监听端口 :%s", port)
        if err := srv.ListenAndServe(); err != nil && err != http.ErrServerClosed {
            log.Fatalf("服务器错误: %v", err)
        }
    }()

    // 阻塞直到收到 SIGINT 或 SIGTERM。
    quit := make(chan os.Signal, 1)
    signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
    <-quit

    // 给正在进行的请求最多 30 秒完成。
    ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()
    if err := srv.Shutdown(ctx); err != nil {
        log.Fatalf("关闭错误: %v", err)
    }
    log.Println("服务器已停止")
}
```

30 秒窗口与 Render 的默认排空超时时间一致。根据你的服务处理的最长请求进行调整。

## 结构化日志

`log.Printf` 的输出会被 Render 捕获，并可在控制台日志查看器中查看。若要使用更易于过滤的结构化日志，请切换到 JSON 日志记录器，例如 `log/slog`（Go 1.21+）：

```go
import "log/slog"

logger := slog.New(slog.NewJSONHandler(os.Stdout, nil))
logger.Info("请求已完成",
    "method", r.Method,
    "path", r.URL.Path,
    "status", statusCode,
    "duration_ms", duration.Milliseconds(),
)
```

Render 会流式传输所有 stdout 和 stderr 输出。请勿将日志写入文件——这些日志不会出现在控制台中，并且会在容器重启时丢失。

## 连接 Render PostgreSQL 数据库

将 Render PostgreSQL 数据库附加到服务时，Render 会自动设置 `DATABASE_URL` 环境变量。在启动时读取它：

```go
import (
    "database/sql"
    "os"

    _ "github.com/lib/pq"
)

func openDB() (*sql.DB, error) {
    dsn := os.Getenv("DATABASE_URL")
    if dsn == "" {
        return nil, fmt.Errorf("DATABASE_URL 未设置")
    }
    db, err := sql.Open("postgres", dsn)
    if err != nil {
        return nil, err
    }
    // 设置保守的连接池限制；Render 免费版 Postgres 对连接数有上限。
    db.SetMaxOpenConns(10)
    db.SetMaxIdleConns(5)
    db.SetConnMaxLifetime(5 * time.Minute)
    return db, nil
}
```

在启动时调用 `db.PingContext` 以在接受流量之前验证连接。如果 ping 失败，以非零代码退出，这样 Render 会将部署标记为失败，而不是将请求路由到不健康的实例。

> **内部连接字符串与外部连接字符串。** Render 提供外部 URL（可从任何地方访问）和内部 URL（仅在同一区域的服务之间可路由）。使用内部 URL 进行服务间连接，以避免出口费用并降低延迟。当数据库和服务共享同一环境时，内部 URL 可作为 `DATABASE_URL` 使用。

## 运行数据库迁移

在构建步骤中运行迁移，或作为单独的[预部署任务](https://render.com/docs/deploys#pre-deploy-commands)运行。使用预部署任务可将迁移与部署保持原子性：

1. 在 Render 控制台中，打开你的服务并进入 **设置 → 部署**。
2. 将 **预部署命令** 设置为你的迁移运行器，例如：
   ```
   ./migrate -database "$DATABASE_URL" -path ./migrations up
   ```
3. 如果迁移命令以非零代码退出，Render 会取消部署，前一版本继续提供服务。

## 跨部署缓存依赖

Render 在构建之间缓存 Go 模块下载缓存。要利用这一特性，请确保你的 `go.sum` 文件已提交且是最新的。需要下载未缓存模块的构建会更慢；完整解析的 `go.sum` 可保持缓存热态。

如果你使用 vendor 目录管理依赖，请将构建命令设置为：

```bash
go build -mod=vendor -o server .
```

使用 vendor 的构建完全可复现，且构建时不需要网络访问。

## 健康检查端点

在 **设置 → 健康与告警 → 健康检查路径** 中配置专用的健康检查路径。一个检查关键依赖项的轻量级端点能为 Render 提供可靠的信号：

```go
mux.HandleFunc("/healthz", func(w http.ResponseWriter, r *http.Request) {
    if err := db.PingContext(r.Context()); err != nil {
        http.Error(w, "数据库不可达", http.StatusServiceUnavailable)
        return
    }
    w.WriteHeader(http.StatusOK)
})
```

保持健康检查快速响应——它们以较短的间隔运行，慢速响应会计入超时时间。避免在处理器中执行开销大的查询。

## 零停机部署

Render 默认对付费计划执行滚动部署：新版本启动并通过健康检查后，旧版本才会停止。要充分利用这一特性：

- 确保在进程绑定到 `PORT` 后健康检查能立即正确响应，即使后台 Worker 或缓存尚未预热。
- 使数据库 schema 更改与你代码的前一版本向后兼容。先部署迁移，再部署代码更改。
- 使用上述优雅关闭模式，以便旧实例在新实例启动时能干净地排空请求。

## 水平扩展

从 **设置 → 扩展** 为你的服务添加实例。Go 标准库的 HTTP 服务器采用每连接一个 goroutine 的模型，能很好地跨核心扩展。在添加实例前，请验证以下几点：

- **会话状态必须存储在进程外。** 使用 Render Redis 实例或外部存储。内存中的会话状态不会在实例间共享。
- **定时任务必须是幂等的。** 如果你在服务内使用 `time.Ticker` 执行后台任务，多个实例会并发运行该 ticker。将时间敏感的定时工作移至专用的 [Cron Job](https://render.com/docs/cronjobs) 服务。
- **数据库连接限制。** 每个额外实例都会打开自己的连接池。请保持在你 Postgres 计划的连接限制内，或添加连接池工具。

## 生产部署检查清单

在将 Go 服务推进到 Render 生产环境之前，请确认以下各项：

- [ ] 应用从环境中读取 `PORT` 并监听它。
- [ ] 已捕获 `SIGTERM`，服务器能优雅关闭。
- [ ] 所有必需的环境变量在启动时均已验证。
- [ ] `DATABASE_URL` 对同区域服务使用内部连接字符串。
- [ ] 数据库迁移作为预部署命令或在代码部署前的受控步骤运行。
- [ ] `/healthz`（或等效端点）已注册并在控制台中配置。
- [ ] 日志以结构化格式输出到 stdout/stderr。
- [ ] `go.sum` 已提交且是最新的。
- [ ] 没有硬编码的 secret；所有凭据通过环境变量或 secret 文件设置。
