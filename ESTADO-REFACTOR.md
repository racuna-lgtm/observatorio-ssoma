# Estado de la refactorización del impacto económico

## Fases completadas
- Fase 1 ✅: eliminación completa del cálculo económico antiguo
- Fase 2 ✅: TABLA_DS67 + DATOS_SUSESO_2024 + 4 funciones helper (getCotizacionAdicional, getEscalonDS67, getSiniestralidadProxyRubro, formatoCLP)
- Fase 3 ✅: HTML del bloque condicional de impacto económico
- Fase 4 ✅: lógica de cálculo y renderizado (mostrarFormSiniestralidad, usarProxyRubro, calcularImpacto, renderResultadoImpacto)
- Fase 5 ✅: activación condicional del bloque (if pais CL && tasa > promRubro)

## Fases pendientes
- Fase 6: CSS (estética Medianoche adaptada a tokens reales del archivo)
- Fase 7: validación funcional + commit final + push

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
- (pendiente) — Update de ESTADO-REFACTOR.md

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

## Cómo retomar en próxima sesión
1. Abrir Claude Code desde C:\Proyectos\observatorio-ssoma
2. Pedir a Claude: "Lee ESTADO-REFACTOR.md y retomamos desde Fase 6"
3. Fase 6 es la estética (CSS Medianoche con tokens `--paper` y `--display` + los demás a confirmar). Fase 7 es validación funcional + push final.
