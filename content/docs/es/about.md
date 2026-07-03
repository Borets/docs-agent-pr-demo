# Acerca de este proyecto

Este repositorio existe como un **destino seguro y desechable** para probar y validar
la creación de pull requests por parte del agente de documentación automatizado.

## Por qué existe

Desarrollar herramientas de documentación requiere un repositorio Git real donde
empujar ramas, abrir PRs de prueba e inspeccionar diferencias. Usar un repositorio
de documentación en producción para ese tipo de pruebas conlleva el riesgo de colas
de PRs ruidosas, fusiones accidentales y fatiga en los revisores.

Este proyecto le da al agente de documentación un lugar dedicado para:

- Demostrar que puede crear ramas correctamente delimitadas (`docs-agent/<slug>`).
- Verificar el contenido, el formato y los metadatos de los PRs de borrador antes
  de que cualquier cambio llegue a la documentación real orientada al usuario.
- Permitir que los ingenieros revisen el comportamiento del agente e iteren sobre
  los prompts sin tocar las páginas visibles para los clientes.

## Qué vive aquí

Todo el contenido de prueba está bajo `content/docs/`. Los archivos son
intencionalmente mínimos para que los revisores puedan centrarse en el resultado
del agente en lugar del contenido en sí.

## Qué no vive aquí

Este repositorio **no** es la fuente de verdad para ninguna documentación de
producto. Nada de lo que hay aquí se publica en un sitio de documentación público.
No enlaces a estas páginas desde materiales orientados al cliente.
