# CLAUDE.md

Este archivo le entrega contexto a Claude Code (claude.ai/code) para trabajar en este repositorio.

## Cómo trabajar conmigo (Naty)

1. NUNCA escribas código sin hacerme preguntas de elicitación primero.
2. Después de las preguntas, muéstrame un PLAN antes de codear. El plan debe incluir: qué vas a construir, qué archivos vas a crear o modificar, qué alertas o riesgos tiene, y orden de construcción por etapas.
3. Espera mi aprobación explícita ("dale", "ok", "adelante") antes de empezar.
4. Si encuentras ambigüedad, pregúntame. No asumas.
5. Prefiero pocos cambios probados que muchos cambios a ciegas.
6. Trabajo en sesiones de fase por fase. Después de cada fase muéstrame el diff antes de pasar a la siguiente.
7. No soy desarrolladora. Explícame los conceptos técnicos en simple.
8. Habla en español chileno, directo, sin condescendencia. Si algo que propongo tiene problemas, dímelo aunque incomode.
9. Nunca uses frases tipo "gran pregunta", "excelente observación" o similar. Tono plano y honesto.

## Contexto editorial del Observatorio

- La credibilidad del Observatorio se apoya en honestidad de datos: nunca inventar promedios, fuentes, ni números. Si un dato no existe oficialmente, decirlo explícitamente con disclaimer.
- Los cálculos económicos (DS 67) deben estar siempre basados en tabla oficial y datos SUSESO.
- Todo texto editorial debe evitar sobreprometer lo que el Observatorio no hace.
- Los cambios en el flujo del diagnóstico son críticos: siempre correrlos mentalmente con un caso de usuario real antes de confirmar.
- Refactorización en curso: el cálculo económico del diagnóstico (hoy basado en un factor 0.4%-0.6% sin base normativa) va a ser reemplazado por la tabla oficial DS 67/1999 y datos SUSESO 2024. Mientras no se haga esa refactorización, el cálculo económico existente está identificado como deuda técnica.

## Proyecto

Observatorio SSOMA · Más Prevención — un sitio editorial estático sobre datos de seguridad y salud ocupacional (SSOMA) para Chile, Perú y LATAM, más una herramienta de autodiagnóstico. Todo el contenido y la UI están en español.

El repo tiene exactamente dos archivos que importan:

- `index.html` — el observatorio: dashboards, mapas, gráficos y tablas con datos de SUSESO (Chile), MTPE (Perú) y OIT/ILOSTAT (LATAM).
- `diagnostico.html` — un cuestionario de 5-6 pasos (con rama adicional para perfil mandante) + gate de lead que entrega un reporte de percentil/cumplimiento y un PDF descargable.

No hay build system, ni package manager, ni tests, ni linter, ni código de servidor. Cada archivo HTML es autocontenido: `<style>` inline y `<script>` inline. Todas las librerías de terceros se cargan desde CDN.

## Cómo correr / desarrollar

Abre los HTML directamente en el navegador, o sirve el directorio como estático (por ejemplo `python -m http.server` desde la raíz del repo). No hay paso de instalación. Los cambios se ven al refrescar.

No introduzcas build step, bundler, framework ni pipeline de assets externo salvo que yo lo pida explícitamente — el modelo de despliegue completo depende de que estos sigan siendo HTMLs estáticos drop-in.

## Dependencias por CDN (versiones fijas)

Ambos archivos las usan; revisa las versiones antes de tocar llamadas a su API:

- `index.html`: Leaflet 1.9.4 (mapa LATAM), Chart.js 4.4.1, D3 7.8.5
- `diagnostico.html`: Chart.js 4.4.1, jsPDF 2.5.1 (bundle UMD — se referencia como `window.jspdf.jsPDF`)
- Google Fonts: Fraunces (serif display), Newsreader (serif editorial), IBM Plex Sans/Mono (UI/mono)

## Arquitectura

### Sistema de diseño ("Paleta Medianoche")

Cada HTML declara los design tokens de forma independiente dentro de su propio `:root { ... }`. Las dos declaraciones son casi duplicados intencionales — cuando edites colores, tokens o tipografía, actualiza **los dos archivos** para mantenerlos visualmente en sincronía.

Grupos de tokens que hay que conocer (definidos al inicio del bloque `<style>`):

- `--ink*` / `--paper*` / `--cream` / `--hairline*` — paleta editorial oscura
- `--accent`, `--accent-deep`, `--gold` — acento dorado
- `--risk-1..6` — escala de riesgo de 6 pasos (bajo-verde → crítico-rojo), usada en barras de semáforo y rellenos de mapa
- `--heat-0..5`, `--geo-0..4` — rampas de heatmap / geo LATAM (solo en index.html)
- `--display` / `--editorial` / `--ui` / `--mono` — stacks tipográficos

El PDF que genera `downloadPDF()` en `diagnostico.html` redeclara la misma paleta como arrays RGB (`BG`, `INK`, `ACCENT`, `RISK_*`, etc.) al inicio de esa función — mantén estos sincronizados con los tokens CSS.

### `index.html` — datos + render

Layout: un solo scroll largo con nav, hero, stats globales, y después capítulos por país (Chile → Perú → LATAM → contratistas) separados por secciones `country-divider`. Cada visualización vive en una `viz-section`.

Los datos viven como objetos `const` de JS hardcodeados cerca del final del bloque `<script>`:

- `TASA_RUBRO`, `MORTALIDAD_RUBRO`, `DIAS_PERDIDOS` — Chile por rubro (SUSESO)
- `REGIONES_CHILE`, `FORMAS_FATAL`, `FORMAS_ACCIDENTE`, `MATRIZ` — detalle Chile
- `PERU_SECTORES`, `PERU_REGIONES` — Perú (MTPE; algunas filas marcadas `official: false` son estimaciones)
- `LATAM_DATA` — países LATAM con `{pais, valor, anio, fuente, lat, lng}`

El render es una secuencia de IIFEs (`(function renderX() { ... })()`) que buscan un elemento destino por id y montan SVG/DOM dentro. `scrollReveal` usa `IntersectionObserver` para hacer fade-in de secciones. Un tooltip global se crea una sola vez en `initTooltip()` y se reutiliza vía `showTooltip`/`hideTooltip`/`moveTooltip`.

Cada sección de gráfico tiene una función `exportX()` que reconstruye el dataset actual como filas y llama a `downloadCSV()`. `sectionActionsHTML(hash, exportHandler)` genera el par de botones "Descargar CSV / Compartir sección" que se inyecta en los headers de sección.

El formulario de suscripción postea al endpoint de Formspree `https://formspree.io/f/xrerlvbv`.

### `diagnostico.html` — wizard con estado + PDF

El estado del cliente vive en dos variables a nivel de módulo: `formData` (respuestas acumuladas) y `currentStep` (el id del paso activo, que puede ser `1`, `'1b'`, `2`, `3`, `4`, `5` o `6` — ojo que `'1b'` es string, usado en la rama de mandante).

Flujo de pasos:

1. `startDiagnostico()` oculta el panel de intro y muestra el contenedor de pasos.
2. `nextStep(to)` / `prevStep(to)` navegan; corren validadores (`validateStep1`, `validateStep1B`, `validateStep2`) antes de avanzar, y el perfil mandante intercala el paso `'1b'` entre el 1 y el 2.
3. El paso 3 renderiza el checklist de forma perezosa vía `renderChecklist()`, que elige la lista transversal del país correspondiente (`CHECKLIST_CL` vs `CHECKLIST_PE`) más los ítems sectoriales que corresponden según `formData.subrubro` en `SUBRUBROS[rubro]`, sacando los títulos de ítem desde `ITEMS_CL` / `ITEMS_PE`.
4. `computeAndPreview()` corre `computeMetrics()` (percentil vía CDF normal basada en `erf`, % de cumplimiento, estimación de brecha económica) y muestra el paso 4.
5. El paso 5 es el gate de email; `submitLead()` guarda el lead en `formData.lead` y llama a `renderReport()`, que construye el HTML del paso 6.
6. `downloadPDF()` redibuja el reporte completo desde cero con jsPDF (no rasteriza el DOM).

La lógica por país se ramifica según `formData.pais` (`'CL'` o `'PE'`). Cuando agregues un ítem normativo nuevo, agrégalo en **los dos**: `ITEMS_CL` y `ITEMS_PE` (misma key), y referéncialo desde las sub-categorías correspondientes en `SUBRUBROS`. La misma key resuelve contra el banco del país correcto al momento de renderizar.

Algunos ítems del checklist declaran `appliesIf(n)` y un `naMessage` — los ítems que no aplican para el tamaño de empresa elegido se renderizan como "NO APLICA" y quedan fuera del denominador de cumplimiento. Preserva este comportamiento: `computeMetrics()` selecciona elementos con `[data-applies="1"]` para calcular `cumplimientoPct`.

### Localización / formato

- Números: `toLocaleString('es-CL')` o `'es-PE'` según el contexto de país.
- Fechas: `toLocaleDateString('es-CL', ...)` se usa tanto en el reporte HTML como en el PDF (intencional, incluso para la salida de Perú).
- El símbolo de moneda en el PDF se elige según `formData.pais`: `'S/.'` para Perú, `'$'` para Chile.

## Convenciones para editar datos

- La atribución de fuente tiene que quedar visible: cuando actualices cifras en los bloques `const` de JS, actualiza también cualquier texto de kicker o footer que nombre el año de la fuente (SUSESO 2024, MTPE 2024, OIT/ILOSTAT, etc.) para que el dato mostrado y la fuente citada queden consistentes.
- Las entradas de LATAM traen `anio`; `renderLatam()` deriva las etiquetas "RECIENTE" / "N AÑOS" a partir de `currentYear - c.anio`. `currentYear` está hardcodeado dentro de esa función — súbelo cuando refresques el dataset.
- Las filas de sectores de Perú usan `official: true|false`; las no oficiales se renderizan con menor opacidad y un marcador `est.`. Mantén ese flag honesto.
- Los baselines del diagnóstico (`TASA_RUBRO_2024`, `PROMEDIO_NACIONAL_2024`, `DESVIACION_ESTIMADA`) son de Chile y se usan para el cálculo de percentil en ambos países. Si se agregan baselines reales de Perú, ramifica según `formData.pais` dentro de `computeMetrics()` en vez de sobrescribir los números de Chile.

## Estilo de escritura

El copy es español editorial / periodístico, con énfasis `<em>` dentro de los titulares (por ejemplo, "La <em>seguridad laboral</em>..."). Mantén esa voz y el patrón tipográfico existente (Fraunces para h1/h2, Newsreader para el cuerpo, IBM Plex para labels/kickers de UI) cuando agregues secciones nuevas.
