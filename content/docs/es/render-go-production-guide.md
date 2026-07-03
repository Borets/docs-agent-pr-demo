# Go en Render: Guía de producción

Esta guía recorre el ciclo de vida completo de ejecutar un servicio web Go en
producción en Render, desde la conexión con una base de datos hasta el manejo de
apagados elegantes y la optimización de los despliegues. Asume que ya has completado
la [configuración básica de Go](render-go.md).

## Estructura de tu servicio para Render

Render ejecuta tu servicio como un proceso sin estado. Algunas decisiones
estructurales desde el principio facilitan todo lo demás:

- **Lee toda la configuración desde variables de entorno.** El `os.Getenv` de Go
  funciona según lo esperado. Usa un helper de inicio para validar las variables
  requeridas y fallar rápido si alguna falta; esto expone la mala configuración
  en el momento del despliegue en lugar de en tiempo de ejecución.
- **Escucha en `PORT`.** Render establece esta variable para cada Servicio Web.
  Codificar un puerto de forma fija causa que fallen las comprobaciones de estado
  y que el despliegue se revierta.
- **Termina limpiamente con `SIGTERM`.** Render envía `SIGTERM` antes de detener
  un contenedor. Un servicio que no lo maneja puede perder solicitudes en curso.

## Apagado elegante

Envuelve tu `http.Server` con un manejador de apagado para que Render pueda
drenar las solicitudes antes de detener el proceso:

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
    // registra tus otras rutas aquí

    srv := &http.Server{
        Addr:    ":" + port,
        Handler: mux,
    }

    // Inicia el servidor en una goroutine.
    go func() {
        log.Printf("escuchando en :%s", port)
        if err := srv.ListenAndServe(); err != nil && err != http.ErrServerClosed {
            log.Fatalf("error del servidor: %v", err)
        }
    }()

    // Bloquea hasta recibir SIGINT o SIGTERM.
    quit := make(chan os.Signal, 1)
    signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
    <-quit

    // Da hasta 30 segundos a las solicitudes en curso para que se completen.
    ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()
    if err := srv.Shutdown(ctx); err != nil {
        log.Fatalf("error de apagado: %v", err)
    }
    log.Println("servidor detenido")
}
```

La ventana de 30 segundos coincide con el tiempo de espera de drenaje predeterminado
de Render. Ajústalo según las solicitudes más largas que maneje tu servicio.

## Registro estructurado

La salida de `log.Printf` es capturada por Render y está disponible en el visor
de registros del panel de control. Para registros estructurados más fáciles de
filtrar, cambia a un registrador JSON como `log/slog` (Go 1.21+):

```go
import "log/slog"

logger := slog.New(slog.NewJSONHandler(os.Stdout, nil))
logger.Info("solicitud completada",
    "method", r.Method,
    "path", r.URL.Path,
    "status", statusCode,
    "duration_ms", duration.Milliseconds(),
)
```

Render transmite toda la salida de stdout y stderr. No escribas registros en un
archivo — no aparecerán en el panel de control y se perderán cuando el contenedor
se reinicie.

## Conexión a una base de datos PostgreSQL de Render

Cuando adjuntas una base de datos PostgreSQL de Render a un servicio, Render
establece la variable de entorno `DATABASE_URL` automáticamente. Léela al inicio:

```go
import (
    "database/sql"
    "os"

    _ "github.com/lib/pq"
)

func openDB() (*sql.DB, error) {
    dsn := os.Getenv("DATABASE_URL")
    if dsn == "" {
        return nil, fmt.Errorf("DATABASE_URL no está configurada")
    }
    db, err := sql.Open("postgres", dsn)
    if err != nil {
        return nil, err
    }
    // Establece límites de grupo conservadores; el Postgres de nivel gratuito de Render limita las conexiones.
    db.SetMaxOpenConns(10)
    db.SetMaxIdleConns(5)
    db.SetConnMaxLifetime(5 * time.Minute)
    return db, nil
}
```

Llama a `db.PingContext` durante el inicio para verificar la conectividad antes
de aceptar tráfico. Si el ping falla, termina con un código distinto de cero para
que Render marque el despliegue como fallido en lugar de enrutar solicitudes a una
instancia no saludable.

> **Cadenas de conexión interna vs. externa.** Render proporciona tanto una URL
> externa (accesible desde cualquier lugar) como una URL interna (solo enrutable
> entre servicios en la misma región). Usa la URL interna para conexiones entre
> servicios para evitar cargos de salida y reducir la latencia. La URL interna
> está disponible como `DATABASE_URL` cuando la base de datos y el servicio
> comparten el mismo entorno.

## Ejecución de migraciones de base de datos

Ejecuta las migraciones como parte del paso de compilación o como un
[trabajo de Pre-Deploy](https://render.com/docs/deploys#pre-deploy-commands)
separado. Usar un trabajo de Pre-Deploy mantiene las migraciones atómicas con el
despliegue:

1. En el panel de control de Render, abre tu servicio y ve a
   **Configuración → Implementación**.
2. Establece el **Comando de Pre-Deploy** en tu ejecutor de migraciones, por ejemplo:
   ```
   ./migrate -database "$DATABASE_URL" -path ./migrations up
   ```
3. Si el comando de migración termina con un código distinto de cero, Render cancela
   el despliegue y la versión anterior continúa sirviendo tráfico.

## Almacenamiento en caché de dependencias entre despliegues

Render almacena en caché la caché de descarga de módulos Go entre compilaciones.
Para aprovechar esto, asegúrate de que tu archivo `go.sum` esté confirmado y
actualizado. Las compilaciones que requieren descargar módulos no almacenados en
caché son más lentas; un `go.sum` completamente resuelto mantiene la caché activa.

Si gestionas dependencias con un directorio vendor, establece tu comando de
compilación en:

```bash
go build -mod=vendor -o server .
```

Las compilaciones con vendor son completamente reproducibles y no requieren acceso
a la red en el momento de la compilación.

## Endpoint de comprobación de estado

Configura una ruta de comprobación de estado dedicada en **Configuración → Estado
y alertas → Ruta de comprobación de estado**. Un endpoint ligero que comprueba
las dependencias críticas le da a Render una señal confiable:

```go
mux.HandleFunc("/healthz", func(w http.ResponseWriter, r *http.Request) {
    if err := db.PingContext(r.Context()); err != nil {
        http.Error(w, "base de datos no alcanzable", http.StatusServiceUnavailable)
        return
    }
    w.WriteHeader(http.StatusOK)
})
```

Mantén las comprobaciones de estado rápidas — se ejecutan en un intervalo corto y
una respuesta lenta cuenta contra el tiempo de espera. Evita consultas costosas
dentro del manejador.

## Despliegues sin tiempo de inactividad

Render realiza despliegues continuos por defecto para los planes de pago: la nueva
versión se inicia y pasa su comprobación de estado antes de que se detenga la
versión anterior. Para aprovechar al máximo esto:

- Asegúrate de que las comprobaciones de estado respondan correctamente en cuanto
  el proceso se vincule a `PORT`, incluso antes de que los trabajadores en segundo
  plano o las cachés estén calientes.
- Haz que los cambios de esquema de base de datos sean compatibles con la versión
  anterior de tu código. Despliega la migración primero, luego el cambio de código.
- Usa el patrón de apagado elegante descrito anteriormente para que la instancia
  antigua drene las solicitudes limpiamente mientras la nueva se está iniciando.

## Escalado horizontal

Agrega instancias a tu servicio desde **Configuración → Escalado**. El servidor
HTTP de la biblioteca estándar de Go es goroutine por conexión y escala bien en
varios núcleos. Algunas cosas que verificar antes de agregar instancias:

- **El estado de la sesión debe estar fuera del proceso.** Usa una instancia de
  Redis de Render o un almacén externo. El estado de sesión en memoria no se
  comparte entre instancias.
- **El trabajo programado debe ser idempotente.** Si usas un `time.Ticker` dentro
  de tu servicio para tareas en segundo plano, varias instancias ejecutarán el
  ticker de forma concurrente. Mueve el trabajo programado sensible al tiempo a un
  servicio dedicado de [Cron Job](https://render.com/docs/cronjobs).
- **Límites de conexión de base de datos.** Cada instancia adicional abre su propio
  grupo de conexiones. Mantente dentro de los límites de conexión de tu plan de
  Postgres o agrega un agrupador de conexiones.

## Lista de verificación de despliegue en producción

Antes de promover un servicio Go a producción en Render, confirma lo siguiente:

- [ ] La aplicación lee `PORT` del entorno y escucha en él.
- [ ] `SIGTERM` es capturado y el servidor se apaga elegantemente.
- [ ] Todas las variables de entorno requeridas se validan al inicio.
- [ ] `DATABASE_URL` usa la cadena de conexión interna para servicios en la misma región.
- [ ] Las migraciones de base de datos se ejecutan como un comando de Pre-Deploy o un paso controlado antes de los despliegues de código.
- [ ] Un endpoint `/healthz` (o equivalente) está registrado y configurado en el panel de control.
- [ ] Los registros van a stdout/stderr en un formato estructurado.
- [ ] `go.sum` está confirmado y actualizado.
- [ ] No hay secretos codificados de forma fija; todas las credenciales se establecen como variables de entorno o archivos secretos.
