# Node.js en Render

Render soporta Node.js de forma nativa. Puedes desplegar cualquier aplicación Node.js como Servicio Web, Trabajador en Segundo Plano o Cron Job — Render detecta tu proyecto, instala las dependencias y ejecuta tu comando de inicio.

## Requisitos

- Un archivo `package.json` en la raíz de tu repositorio (o en un subdirectorio que especifiques como directorio raíz).
- Se recomienda Node.js 18 o posterior. Render usa por defecto una versión LTS reciente; fija una versión específica con un archivo `.node-version` o `.nvmrc`, o con la variable de entorno `NODE_VERSION`.

## Inicio rápido

1. Sube tu repositorio Node.js a GitHub o GitLab.
2. En el panel de control de Render, selecciona **Nuevo → Servicio Web** y conecta tu repositorio.
3. Confirma los comandos de compilación e inicio detectados (ver más abajo) o modifícalos.
4. Haz clic en **Crear Servicio Web**.

Render compila y despliega tu servicio automáticamente en cada push a la rama configurada.

## Comandos de compilación e inicio

Render detecta automáticamente los proyectos Node.js y establece valores predeterminados razonables.

| Configuración | Valor predeterminado |
|---|---|
| **Comando de compilación** | `npm install` |
| **Comando de inicio** | `node index.js` |

Si tu proyecto usa un punto de entrada diferente o una CLI de framework, cambia el comando de inicio. Ejemplos habituales:

| Framework / herramienta | Comando de inicio |
|---|---|
| Express (`server.js`) | `node server.js` |
| Fastify | `node app.js` |
| NestJS | `node dist/main.js` |
| Next.js (servidor) | `npm run start` |

Puedes cambiar cualquiera de los dos comandos desde **Configuración → Compilación e implementación** en cualquier momento.

## Fijar una versión de Node

Crea un archivo `.node-version` o `.nvmrc` en la raíz de tu repositorio:

```
# .node-version
20.14.0
```

También puedes establecer la variable de entorno `NODE_VERSION` en tu servicio. El enfoque con archivo es preferible porque mantiene la versión en el control de versiones y se aplica a todos los entornos de forma consistente.

## Escuchar en PORT

Render inyecta la variable de entorno `PORT` para cada Servicio Web. Tu aplicación debe escuchar en ese puerto:

```js
const express = require('express');
const app = express();

const port = process.env.PORT || 3000;
app.listen(port, () => {
  console.log(`Servidor escuchando en el puerto ${port}`);
});
```

Codificar un número de puerto de forma fija causará que fallen las comprobaciones de estado y que el despliegue se revierta.

## Variables de entorno

Establece las variables de entorno en la pestaña **Entorno** de tu servicio, o referencialas desde un Grupo de Entorno de Render para compartir valores entre servicios. Accede a ellas en tu código mediante `process.env`:

```js
const db = new Client({ connectionString: process.env.DATABASE_URL });
const apiKey = process.env.THIRD_PARTY_API_KEY;
```

Nunca confirmes secretos en tu repositorio. Establece todas las credenciales y claves de API como variables de entorno en el panel de control.

## Comprobaciones de estado

Render envía solicitudes HTTP GET a tu ruta de comprobación de estado configurada (predeterminado: `/`) para verificar que tu servicio está en ejecución. Responde con un código de estado `2xx` en cuanto tu aplicación esté lista:

```js
app.get('/healthz', (req, res) => {
  res.sendStatus(200);
});
```

Configura la ruta en **Configuración → Estado y alertas → Ruta de comprobación de estado**. Un endpoint dedicado que comprueba las dependencias críticas (conectividad de la base de datos, configuración requerida) proporciona una señal más confiable que la ruta raíz.

## TypeScript

Para proyectos TypeScript, compila a JavaScript en el paso de compilación y ejecuta la salida compilada:

| Configuración | Valor de ejemplo |
|---|---|
| **Comando de compilación** | `npm install && npm run build` |
| **Comando de inicio** | `node dist/index.js` |

Asegúrate de que `tsc` o tu herramienta de compilación (esbuild, tsup, etc.) esté listada como dependencia de desarrollo en `package.json` para que esté disponible durante la compilación.

## Archivos estáticos

Evita servir archivos estáticos desde un Servicio Web Node.js en sitios con mucho tráfico. En su lugar, aloja los recursos en un Sitio Estático de Render o en una CDN y deja que tu servicio Node.js gestione solo las solicitudes de API. Si sirves recursos desde Node.js, usa un middleware de caché y establece las cabeceras `Cache-Control` adecuadas.

## Uso de yarn o pnpm

Para usar `yarn` o `pnpm` en lugar de `npm`, actualiza el comando de compilación:

```bash
# yarn
yarn install --frozen-lockfile

# pnpm
npm install -g pnpm && pnpm install --frozen-lockfile
```

Confirma tu archivo de bloqueo (`yarn.lock` o `pnpm-lock.yaml`) para mantener las instalaciones reproducibles.

## Configuración de monorepo

Si tu servicio es uno de varios en un monorepo, establece el **Directorio raíz** del servicio en el subdirectorio que contiene el `package.json` correspondiente. Render ejecutará todos los comandos de compilación e inicio de forma relativa a ese directorio.

Si tu monorepo usa una herramienta de espacios de trabajo (Turborepo, Nx, Lerna), limita el comando de compilación al servicio que se está desplegando:

```bash
# Turborepo: compilar solo el paquete 'api'
npx turbo build --filter=api
```

## Problemas comunes

**La compilación falla con `Cannot find module`**
El módulo no está listado en `package.json` o el archivo de bloqueo está desincronizado. Ejecuta `npm install` localmente, confirma el archivo de bloqueo actualizado y vuelve a desplegar.

**El servicio termina inmediatamente después del despliegue**
Comprueba que la variable de entorno `PORT` se lee y que el servidor se vincula a ella antes de registrar cualquier mensaje de listo. Un proceso que termina sin vincular un puerto se marca como fallido.

**`npm run build` no se reconoce**
El script `build` falta en tu `package.json`. Añádelo o actualiza el comando de compilación en el panel de control con el comando exacto que quieres ejecutar.

**Errores de compilación de TypeScript en producción pero no en local**
Las compilaciones locales pueden usar una versión diferente de Node o TypeScript. Fija `NODE_VERSION` en tu servicio de Render para que coincida con tu entorno local, y añade TypeScript como dependencia de desarrollo para que se use la misma versión en CI y en las compilaciones de producción.
