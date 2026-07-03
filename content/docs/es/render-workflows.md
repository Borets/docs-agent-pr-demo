# Flujos de trabajo de Render

Un flujo de trabajo de Render es la secuencia completa de pasos que se ejecuta
cada vez que despliegas un servicio — desde el momento en que se activa un cambio
hasta el momento en que el tráfico cambia a la nueva versión. Entender cada fase
te permite desplegar más rápido, detectar problemas antes y recuperarte de fallos
con confianza.

## Cómo se activa un despliegue

Cada despliegue comienza con un activador. Render admite tres:

| Activador | Cuándo se ejecuta |
|---|---|
| **Auto-despliegue** | Automáticamente en cada push a tu rama configurada (habilitado por defecto). |
| **Despliegue manual** | A demanda desde el panel de control (**Despliegue manual → Desplegar último commit**) o a través de la API de Render. |
| **Hook de despliegue** | A demanda a través de una URL de webhook autenticada que generas desde la configuración del servicio. |

Configura el auto-despliegue en **Configuración → Compilación e implementación →
Auto-despliegue**. Desactivarlo te permite controlar los despliegues detrás de tu
propio pipeline de CI o paso de aprobación — llama al hook de despliegue o a la
API solo después de que pasen tus verificaciones.

### Hooks de despliegue

Un hook de despliegue es una URL HTTPS única. Una solicitud `POST` a esa URL inicia
un despliegue del último commit en tu rama configurada. Usa los hooks para integrar
los despliegues de Render en cualquier sistema de CI, script o servicio externo sin
darle acceso completo al panel de control.

Genera un hook desde **Configuración → Compilación e implementación → Hook de
despliegue**. Trata la URL como un secreto — cualquiera con la URL puede activar
un despliegue.

## La fase de compilación

Después de que se activa un despliegue, Render ejecuta tu **Comando de compilación**
dentro de un contenedor de compilación nuevo. El contenedor está aislado por
despliegue; nada del sistema de archivos de una compilación anterior persiste (las
cachés de módulos y paquetes se almacenan por separado — ver [Caché](#caché) más
abajo).

Render transmite la salida de la compilación al visor de registros del panel de
control en tiempo real. Un código de salida distinto de cero del comando de
compilación falla el despliegue de inmediato. La versión actual continúa sirviendo
tráfico.

### Comandos de compilación predeterminados por entorno de ejecución

| Entorno de ejecución | Comando de compilación predeterminado |
|---|---|
| Go | `go build -o server .` |
| Node | `npm install; npm run build` |
| Python | `pip install -r requirements.txt` |
| Ruby | `bundle install` |
| Docker | _(imagen construida desde Dockerfile)_ |

Cambia cualquier predeterminado desde **Configuración → Compilación e implementación
→ Comando de compilación**.

### Caché

Render almacena en caché los gestores de paquetes de lenguaje entre compilaciones
(caché de módulos Go, `node_modules`, caché de wheels de pip, etc.). La caché está
indexada por servicio y por rama. Los commits que agregan o eliminan dependencias
invalidan solo los paquetes cambiados — el resto de la caché permanece activo.

Para forzar una compilación limpia, selecciona **Despliegue manual → Borrar caché
de compilación y desplegar** desde el panel de control.

## Comandos de Pre-Deploy

Un **Comando de Pre-Deploy** se ejecuta dentro de un contenedor de corta duración
*después* de que la compilación tiene éxito pero *antes* de que el tráfico cambie
a la nueva versión. Usa este paso para tareas que deben ser atómicas con el
despliegue:

- Migraciones de base de datos
- Calentamiento de caché
- Validación de esquema

Establece el comando en **Configuración → Compilación e implementación → Comando
de Pre-Deploy**. Si el comando termina con un código distinto de cero, Render
cancela el despliegue y deja la versión anterior en su lugar. No se interrumpe
ningún tráfico.

> **Mantén los comandos de pre-deploy idempotentes.** Si un despliegue se reintenta
> después de un fallo parcial, el comando de pre-deploy se vuelve a ejecutar. Las
> migraciones y otras operaciones que cambian el estado deben ser seguras para
> ejecutarse más de una vez.

## Comprobaciones de estado

Una vez que la nueva instancia se inicia, Render espera a que pase su comprobación
de estado antes de enrutar tráfico hacia ella. Para los Servicios Web, Render envía
un HTTP GET a tu **Ruta de comprobación de estado** configurada (predeterminado:
`/`). El servicio debe devolver una respuesta `2xx` dentro de la ventana de tiempo
de espera.

Configura la ruta en **Configuración → Estado y alertas → Ruta de comprobación de
estado**. Un endpoint dedicado — `/healthz` o `/ping` — que comprueba las
dependencias críticas (conectividad de la base de datos, configuración requerida)
te da la señal más confiable.

Si la comprobación de estado nunca tiene éxito:

- El despliegue se marca como fallido.
- La versión anterior continúa sirviendo tráfico.
- La nueva instancia se detiene.

## Ciclo de vida del despliegue de un vistazo

```
Activador recibido
  └─ Comando de compilación se ejecuta
       └─ [Compilación falla] → despliegue cancelado, la versión anterior permanece
       └─ Compilación exitosa
            └─ Comando de Pre-Deploy se ejecuta (si está configurado)
                 └─ [Pre-Deploy falla] → despliegue cancelado, la versión anterior permanece
                 └─ Pre-Deploy exitoso
                      └─ Nueva instancia inicia, comienza comprobación de estado
                           └─ [Comprobación falla] → despliegue fallido, la versión anterior permanece
                           └─ Comprobación exitosa
                                └─ El tráfico cambia a la nueva versión
                                     └─ La instancia anterior drena y se detiene
```

## Despliegues sin tiempo de inactividad

Render realiza despliegues sin tiempo de inactividad en los planes de pago. La
nueva versión debe pasar su comprobación de estado antes de que la versión anterior
reciba una señal de detención. Durante la ventana de superposición, ambas instancias
están en ejecución — diseña tu aplicación para que ambas versiones puedan coexistir
de forma segura:

- Haz que los cambios de esquema de base de datos sean compatibles con la versión
  anterior de tu código.
- No elimines columnas ni cambies nombres de campos en el mismo despliegue que
  agrega código que espera el nuevo esquema.
- Usa el comando de pre-deploy para ejecutar migraciones *antes* de que el nuevo
  código comience a servir tráfico.

## Reversiones

Para revertir a un despliegue anterior, abre el servicio en el panel de control,
ve a la pestaña **Eventos**, encuentra el despliegue que quieres restaurar y
selecciona **Revertir a este despliegue**. Render ejecuta el mismo flujo de trabajo
— incluido el comando de pre-deploy — para la versión revertida.

Como las reversiones reproducen el comando de pre-deploy, asegúrate de que las
migraciones sean reversibles o de que revertir a una versión anterior del código
sea seguro con el esquema de base de datos actual.

## Vistas previas de pull requests

Cuando habilitas las **Vistas previas de pull requests** para un servicio, Render
crea automáticamente una copia temporal del servicio para cada pull request abierto.
La vista previa usa el mismo comando de compilación, variables de entorno (menos
las que excluyas explícitamente) y configuración que tu servicio principal.

Las vistas previas se crean cuando se abre un PR y se eliminan cuando se cierra o
fusiona. Un enlace a la URL de la vista previa se publica directamente en el pull
request.

Habilita las vistas previas en **Configuración → Vistas previas de pull requests**.
Puedes elegir heredar todas las variables de entorno o configurar un conjunto
separado para los entornos de vista previa.

> **Los entornos de vista previa comparten recursos.** Si tu servicio se conecta a
> una base de datos, las vistas previas se conectarán a la misma base de datos a
> menos que reemplaces `DATABASE_URL` para las vistas previas. Usa una base de
> datos o un esquema separado para pruebas de vista previa aisladas.

## Notificaciones y alertas

Render puede notificarte cuando un despliegue tiene éxito, falla o se revierte.
Configura los destinos de notificación en **Configuración → Notificaciones**:

- **Correo electrónico** — enviado al propietario del servicio o a una dirección configurada.
- **Slack** — publica eventos de despliegue en un canal a través de un webhook.
- **PagerDuty / webhook genérico** — reenvía eventos a cualquier endpoint.

Los fallos de comprobación de estado también activan alertas. Establece umbrales
de alerta en **Configuración → Estado y alertas**.

## Patrón de promoción de entornos

Para servicios que progresan a través de entornos de staging y producción, un
patrón común es:

1. **Servicio de staging** — se auto-despliega en cada push a `main`. Los hooks
   de despliegue están deshabilitados.
2. **Servicio de producción** — el auto-despliegue está desactivado. Un hook de
   despliegue se llama solo después de que el staging pasa tus criterios de
   aceptación.

Esto te da una puerta humana o automatizada entre staging y producción mientras
usas la misma infraestructura de flujo de trabajo de Render para ambos entornos.

## Problemas comunes

**La compilación tiene éxito pero el despliegue nunca queda activo**
La comprobación de estado probablemente está fallando. Verifica que tu aplicación
se vincule a la variable de entorno `PORT` y responda `2xx` en tu ruta de
comprobación de estado configurada antes del tiempo de espera.

**El comando de pre-deploy se ejecuta cada vez aunque las migraciones ya estén aplicadas**
Este es el comportamiento esperado — diseña tu herramienta de migración para que
sea una operación sin efecto cuando el esquema ya está actualizado (la mayoría de
los frameworks de migración hacen esto por defecto).

**El auto-despliegue se activa en una rama que no configuré**
Verifica el nombre de la rama en **Configuración → Compilación e implementación →
Rama**. Un cambio de nombre de rama en tu repositorio no actualiza automáticamente
la configuración de Render.

**La versión antigua sigue sirviendo tráfico después de un nuevo despliegue**
En los servicios de nivel gratuito, Render no realiza despliegues sin tiempo de
inactividad. Hay un breve período en el que ninguna versión sirve tráfico mientras
la instancia antigua se detiene y la nueva se inicia.
