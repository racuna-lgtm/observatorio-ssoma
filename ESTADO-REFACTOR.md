# Estado de la refactorización del impacto económico

## Fases completadas
- Fase 1 ✅: eliminación completa del cálculo económico antiguo

## Fases pendientes
- Fase 2: agregar TABLA_DS67 + DATOS_SUSESO_2024 + 4 funciones helper (getCotizacionAdicional, getEscalonDS67, getSiniestralidadProxyRubro, formatoCLP)
- Fase 3: HTML del bloque condicional de impacto económico
- Fase 4: lógica de cálculo y renderizado (mostrarFormSiniestralidad, usarProxyRubro, calcularImpacto, renderResultadoImpacto)
- Fase 5: activación condicional del bloque (if pais CL && tasa > promRubro)
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

## Cómo retomar en próxima sesión
1. Abrir Claude Code desde C:\Proyectos\observatorio-ssoma
2. Pedir a Claude: "Lee ESTADO-REFACTOR.md y retomamos desde Fase 2"
3. El plan técnico completo con todo el código de cada fase vive en la conversación previa de Claude.ai donde se trabajó este refactor.
