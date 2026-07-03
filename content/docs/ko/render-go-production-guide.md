# Render에서의 Go: 프로덕션 가이드

이 가이드는 Render에서 프로덕션 Go 웹 서비스를 실행하는 전체 라이프사이클을 안내합니다 — 데이터베이스 연결, 그레이스풀 셧다운 처리, 빠른 배포 유지까지. [기본 Go 설정](render-go.md)을 이미 완료했다고 가정합니다.

## Render에 맞는 서비스 구조화

Render는 서비스를 상태 비저장 프로세스로 실행합니다. 처음에 몇 가지 구조적 선택을 하면 나머지가 훨씬 쉬워집니다:

- **환경 변수에서 모든 구성을 읽습니다.** Go의 `os.Getenv`는 예상대로 작동합니다. 필요한 변수를 검증하고 누락된 것이 있으면 빠르게 실패하는 시작 헬퍼를 사용합니다. 이렇게 하면 런타임이 아닌 배포 시에 잘못된 구성을 발견할 수 있습니다.
- **`PORT`에서 수신합니다.** Render는 모든 웹 서비스에 이 변수를 설정합니다. 포트를 하드코딩하면 헬스 체크가 실패하고 배포가 롤백됩니다.
- **`SIGTERM`에서 깨끗하게 종료합니다.** Render는 컨테이너를 중지하기 전에 `SIGTERM`을 보냅니다. 이를 처리하지 않는 서비스는 진행 중인 요청을 중단시킬 수 있습니다.

## 그레이스풀 셧다운

Render가 프로세스를 중지하기 전에 요청을 드레인할 수 있도록 `http.Server`에 셧다운 핸들러를 래핑합니다:

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
    // 다른 라우트를 여기에 등록합니다

    srv := &http.Server{
        Addr:    ":" + port,
        Handler: mux,
    }

    // 고루틴에서 서버를 시작합니다.
    go func() {
        log.Printf("listening on :%s", port)
        if err := srv.ListenAndServe(); err != nil && err != http.ErrServerClosed {
            log.Fatalf("server error: %v", err)
        }
    }()

    // SIGINT 또는 SIGTERM이 수신될 때까지 차단합니다.
    quit := make(chan os.Signal, 1)
    signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
    <-quit

    // 진행 중인 요청이 완료될 때까지 최대 30초를 줍니다.
    ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()
    if err := srv.Shutdown(ctx); err != nil {
        log.Fatalf("shutdown error: %v", err)
    }
    log.Println("server stopped")
}
```

30초 창은 Render의 기본 드레인 타임아웃과 일치합니다. 서비스가 처리하는 가장 긴 요청에 맞게 조정하세요.

## 구조화된 로깅

일반 `log.Printf` 출력은 Render에 의해 캡처되어 대시보드 로그 뷰어에서 사용 가능합니다. 필터링하기 쉬운 구조화된 로그를 위해 `log/slog`(Go 1.21+)와 같은 JSON 로거로 전환합니다:

```go
import "log/slog"

logger := slog.New(slog.NewJSONHandler(os.Stdout, nil))
logger.Info("request completed",
    "method", r.Method,
    "path", r.URL.Path,
    "status", statusCode,
    "duration_ms", duration.Milliseconds(),
)
```

Render는 모든 stdout 및 stderr 출력을 스트리밍합니다. 파일에 로그를 작성하지 마세요 — 대시보드에 표시되지 않으며 컨테이너가 재시작될 때 손실됩니다.

## Render PostgreSQL 데이터베이스 연결

Render PostgreSQL 데이터베이스를 서비스에 연결하면 Render가 자동으로 `DATABASE_URL` 환경 변수를 설정합니다. 시작 시 이를 읽습니다:

```go
import (
    "database/sql"
    "os"

    _ "github.com/lib/pq"
)

func openDB() (*sql.DB, error) {
    dsn := os.Getenv("DATABASE_URL")
    if dsn == "" {
        return nil, fmt.Errorf("DATABASE_URL is not set")
    }
    db, err := sql.Open("postgres", dsn)
    if err != nil {
        return nil, err
    }
    // 보수적인 풀 한도를 설정합니다. Render의 무료 티어 Postgres는 연결 수를 제한합니다.
    db.SetMaxOpenConns(10)
    db.SetMaxIdleConns(5)
    db.SetConnMaxLifetime(5 * time.Minute)
    return db, nil
}
```

시작 시 `db.PingContext`를 호출하여 트래픽을 수락하기 전에 연결을 확인합니다. 핑이 실패하면 0이 아닌 코드로 종료하여 Render가 비정상 인스턴스에 요청을 라우팅하는 대신 배포를 실패로 표시하도록 합니다.

> **내부 vs. 외부 연결 문자열.** Render는 외부 URL(어디서나 접근 가능)과 내부 URL(동일 지역의 서비스 간에만 라우팅 가능)을 모두 제공합니다. 이그레스 비용을 피하고 지연 시간을 줄이기 위해 서비스 간 연결에 내부 URL을 사용합니다. 데이터베이스와 서비스가 동일한 환경을 공유할 때 내부 URL은 `DATABASE_URL`로 사용 가능합니다.

## 데이터베이스 마이그레이션 실행

빌드 단계의 일부로 또는 별도의 [사전 배포 작업](https://render.com/docs/deploys#pre-deploy-commands)으로 마이그레이션을 실행합니다. 사전 배포 작업을 사용하면 마이그레이션이 배포와 원자적으로 유지됩니다:

1. Render 대시보드에서 서비스를 열고 **설정 → 배포**로 이동합니다.
2. **사전 배포 명령**을 마이그레이션 실행기로 설정합니다. 예:
   ```
   ./migrate -database "$DATABASE_URL" -path ./migrations up
   ```
3. 마이그레이션 명령이 0이 아닌 값으로 종료되면 Render는 배포를 취소하고 이전 버전이 계속 트래픽을 처리합니다.

## 배포 간 의존성 캐싱

Render는 빌드 간에 Go 모듈 다운로드 캐시를 캐시합니다. 이를 활용하려면 `go.sum` 파일이 커밋되어 있고 최신 상태인지 확인하세요. 캐시되지 않은 모듈을 다운로드해야 하는 빌드는 느립니다. 완전히 해결된 `go.sum`은 캐시를 따뜻하게 유지합니다.

vendor 디렉터리로 의존성을 관리하는 경우 빌드 명령을 다음과 같이 설정합니다:

```bash
go build -mod=vendor -o server .
```

벤더링된 빌드는 완전히 재현 가능하며 빌드 시 네트워크 액세스가 필요하지 않습니다.

## 헬스 체크 엔드포인트

**설정 → 헬스 & 알림 → 헬스 체크 경로**에서 전용 헬스 체크 경로를 설정합니다. 중요한 의존성을 확인하는 가벼운 엔드포인트는 Render에 신뢰할 수 있는 신호를 제공합니다:

```go
mux.HandleFunc("/healthz", func(w http.ResponseWriter, r *http.Request) {
    if err := db.PingContext(r.Context()); err != nil {
        http.Error(w, "database unreachable", http.StatusServiceUnavailable)
        return
    }
    w.WriteHeader(http.StatusOK)
})
```

헬스 체크를 빠르게 유지하세요 — 짧은 간격으로 실행되며 느린 응답은 타임아웃으로 계산됩니다. 핸들러 내부에서 비용이 많이 드는 쿼리를 피하세요.

## 무중단 배포

Render는 유료 플랜에서 기본적으로 롤링 배포를 수행합니다: 새 버전이 시작되고 헬스 체크를 통과한 후에 이전 버전이 중지됩니다. 이를 최대한 활용하려면:

- 백그라운드 워커나 캐시가 준비되기 전에도 프로세스가 `PORT`에 바인드되는 즉시 헬스 체크가 올바르게 응답하도록 합니다.
- 데이터베이스 스키마 변경을 이전 버전의 코드와 하위 호환 가능하게 만듭니다. 코드 변경 전에 마이그레이션을 먼저 배포합니다.
- 위의 그레이스풀 셧다운 패턴을 사용하여 새 인스턴스가 시작되는 동안 이전 인스턴스가 요청을 깨끗하게 드레인하도록 합니다.

## 수평 확장

**설정 → 확장**에서 서비스에 인스턴스를 추가합니다. Go의 표준 라이브러리 HTTP 서버는 연결당 고루틴 방식이며 코어 전체에서 잘 확장됩니다. 인스턴스를 추가하기 전에 확인할 사항:

- **세션 상태는 프로세스 외부에 있어야 합니다.** Render Redis 인스턴스나 외부 스토어를 사용합니다. 인메모리 세션 상태는 인스턴스 간에 공유되지 않습니다.
- **예약된 작업은 멱등성이 있어야 합니다.** 백그라운드 작업을 위해 서비스 내부에 `time.Ticker`를 사용하는 경우 여러 인스턴스가 동시에 ticker를 실행합니다. 시간에 민감한 예약 작업은 전용 [크론 잡](https://render.com/docs/cronjobs) 서비스로 이동합니다.
- **데이터베이스 연결 한도.** 각 추가 인스턴스는 자체 연결 풀을 엽니다. Postgres 플랜의 연결 한도 내에서 유지하거나 연결 풀러를 추가합니다.

## 프로덕션 배포 체크리스트

Render에서 Go 서비스를 프로덕션으로 승격하기 전에 다음을 확인합니다:

- [ ] 앱이 환경에서 `PORT`를 읽고 해당 포트에서 수신합니다.
- [ ] `SIGTERM`이 포착되고 서버가 그레이스풀하게 셧다운됩니다.
- [ ] 모든 필수 환경 변수가 시작 시 검증됩니다.
- [ ] `DATABASE_URL`이 동일 지역 서비스에 내부 연결 문자열을 사용합니다.
- [ ] 데이터베이스 마이그레이션이 사전 배포 명령으로 또는 코드 배포 전의 제어된 단계로 실행됩니다.
- [ ] `/healthz`(또는 동등한) 엔드포인트가 등록되어 대시보드에 설정되어 있습니다.
- [ ] 로그가 구조화된 형식으로 stdout/stderr로 출력됩니다.
- [ ] `go.sum`이 커밋되어 있고 최신 상태입니다.
- [ ] 비밀이 하드코딩되어 있지 않습니다. 모든 자격 증명은 환경 변수 또는 시크릿 파일로 설정됩니다.
