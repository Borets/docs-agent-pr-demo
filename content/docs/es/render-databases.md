# Bases de datos PostgreSQL

Render proporciona bases de datos PostgreSQL administradas que se despliegan en segundos y se conectan directamente a tus otros servicios de Render. Esta página cubre la creación de una base de datos, la conexión de tu servicio a ella y su gestión en producción.

## Creación de una base de datos

1. En el panel de control de Render, selecciona **Nuevo → PostgreSQL**.
2. Asigna un nombre a la base de datos y elige una región. Usa la misma región que los servicios que se conectarán a ella para evitar latencia entre regiones.
3. Selecciona un plan. Las bases de datos de nivel gratuito están disponibles para desarrollo y pruebas; caducan después de 90 días.
4. Haz clic en **Crear base de datos**.

Render aprovisiona la base de datos y muestra sus detalles de conexión en la pestaña **Información**.

## Cadenas de conexión

Cada base de datos expone dos cadenas de conexión:

| Cadena | Cuándo usarla |
|---|---|
| **URL de base de datos interna** | Servicios que se ejecutan en la misma región de Render. Sin cargos de salida, menor latencia. |
| **URL de base de datos externa** | Conexiones desde fuera de Render — tu máquina local, un runner de CI o un servicio externo. |

Siempre usa la URL interna para conexiones entre servicios dentro de Render. La URL interna no es accesible desde Internet.

Cuando vinculas una base de datos a un servicio con el botón **Conectar** en el panel de control, Render inyecta automáticamente `DATABASE_URL` (configurada como la URL interna) como variable de entorno en el servicio.

## Conexión desde un servicio

Tras vincular la base de datos, tu servicio lee la cadena de conexión desde `DATABASE_URL`. No se necesitan credenciales codificadas:

```js
// Ejemplo en Node.js con el paquete pg
const { Pool } = require('pg');
const pool = new Pool({ connectionString: process.env.DATABASE_URL });
```

```python
# Ejemplo en Python con psycopg2
import os, psycopg2
conn = psycopg2.connect(os.environ["DATABASE_URL"])
```

```go
// Ejemplo en Go con database/sql
import (
    "database/sql"
    "os"
    _ "github.com/lib/pq"
)

db, err := sql.Open("postgres", os.Getenv("DATABASE_URL"))
```

Valida la conexión al inicio y falla rápido si falta `DATABASE_URL` o no se puede establecer la conexión. Un servicio que arranca sin una conexión funcional a la base de datos probablemente fallará más adelante con tráfico real.

## SSL

Render PostgreSQL requiere SSL para todas las conexiones externas. La URL interna no requiere configuración SSL adicional. Si te conectas desde fuera de Render, añade `?sslmode=require` a la cadena de conexión o configura tu cliente para imponer SSL.

## Ejecución de migraciones

Ejecuta las migraciones antes de que tu nuevo código empiece a servir tráfico. El enfoque recomendado es un **Comando de Pre-Deploy**:

1. Abre tu servicio en el panel de control y ve a **Configuración → Compilación e implementación**.
2. Establece el **Comando de Pre-Deploy** en tu ejecutor de migraciones. Ejemplos:

   ```bash
   # Prisma
   npx prisma migrate deploy

   # golang-migrate
   ./migrate -database "$DATABASE_URL" -path ./migrations up

   # Alembic (Python)
   alembic upgrade head
   ```

3. Si la migración falla, Render cancela el despliegue y la versión anterior de tu servicio continúa ejecutándose con el esquema sin cambios.

Diseña las migraciones para que sean compatibles con la versión anterior de tu código siempre que sea posible. Esto mantiene seguros los despliegues sin tiempo de inactividad.

## Agrupación de conexiones

Cada instancia del servicio abre su propio pool de conexiones. En planes con un límite de conexiones bajo, múltiples instancias escaladas pueden agotar el `max_connections` de la base de datos. Para evitarlo:

- Establece tamaños de pool conservadores en tu aplicación.
- Usa [PgBouncer](https://www.pgbouncer.org/) u otro agrupador de conexiones como servicio separado.
- Actualiza a un plan de Render PostgreSQL con un límite de conexiones más alto.

## Copias de seguridad

Render realiza automáticamente copias de seguridad diarias de las bases de datos de planes de pago. También puedes activar una copia de seguridad manual en cualquier momento desde la pestaña **Copias de seguridad** de tu base de datos. Las copias de seguridad se retienen según la política de retención de tu plan.

Las bases de datos de nivel gratuito no incluyen copias de seguridad automatizadas.

## Acceso directo a la base de datos

Para conectarte a tu base de datos desde una máquina local o ejecutar consultas puntuales, usa la cadena de conexión externa con `psql` o cualquier cliente PostgreSQL:

```bash
psql "$EXTERNAL_DATABASE_URL"
```

También puedes abrir un shell interactivo desde la pestaña **Shell** en la página del panel de control de la base de datos.

## Actualización de PostgreSQL

Render admite actualizaciones de versiones principales de PostgreSQL. Antes de actualizar:

1. Haz una copia de seguridad manual desde la pestaña **Copias de seguridad**.
2. Prueba la actualización en una base de datos que no sea de producción primero.
3. Revisa las notas de la versión de PostgreSQL para detectar cambios incompatibles que afecten a tu aplicación.

Las actualizaciones requieren un breve período de tiempo de inactividad. Planifica la actualización durante una ventana de bajo tráfico.

## Problemas comunes

**Errores de `too many connections`**
Tu aplicación está abriendo más conexiones de las que permite el plan de base de datos. Reduce el tamaño del pool por instancia, añade un agrupador de conexiones o actualiza tu plan.

**`SSL connection is required` al conectarse localmente**
Añade `?sslmode=require` a tu cadena de conexión externa, o establece `PGSSLMODE=require` en tu shell.

**Las migraciones fallan durante el pre-deploy**
Comprueba que `DATABASE_URL` esté disponible en el entorno de pre-deploy. La variable se inyecta automáticamente cuando vinculas la base de datos al servicio desde el panel de control.

**Base de datos de nivel gratuito caducada**
Las bases de datos PostgreSQL de nivel gratuito caducan después de 90 días. Actualiza a un plan de pago o crea una nueva base de datos y restaura tus datos desde una copia de seguridad.
