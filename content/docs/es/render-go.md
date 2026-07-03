# Render Go

Render soporta Go de forma nativa. Puedes desplegar cualquier aplicación Go como
un Servicio Web, Trabajador en Segundo Plano o Cron Job sin necesidad de herramientas
adicionales — Render detecta tu módulo Go, compila tu binario y lo ejecuta.

## Requisitos

- Un archivo `go.mod` en la raíz de tu repositorio (o en un subdirectorio que
  especifiques como directorio raíz).
- Se recomienda Go 1.20 o posterior. Render utiliza por defecto una versión estable
  reciente; fija una versión específica con la variable de entorno `GOVERSION` si
  tu aplicación lo requiere.

## Inicio rápido

1. Sube tu repositorio Go a GitHub o GitLab.
2. En el panel de control de Render, selecciona **Nuevo → Servicio Web** y conecta
   tu repositorio.
3. Confirma los comandos de compilación e inicio detectados (ver más abajo) o
   cámbialos.
4. Haz clic en **Crear Servicio Web**.

Render compila y despliega tu servicio automáticamente en cada push a la rama
configurada.

## Comandos de compilación e inicio

Render detecta automáticamente los proyectos Go y establece valores predeterminados
razonables.

| Configuración | Valor predeterminado |
|---|---|
| **Comando de compilación** | `go build -o server .` |
| **Comando de inicio** | `./server` |

Si tu paquete `main` está en un subdirectorio, ajusta el comando de compilación
según corresponda:

```bash
# Comando de compilación para un proyecto cuyo paquete main está en cmd/api
go build -o server ./cmd/api
```

Puedes cambiar cualquiera de los dos comandos en la sección **Configuración →
Compilación e implementación** de tu servicio en cualquier momento.

## Fijar una versión de Go

Establece la variable de entorno `GOVERSION` en tu servicio para solicitar una
versión específica:

```
GOVERSION=1.22.4
```

Render instala la versión solicitada en el momento de la compilación. La variable
debe estar establecida antes del próximo despliegue para que surta efecto.

## Variables de entorno

Las aplicaciones Go que se ejecutan en Render leen las variables de entorno de la
misma manera que lo hacen localmente, mediante `os.Getenv`. Establece las variables
en la pestaña **Entorno** de tu servicio o referencialas desde un Grupo de Entorno
de Render para compartir valores entre servicios.

Tu aplicación debe leer la variable de entorno `PORT` para saber en qué puerto
escuchar:

```go
port := os.Getenv("PORT")
if port == "" {
    port = "8080"
}
http.ListenAndServe(":"+port, nil)
```

Render inyecta `PORT` automáticamente para los Servicios Web. Codificar un número
de puerto de forma fija causará que fallen las comprobaciones de estado.

## Comprobaciones de estado

Render envía solicitudes HTTP GET a `/` por defecto para verificar que tu servicio
está activo. Configura una ruta diferente en **Configuración → Estado y alertas →
Ruta de comprobación de estado** si tu aplicación expone un endpoint dedicado (por
ejemplo `/healthz` o `/ping`).

La comprobación espera una respuesta 2xx dentro del tiempo de espera predeterminado.
Los servicios que no se vinculan a `PORT` a tiempo se marcan como no saludables y
el despliegue se revierte.

## Archivos estáticos y recursos

Si tu aplicación sirve recursos estáticos, incrústalos con `//go:embed` o
inclúyelos de forma relativa al directorio de trabajo. El directorio de trabajo
en tiempo de ejecución es la raíz de tu repositorio. Las rutas que dependen de
una ruta absoluta del equipo de compilación no se resolverán correctamente.

## Módulos privados

Para obtener módulos de un repositorio privado, agrega una variable `GONOSUMCHECK`
o `GONOSUMDB` y proporciona credenciales a través de una variable de entorno
`NETRC` o un archivo `~/.netrc` inyectado mediante un archivo secreto. Render no
almacena en caché las descargas de módulos privados fuera del contenedor de
compilación, por lo que cada despliegue descarga copias nuevas.

## Configuración de monorepo

Si tu servicio es uno de varios en un monorepo, establece el **Directorio raíz**
del servicio en el subdirectorio que contiene `go.mod`. Render ejecutará todos los
comandos de compilación e inicio de forma relativa a ese directorio.

## Cron Jobs y trabajadores en segundo plano

Los binarios Go funcionan igualmente bien como Trabajadores en Segundo Plano o
Cron Jobs en Render. Selecciona el tipo de servicio apropiado al crear el servicio.
Los Cron Jobs no necesitan vincularse a `PORT`; los Trabajadores en Segundo Plano
solo deben hacerlo si también exponen un endpoint HTTP interno o público.

## Problemas comunes

**La compilación falla con `no Go files in .`**
El comando de compilación se está ejecutando en el directorio incorrecto. Establece
el **Directorio raíz** en la carpeta que contiene tu `go.mod`, o actualiza el
comando de compilación para apuntar a la ruta de paquete correcta (por ejemplo
`go build -o server ./cmd/myapp`).

**El servicio se reinicia inmediatamente después del despliegue**
Verifica que tu aplicación lea `PORT` del entorno y escuche en ese puerto. Un
proceso que termina de inmediato (por ejemplo, porque el puerto ya está en uso o
el indicador es incorrecto) hace que Render lo reinicie repetidamente.

**Errores de descarga de módulos durante la compilación**
Confirma que todas las dependencias estén listadas en `go.sum` y confirmadas. Si
usas un proxy de módulos privado, establece `GOPROXY` y las credenciales necesarias
como variables de entorno en el servicio.
