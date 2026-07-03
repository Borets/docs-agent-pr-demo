# Primeros pasos

Esta página es intencionalmente pequeña para que los PRs del agente de documentación
sean fáciles de revisar.

## Configuración local

1. Instala las dependencias.
2. Inicia el servidor de desarrollo.
3. Abre el sitio de documentación local.

## Escalado automático

Render puede escalar tu servicio hacia arriba o hacia abajo automáticamente en
función del uso de CPU o memoria. Configura el escalado automático en
**Configuración → Escalado** para cualquier servicio web o worker en segundo plano.

### Políticas de escalado

| Configuración | Qué controla |
|---|---|
| **Instancias mínimas** | El límite inferior — Render nunca escala por debajo de este número. Establécelo en `1` o más para evitar arranques en frío. |
| **Instancias máximas** | El límite superior — Render no aprovisionará más instancias que este valor, independientemente de la carga. |
| **Umbral de escala ascendente** | Porcentaje de uso de CPU o memoria en el que Render agrega una instancia. Un punto de partida habitual es `75 %`. |
| **Umbral de escala descendente** | Porcentaje de uso en el que Render elimina una instancia. Mantenlo bien por debajo del umbral de escala ascendente (p. ej. `25 %`) para evitar oscilaciones. |
| **Período de enfriamiento** | Segundos mínimos entre eventos de escalado consecutivos. El valor predeterminado es `300 s`; auméntalo para servicios con tiempos de inicio lentos. |

### Consejos

- **Las comprobaciones de estado controlan la escala ascendente.** Una nueva
  instancia empieza a recibir tráfico solo después de superar su endpoint de
  comprobación de estado. Configura el endpoint en
  **Configuración → Estado y alertas** antes de habilitar el escalado automático.
- **El estado de sesión debe ser externo.** Varias instancias no comparten
  memoria. Almacena las sesiones en una instancia de Redis de Render u otro
  almacén externo.
- **Las conexiones de base de datos se multiplican.** Cada instancia abre su
  propio grupo de conexiones. Mantente dentro del límite de conexiones de tu plan
  de base de datos; considera un agrupador de conexiones como PgBouncer si
  escalas a muchas instancias.
- **El escalado depende del plan.** El escalado automático está disponible en
  planes de pago. Comprueba **Configuración → Escalado** para confirmar que está
  habilitado para tu tipo de servicio.

## Validación

Antes de abrir un PR, ejecuta las verificaciones del proyecto e incluye los
resultados en la descripción del PR.
