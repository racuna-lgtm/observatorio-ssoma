# Estado de la refactorización del impacto económico

## Fases completadas
- Fase 1 ✅: eliminación completa del cálculo económico antiguo
- Fase 2 ✅: TABLA_DS67 + DATOS_SUSESO_2024 + 4 funciones helper (getCotizacionAdicional, getEscalonDS67, getSiniestralidadProxyRubro, formatoCLP)
- Fase 3 ✅: HTML del bloque condicional de impacto económico
- Fase 4 ✅: lógica de cálculo y renderizado (mostrarFormSiniestralidad, usarProxyRubro, calcularImpacto, renderResultadoImpacto)
- Fase 5 ✅: activación condicional del bloque (if pais CL && tasa > promRubro)
- Fase 6 ✅: CSS del bloque en estética Medianoche, scoped bajo `.impacto-economico`
- Fase 7 ✅: validación funcional end-to-end (6 casos, pasó al primer intento sin fixes)

## Fases pendientes
— ninguna, refactor completo.

## Decisiones tomadas en sesión previa

### Nombres de rubros — opción A confirmada
Los datos SUSESO se escriben con los nombres EXACTOS del formulario actual (no los canónicos SUSESO):
- 'Industria Manufacturera'
- 'Agricultura y pesca'
- 'Transporte y com.'
- 'Comercio'
- 'Construcción'
- 'Servicios'
- 'Electricidad/Gas/Agua'
- 'Minería'
- 'Promedio Nacional' (para fallback cuando no aplique ningún rubro)

### Nombres de formData reales (NO los que dice el plan original)
- formData.pais ('CL' | 'PE')
- formData.rubro (string, value del select)
- formData.trabajadores (NO nTrabajadores)
- formData.renta (NO rentaBruta)
- formData.tasa

### Tokens CSS reales
- --paper (fondo oscuro, NO --bg-dark)
- --display (Fraunces serif, NO --font-display)
- Los demás tokens se confirmarán al aplicar Fase 6

### Observaciones sobre el estado previo al refactor
- La "tarjeta económica del resumen ejecutivo" NO existía como tarjeta separada. El resumen ejecutivo tiene 3 tarjetas (percentil, cumplimiento, brecha). La info económica vivía en impactoPhrase (narrativa) + sección V completa. Ambas eliminadas en Fase 1.
- El factor 1.08 de planillaAnual fue eliminado en commit anterior 6c062ab (parche en GitHub web, antes de Fase 1).
- formatMM era helper huérfano que se usaba solo desde impactoPhrase. Eliminado como parte de Fase 1.

### Validaciones matemáticas confirmadas para Fase 2
- getCotizacionAdicional(180) → 1.70
- getCotizacionAdicional(60) → 0.34
- getSiniestralidadProxyRubro('Construcción') → 60.48
- Caso de prueba Fase 7: 9 trabajadores × $800.000 × 12 = $86.400.000 × 1.70% = $1.468.800

## Decisiones nuevas sesión 2

### Commits de la sesión
- e546876 — Fase 2
- 84db157 — Fase 3
- a20d96f — Fase 4
- 592d6fb — Fase 5
- 6294d58 — Update de ESTADO-REFACTOR.md

### Exploración previa
- `renderReport` es el nombre real de la función que renderiza el reporte completo (sin parámetros, `function renderReport()`).
- `computeMetrics` retorna un objeto con los campos `m.tasa` y `m.promRubro` correctos. No hubo que adaptar el `if` de Fase 5.
- Los 5 campos de formData (`pais`, `rubro`, `trabajadores`, `renta`, `tasa`) siguen intactos post-Fase 1.
- Grep de los 10 identificadores nuevos: cero choques de nombres. Inserción limpia.

### Puntos de inserción aplicados
- Fase 2: entre cierre de `SUBRUBROS` (línea 1378 pre-edit) y `// Estado` / `let formData = {}`.
- Fase 3: entre cierre del bloque "I · Resumen ejecutivo" y el comentario `<!-- II. Cumplimiento legal -->`, dentro del template literal de `renderReport()`.
- Fase 4: inmediatamente después de `formatoCLP`, pegado a Fase 2 sin código intercalado.
- Fase 5: entre `renderComplianceUI(m);` y el `}` de cierre de `renderReport`.

### Comportamiento conocido (no bug)
El `<section id="impacto-economico">` se re-inyecta en cada llamada a `renderReport()` porque vive dentro del template literal del reporte. Cualquier estado interno (formulario abierto, resultado calculado) se pierde si `renderReport()` se vuelve a ejecutar. Aceptable para el flujo actual; considerar en Fase 6/7 si se vuelve relevante.

### Desviaciones menores aprobadas
- Fase 3: se agregó un comentario HTML explicativo encima del `<section id="impacto-economico">` que documenta la activación condicional desde `renderReport`. Aprobado explícitamente.

### Validación mínima de Fase 2
Saltada por fricción técnica al generar el script de validación en Windows (primer intento: one-liner con código corrupto; segundo intento: archivo temporal con contenido corrupto). La validación real queda para Fase 7. Decisión consistente con la regla del prompt: la validación matemática no era bloqueante.

### Alcance geográfico: Chile únicamente (decisión de diseño)
El bloque de impacto económico aplica exclusivamente a usuarios que marcaron Chile en el formulario. Tres razones que anclan esta decisión:
1. **Guard explícito en Fase 5:** `if (formData.pais === 'CL' && m.tasa > m.promRubro)`. Para usuarios de Perú, el `<section>` queda oculto por `display:none;` y nunca se activa.
2. **Contenido editorial Chile-específico:** el texto del bloque referencia ACHS, Mutual CCHC, IST, ISL, Carta Mutual/ISL, DS 67/1999, SUSESO 2024. Nada de eso aplica al sistema peruano.
3. **Tabla y datos son del sistema chileno:** `TABLA_DS67` es la tabla del Decreto Supremo 67/1999 de Chile. `DATOS_SUSESO_2024` son datos del organismo chileno SUSESO.

Para usuarios de Perú con tasa alta, el resto del reporte (resumen ejecutivo, checklist legal, narrativa) sigue renderizando sin el bloque económico — verificado en sesión 2 con grep de referencias cruzadas.

**Pendiente estratégico (fuera del scope actual):** evaluar si más adelante se construye un bloque económico equivalente para Perú basado en SCTR (Seguro Complementario de Trabajo de Riesgo, regulado por SBS/SUNAFIL, con primas por nivel de riesgo I-V). Sería una fase nueva a futuro, no parte de este refactor. Decisión sujeta a priorización de producto.

## Decisiones sesión 3

### Tokens reales del sistema Medianoche confirmados

| Familia | Token | Valor |
|---|---|---|
| Texto | `--ink` | `#EEE9DE` (marfil cálido, texto principal) |
| | `--ink-soft` | `#D4CFC4` |
| | `--ink-muted` | `#8E8B81` |
| | `--ink-faint` | `#5C5A52` |
| Fondos | `--paper` | `#0F1419` (fondo oscuro profundo) |
| | `--paper-soft` | `#1A1F26` |
| | `--paper-warm` | `#141820` |
| | `--cream` | `#252A32` |
| Hairlines | `--hairline` | `#2D3340` |
| | `--hairline-soft` | `#20252E` |
| Acentos | `--accent` | `#D4B98F` (dorado) |
| | `--accent-deep` | `#B89B6F` |
| | `--accent-soft` | `#2C2518` |
| Semánticos | `--critical` | `#C06846` |
| | `--critical-soft` | `#3A1F19` |
| | `--positive` | `#5F8060` |
| Escala riesgo | `--risk-1` | `#5F8060` |
| | `--risk-2` | `#8E9B5E` |
| | `--risk-3` | `#C9B353` |
| | `--risk-4` | `#D49447` |
| | `--risk-5` | `#C06846` |
| | `--risk-6` | `#9B3828` |
| Tipografía | `--display` | `'Fraunces', serif` |
| | `--editorial` | `'Newsreader', Georgia, serif` |
| | `--ui` | `'IBM Plex Sans', system-ui, sans-serif` |
| | `--mono` | `'IBM Plex Mono', monospace` |

Nota: `--gold` NO existe como token independiente — el dorado del sistema es `--accent`. Se eliminó la mención a `--gold` del plan original al confirmar esto en la exploración del Paso 1.

### Tokens semánticos reutilizados en vez de declarar locales
Decisión tomada al descubrir en el Paso 1 que el sistema ya tenía tokens semánticos adecuados. Se evitó contaminar el bloque con tokens locales redundantes (los `--ie-danger/warning/success` que originalmente proponía el plan).

- `.fila-usuario` → `--critical` en `border-left`
- `.fila-rubro` → `--risk-3` en `border-left`
- `.fila-aspiracional` → `--positive` en `border-left`
- `.warning-proxy` → `--risk-3` en `border-left`
- Fondo tenue de `.fila-rubro-escalon` → `rgba(201, 179, 83, 0.03)` (componentes RGB de `--risk-3` con alpha bajo, para marcar sutilmente el punto de referencia)

### Decisión sobre emojis 🔴🟡🟢🟠
Atenuación tipográfica con `filter: grayscale(0.3)` y `opacity: 0.95` sobre el `<td>:first-child` de ambas tablas (principal y escalones).

**Limitación conocida:** el `filter` se hereda al `<strong>` interno de la celda y afecta marginalmente al texto del escenario. Dado que `--ink` (#EEE9DE) es casi-neutro, el impacto visual sobre el texto es imperceptible en uso real; validado en Fase 7.

**Semántica fuerte:** queda anclada en el `border-left: 3px solid` de cada fila, siguiendo el patrón establecido por `.summary-stat` y `.rec-card` en el resto del reporte. El color del borde comunica la categoría, no el emoji.

**Upgrade futuro (Fase 8+):** círculos CSS perfectos requerirían wrapping HTML de los emojis en `<span class="ie-dot">`, lo cual queda fuera del scope de Fase 6 (CSS puro). Registrado como mejora potencial pendiente de priorización.

### Rol del usuario (operativa / mandante / contratista)
Decisión de diseño validada en Fase 7 con Casos E y F: el rol NO modula el bloque económico en esta iteración. El cálculo DS 67 aplica uniformemente a los 3 roles, porque todos pagan cotización adicional por su propia planilla.

Las diferencias narrativas por rol son relevantes pero fuera del scope de este refactor:
- **Mandante:** responsabilidad solidaria Art. 66 bis Ley 16.744 + DS 76 sobre accidentes de contratistas.
- **Contratista:** impacto reputacional/contractual del nivel de cotización en procesos de licitación.

Registrado como mejora futura Fase 8+ pendiente de priorización.

### Ajustes aplicados durante Fase 7
Ninguno. Fase 7 pasó sin fixes. Los 6 casos (A camino principal, B proxy, C guard Perú, D guard tasa baja, E mandante, F contratista) validaron OK al primer intento.

### Commits de la sesión 3
- 1fed677 — Fase 6 (CSS Medianoche)
- b829d3e — Fase 7 (validación end-to-end, commit marker sin cambios de código)
- (pendiente) — Cierre de refactor + update de ESTADO-REFACTOR.md

## Pendientes de producto fuera del refactor DS 67

### Export PDF del reporte
El export PDF va por una ruta distinta a `renderReport()` — la función `downloadPDF()` redibuja el reporte completo con jsPDF desde cero en vez de rasterizar el DOM, y no incluye el bloque de impacto económico. Queda como issue para próxima iteración. Decisión pendiente de producto: ¿se agrega una sección al PDF equivalente al bloque HTML, se ofrece descarga separada, o se deja el impacto económico solo como vista web?

### Datos de contacto del formulario (nombre, cargo, email, teléfono)
Naty levantó durante Fase 7 la duda sobre qué pasa con esos datos cuando un usuario completa el diagnóstico. No se investigó en esta sesión.

Pendientes paralelos:
1. **Técnico:** rastrear si se envían a algún endpoint, se guardan localmente, o se pierden al cerrar el navegador.
2. **Producto:** definir qué hacer con ellos, considerando la Ley 21.719 chilena de protección de datos personales (consentimiento explícito, finalidad declarada, derechos ARCO, etc.).

Conversación dedicada de producto pendiente antes de cualquier implementación.

## Cómo retomar en próxima sesión
Refactor DS 67 cerrado. Para trabajo futuro sobre el bloque de impacto económico o temas relacionados, ver la sección "Pendientes de producto fuera del refactor DS 67" más arriba.
