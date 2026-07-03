# Sitios estáticos

Render puede compilar y servir sitios estáticos directamente desde tu repositorio Git. Los sitios estáticos son gratuitos en todos los planes y se despliegan automáticamente en cada push.

## Qué se considera un sitio estático

Un sitio estático es cualquier proyecto cuya salida de compilación es un directorio de archivos HTML, CSS, JavaScript y recursos. Los frameworks habituales incluyen:

- Next.js (exportación estática)
- Gatsby
- Vite / Create React App
- Hugo, Jekyll, Eleventy
- Astro (modo estático)
- HTML plano

Si tu proyecto necesita un servidor para renderizar respuestas en tiempo de solicitud, despliégalo como **Servicio Web**.

## Creación de un sitio estático

1. Sube tu proyecto a GitHub o GitLab.
2. En el panel de control de Render, selecciona **Nuevo → Sitio estático** y conecta tu repositorio.
3. Establece el **Comando de compilación** y el **Directorio de publicación** (ver valores predeterminados más abajo).
4. Haz clic en **Crear sitio estático**.

Render compila el sitio y lo publica en una URL `*.onrender.com`. Cada push posterior a la rama configurada activa automáticamente una nueva compilación y despliegue.

## Comando de compilación y directorio de publicación

Render necesita dos datos para compilar y servir tu sitio:

| Configuración | Descripción |
|---|---|
| **Comando de compilación** | El comando que genera la salida estática. |
| **Directorio de publicación** | El directorio que Render sirve una vez completada la compilación. |

Valores predeterminados habituales por framework:

| Framework | Comando de compilación | Directorio de publicación |
|---|---|---|
| Create React App | `npm run build` | `build` |
| Vite | `npm run build` | `dist` |
| Next.js (exportación estática) | `npm run build` | `out` |
| Gatsby | `gatsby build` | `public` |
| Hugo | `hugo` | `public` |
| Eleventy | `npx @11ty/eleventy` | `_site` |

Puedes cambiar ambas configuraciones desde **Configuración → Compilación e implementación** en cualquier momento.

## Dominios personalizados

Añade un dominio personalizado desde la pestaña **Configuración → Dominios personalizados**:

1. Introduce el nombre de tu dominio y haz clic en **Guardar**.
2. Añade un registro `CNAME` que apunte a tu dirección `*.onrender.com` con tu proveedor de DNS.
3. Render aprovisiona y renueva automáticamente el certificado TLS.

Los dominios raíz (por ejemplo, `ejemplo.com` sin `www`) requieren un registro `ALIAS` o `ANAME`. No todos los proveedores de DNS admiten estos tipos de registros; consulta con tu proveedor si la opción no está disponible.

## Redirecciones y reescrituras

Para aplicaciones de página única (SPA), necesitas reescribir todas las rutas a `index.html` para que el enrutamiento del lado del cliente funcione correctamente. Crea un archivo `_redirects` en la raíz de tu **directorio de publicación**:

```
/* /index.html 200
```

También puedes definir redirecciones explícitas:

```
/ruta-antigua /ruta-nueva 301
/blog/* /posts/:splat 302
```

Las reglas se evalúan de arriba a abajo. Gana la primera regla que coincida.

## Cabeceras

Establece cabeceras de respuesta HTTP personalizadas con un archivo `_headers` en tu directorio de publicación:

```
/*
  X-Frame-Options: DENY
  X-Content-Type-Options: nosniff
  Content-Security-Policy: default-src 'self'

/assets/*
  Cache-Control: public, max-age=31536000, immutable
```

## Variables de entorno

Las variables de entorno en tiempo de compilación están disponibles durante el proceso de compilación. Configúralas en **Configuración → Entorno**.

Ten en cuenta que las variables de entorno en los sitios estáticos son **solo de tiempo de compilación**. Se incorporan a los archivos compilados. No pongas secretos ni credenciales en las variables de entorno de un sitio estático — los valores pueden ser visibles en la salida compilada.

Para frameworks que requieren un prefijo (por ejemplo `VITE_` para Vite o `REACT_APP_` para Create React App), añade el prefijo al establecer el nombre de la variable.

## Vistas previas de pull requests

Habilita las **Vistas previas de pull requests** en **Configuración → Vistas previas de pull requests**. Render compila y despliega un sitio de vista previa separado para cada pull request abierto. Un enlace a la vista previa se publica directamente en el PR.

Los sitios de vista previa se eliminan cuando se cierra o fusiona el PR.

## Caché de compilación

Render almacena en caché `node_modules` y otros directorios de gestores de paquetes entre compilaciones. La mayoría de las instalaciones de dependencias se sirven desde la caché, por lo que las compilaciones solo ejecutan el paso de compilación real de tu framework.

Para borrar la caché y forzar una reinstalación completa, selecciona **Despliegue manual → Borrar caché de compilación y desplegar** desde el panel de control.

## Problemas comunes

**El sitio carga pero las rutas del lado del cliente devuelven 404**
A tu directorio de publicación le falta un archivo `_redirects` con la regla de reescritura `/* /index.html 200`. Añade el archivo y vuelve a desplegar.

**La compilación falla con `command not found`**
El comando de compilación hace referencia a una CLI que no está instalada. Añádela como dependencia de desarrollo en tu `package.json` o instálala explícitamente en el comando de compilación.

**La variable de entorno no está disponible en tiempo de ejecución**
Las variables de entorno de los sitios estáticos solo se inyectan en tiempo de compilación. Si falta una variable en la salida compilada, confirma que estaba establecida antes de la última compilación y vuelve a desplegar para recoger el nuevo valor.

**El dominio personalizado muestra un error de certificado**
Render aprovisiona TLS automáticamente una vez que se propaga el registro `CNAME` (normalmente en pocos minutos o una hora). Si el error persiste después de una hora, verifica que el registro DNS apunta a la dirección `*.onrender.com` correcta que se muestra en el panel de control.
