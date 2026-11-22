# Prompt Maestro — Brief Diario "Panorama Global y de México"

> Documento de referencia técnica y de instrucción para generar, de forma consistente, el brief de inversión diario en formato infografía (estilo Gemini Notebook), con investigación verificada, PDF a 2 hojas carta y posts de LinkedIn. Creado a partir del proceso iterativo usado entre el 21 de agosto y el 16 de septiembre de 2026.

---

## 1. Prompt de instrucción (copiar y pegar, ajustando la fecha)

```
Hagamos el panorama [de apertura / al cierre] del día de hoy [día de la semana] [fecha] de [mes] de [año],
con una expectativa de resto de semana que comprende [rango de días restantes],
y resumen de [día(s) anterior(es) ya transcurridos de esta semana] y fin de semana
(del sábado [fecha] al domingo [fecha]), usa el mismo formato que hemos manejado,
estructura y lógica que hemos hecho.
```

Variantes válidas de corte:
- **Apertura / premercado**: para reportes temprano en el día, antes o al inicio de la sesión.
- **Cierre de mercados**: para reportes después del cierre de Wall Street/BMV.
- **Cierre de mes**: agregar explícitamente "resumen del cierre de mes de [mes]" cuando el corte coincide con fin de mes — se añade una tabla de "Resumen Mensual" comparando cierre del mes anterior vs. cierre del mes actual.

Instrucción permanente ya acordada en este proceso:
> Cada vez que se genere esta actualización, entregar en el mismo chat (no dentro del HTML) dos posts de LinkedIn: uno divulgativo/informativo largo, y uno resumen corto con hashtags.

---

## 2. Rol y marco analítico (skill base)

Este brief se ejecuta bajo la lógica del skill **`inteligencia-inversion-macro-geopolitica`**: actuar como analista senior de inteligencia geopolítica, macroeconómica y de mercados para un inversionista mexicano con acceso a BMV, SIC, NYSE, Nasdaq, ETFs, renta fija, USD/MXN, oro y commodities.

Reglas no negociables heredadas de ese skill:
- Separar **hechos confirmados** de **expectativas de mercado** / **interpretación analítica**. Nunca presentar una expectativa (ej. "la Fed subirá tasas") como hecho consumado si el anuncio oficial todavía no ocurrió al momento del corte.
- Toda cifra de mercado lleva **fecha, hora (si aplica) y fuente**.
- Si no hay información suficiente o verificable, decirlo explícitamente en vez de inventar un dato.
- No se emiten recomendaciones directas de compra/venta. Es análisis informativo de riesgos y escenarios.
- Siempre cerrar con la leyenda de responsabilidad (ver sección 6, bloque `.legend`).

---

## 3. Proceso técnico paso a paso

### Paso 1 — Investigación con fuentes verificadas
Usar `WebSearch` (y si es necesario `WebFetch`/agente `web-fetch`) para obtener, como mínimo:

| Bloque | Qué buscar |
|---|---|
| Mercados México | Cierre S&P/BMV IPC (puntos y % del día), USD/MXN (FIX de Banxico y spot de cierre) |
| Mercados EE. UU. | Cierre S&P 500, Nasdaq, Dow Jones (puntos y %), VIX |
| Commodities | Petróleo Brent y WTI (USD/barril y %), Oro (futuros, USD/oz) |
| Tasas / banco central | Tasa objetivo Banxico, decisión/expectativa de la Fed, probabilidad CME FedWatch si hay reunión próxima |
| Inflación México | Dato INEGI más reciente (quincenal o mensual), general y subyacente |
| Geopolítica | Evento dominante de la semana (en este periodo: crisis del Estrecho de Ormuz, ataques a infraestructura petrolera, Jackson Hole, FOMC) |
| Noticias México | 3-4 notas de alto impacto local (política, economía, sociedad, mercados) |
| Agenda | Eventos confirmados para los días que pide el usuario (lo que "viene"), nunca inventados |

Reglas de búsqueda:
- Buscar primero por fecha exacta ("cierre BMV [fecha]", "USD/MXN [fecha]", "petróleo Brent WTI [fecha]").
- Si una cifra aparece distinta en dos fuentes, usar la más reciente/oficial y anotar la discrepancia o marcarla como aproximada (`~`) en la tabla.
- Si el evento central del día (ej. decisión de la Fed) **aún no se ha anunciado** al momento del corte, decirlo explícitamente en el resumen y en la nota metodológica final — no inventar el resultado.
- Fuentes primarias preferidas: Banxico, INEGI, Federal Reserve, CME FedWatch, BLS.
- Fuentes secundarias de respaldo: Investing.com, El Financiero, El CEO, Infobae, CNBC, Bloomberg Línea, TheStreet, Milenio, dolardof.com.

### Paso 2 — Construcción del HTML
1. Partir siempre de la **última versión publicada** del archivo (mismo CSS/estructura; no rediseñar desde cero cada vez).
2. Actualizar únicamente: `<title>`, masthead (tag de fecha + título + subtítulo), bloque `.summary` (3-4 párrafos cronológicos), tablas de "Cambios Materiales" e "Indicadores Económicos Clave", "Noticias de Alto Impacto", "México 60 Segundos", timeline de "Agenda y Riesgos"/"Expectativa Resto de Semana", y el `.footer-band` + `.legend`.
3. Mantener la paleta y tipografía (ver sección 6) y el bloque `@media print` íntegro — ahí es donde se ajusta el tamaño a 2 hojas carta.
4. Guardar el archivo en el **scratchpad de la sesión**, no en `/tmp` directo.

### Paso 3 — Generar el PDF (Chromium headless vía Playwright)
No depender del diálogo de impresión del navegador del usuario (falla en Windows / produce 3 hojas con cortes). Generarlo del lado del servidor:

```js
// make_pdf.js
const { chromium } = require('/opt/node22/lib/node_modules/playwright');
const path = require('path');

(async () => {
  const browser = await chromium.launch({ executablePath: '/opt/pw-browsers/chromium-1194/chrome-linux/chrome' });
  const page = await browser.newPage();
  const filePath = 'file://' + path.resolve(process.argv[2]);
  await page.goto(filePath, { waitUntil: 'networkidle' });
  await page.emulateMedia({ media: 'print' });
  await page.pdf({
    path: process.argv[3],
    format: 'Letter',
    printBackground: true,
    margin: { top: '0mm', bottom: '0mm', left: '0mm', right: '0mm' }
  });
  await browser.close();
})();
```

```bash
node make_pdf.js panorama-XX.html panorama-XX.pdf
```

> ⚠️ **Gotcha de ruta**: el binario de Chromium **no** vive en `/opt/pw-browsers/chromium/chrome-linux/chrome` sino en `/opt/pw-browsers/chromium-1194/chrome-linux/chrome` (el nombre de carpeta incluye el número de build). Verificar con `find /opt/pw-browsers -iname "*chrome*" -type f` si falla el lanzamiento.

### Paso 4 — Verificar que sean EXACTAMENTE 2 páginas
No asumir que "se ve bien" — contar páginas reales del PDF generado:

```bash
python3 - << 'EOF'
import re
with open('archivo.pdf','rb') as f:
    data = f.read()
pages = re.findall(rb'/Type\s*/Page[^s]', data)   # ojo: excluir "/Type /Pages" (el nodo padre)
print('Pages:', len(pages))
EOF
```

Si da 3 o más páginas: reducir aún más los tamaños de fuente/padding dentro de `@media print` (no cambiar el contenido) e iterar hasta que dé 2.

### Paso 5 — Verificación visual (nada cortado)
Tomar una captura del documento ya renderizado en modo impresión, a tamaño carta, y revisarla con el tool de lectura de imágenes antes de publicar:

```js
// screenshot_print.js
const { chromium } = require('/opt/node22/lib/node_modules/playwright');
const path = require('path');

(async () => {
  const browser = await chromium.launch({ executablePath: '/opt/pw-browsers/chromium-1194/chrome-linux/chrome' });
  const page = await browser.newPage({ viewport: { width: 816, height: 1200 } });
  await page.goto('file://' + path.resolve(process.argv[2]), { waitUntil: 'networkidle' });
  await page.emulateMedia({ media: 'print' });
  await page.screenshot({ path: process.argv[3], fullPage: true });
  await browser.close();
})();
```

Revisar explícitamente: que ninguna tabla, tarjeta o párrafo quede cortado a la mitad entre el salto de página (`.page2-start`), que el texto no se desborde de sus contenedores, y que los colores/semáforos se impriman (no queden en blanco).

### Paso 6 — Publicar como Artifact
Publicar el HTML con el tool de Artifact (favicon 🌐, título corto tipo "Panorama Global y México — [fecha corta]"). Si ya existe una versión previa de esta sesión y se quiere mantener el mismo link, usar `url` en la republicación; si es una fecha nueva, publicar sin `url` (crea un artifact nuevo).

### Paso 7 — Entregar el PDF directamente al usuario
Usar el tool de envío de archivos para mandar el PDF ya generado (no solo el link del artifact) — esto evita que el usuario dependa de "Guardar como PDF" desde su navegador en Windows, que históricamente dio problemas (ver sección 5).

### Paso 8 — Posts de LinkedIn (en el chat, no en el HTML)
Generar siempre dos bloques de texto plano, listos para copiar y pegar:
1. **Post largo divulgativo**: narrativa completa del período cubierto, tono profesional-accesible, cierra con una pregunta abierta a la audiencia.
2. **Post corto con hashtags**: bullets con emoji + cifra clave, cierra con bloque de hashtags (`#Mercados #Inversión #México #BMV #Fed #Petroleo #Geopolitica #Macro #Finanzas` + hashtags específicos del evento de la semana, ej. `#JacksonHole #Ormuz #TasasDeInteres`).

---

## 4. Estructura exacta del documento (secciones obligatorias, en este orden)

1. **Masthead**: etiqueta superior con tipo de corte + fecha + hora (ej. "Corte: Miércoles 16 de septiembre de 2026 · Previo a la decisión de la Fed"), título (H1, gancho editorial del día), subtítulo resumen de 3-4 líneas.
2. **`.summary`** (banda con barras de color a la izquierda): 3-4 párrafos cronológicos — ejemplo de secuencia usada: *fin de semana → día(s) anterior(es) de la semana → hoy → lo que viene*. Cada párrafo con negritas en las cifras/eventos clave.
3. **Grid 1, columna izquierda — "Riesgo Global y Confianza de Mercado"**: 1-2 `impact-block` (crítico/medio/positivo) + 2 `riesgo-card` adicionales.
4. **Grid 1, columna derecha — "Cambios Materiales"** (tabla comparando el día de corte contra el día/semana anterior) + **"Indicadores Económicos Clave"** (tabla de tasas/inflación con semáforo).
   - Si el corte coincide con **cierre de mes**, insertar aquí una tabla adicional "Resumen de [Mes] (Cierre Mensual)" comparando cierre del mes previo vs. cierre del mes actual.
5. **`page2-start`** (fuerza salto de página aquí) — Grid 2, columna izquierda — **"Noticias de Alto Impacto"**: 3 `riesgo-card` con semáforo (rojo/ámbar/gris).
6. **Grid 2, columna derecha — "México 60 Segundos"**: 4 `mx60-card` con ícono/emoji, cifra y una frase.
7. **"Expectativa Resto de Semana" / "Agenda y Riesgos"**: timeline horizontal (`.tl-item`) con 3-4 hitos fechados + 1 ítem de "Riesgo" (escenario de reversión).
8. **`.footer-band`** ("Cierre Ejecutivo"): 4 celdas — Lectura del día/semana, Riesgo principal, Oportunidad potencial, Qué vigilar.
9. **`.legend`**: nota metodológica + disclaimer de no-asesoría (texto fijo, ver abajo).

### Texto fijo de la nota metodológica (adaptar solo la lista de fuentes citadas)
```
Nota metodológica: Este documento es un análisis informativo de riesgos y escenarios
elaborado con fuentes primarias y secundarias verificadas ([listar fuentes usadas ese día]).
No constituye asesoría financiera personalizada ni una recomendación directa de compra o venta.
[Si aplica: "El resultado de [evento] no se había confirmado oficialmente al momento de este
corte — se presenta como expectativa de mercado, no como hecho consumado."]
Toda decisión de inversión debe considerar objetivos, horizonte, tolerancia al riesgo,
liquidez y situación financiera individual.
```

---

## 5. Errores ya cometidos y cómo se corrigieron (no repetir)

| # | Problema observado | Causa raíz | Corrección aplicada |
|---|---|---|---|
| 1 | El usuario no podía "Guardar como PDF" desde su navegador en Windows | Dependía 100% del diálogo de impresión del cliente, con comportamiento inconsistente entre navegadores/SO | Generar el PDF del lado del servidor con Playwright/Chromium headless y enviarlo como archivo directo |
| 2 | El PDF salía en **3 hojas** y el texto quedaba **cortado** a la mitad | Los tamaños de fuente/padding del CSS de impresión eran insuficientes para el volumen real de contenido; además faltaba `break-inside: avoid` en varios contenedores | Reducir tamaños tipográficos en `@media print` de forma iterativa, añadir `break-inside: avoid` a `.impact-block`, `.riesgo-card`, `.mx60-card`, `.tl-item`, `.footer-band`, `.legend`, `.table-wrap`; **contar páginas programáticamente** después de cada ajuste en vez de confiar en "se ve bien" |
| 3 | Ruta de ejecutable de Chromium incorrecta (`/opt/pw-browsers/chromium/...`) | El nombre real de carpeta incluye el número de build (`chromium-1194`) | Verificar con `find /opt/pw-browsers -iname "*chrome*" -type f` antes de asumir la ruta |
| 4 | Conteo de páginas del PDF con regex ingenuo contaba de más | `/Type /Pages` (nodo padre del árbol) se confundía con `/Type /Page` (página individual) | Usar regex que excluya explícitamente la "s" final: `rb'/Type\s*/Page[^s]'` |
| 5 | Presentar un resultado de evento (ej. decisión de la Fed) como hecho confirmado cuando el anuncio oficial aún no ocurría al momento del corte de información | Las búsquedas web devuelven mezcla de notas "preview" (antes del evento) y, a veces, ninguna nota post-evento si el corte de la sesión es anterior a la publicación | Verificar explícitamente si existe nota post-anuncio; si no, presentar como **expectativa de mercado con probabilidad** (ej. "CME FedWatch asigna >90%"), nunca como hecho, y anotarlo en la leyenda final |
| 6 | Intentar acceder a un Google NotebookLM del usuario vía link directo | NotebookLM requiere sesión autenticada de Google; el fetch de la sesión no tiene esas credenciales | Explicar la limitación al usuario y ofrecer alternativas (exportar a Drive, pegar texto/fuentes directamente, o reconstruir con fuentes públicas equivalentes vía WebSearch) |
| 7 | Discrepancias entre fuentes para la misma cifra (ej. USD/MXN con 3 valores distintos el mismo día) | Distintos medios citan spot intradía vs. FIX oficial vs. cierre de distintas horas | Usar el dato oficial (Banxico FIX) como ancla, citar el intradía/spot como contexto adicional, marcar con `~` cuando sea aproximado, y siempre anotar la fuente en `.src` bajo la tabla |
| 8 | Riesgo de desalineación de fecha ("hoy" del usuario vs. reloj del sistema) | El sistema puede reportar una fecha distinta a la que el usuario declara como "hoy" en su instrucción | Priorizar siempre la fecha que el usuario indica explícitamente en su prompt sobre el reloj de sistema; si hay una discrepancia notoria, puede mencionarse brevemente pero sin bloquear el entregable |

---

## 6. Especificación visual (design tokens) — no modificar sin que el usuario lo pida

**Paleta** (variables CSS en `:root`, con variante dark-mode ya resuelta vía `prefers-color-scheme` y `data-theme`):
- `--navy: #152a4e` / `--navy-2: #1f3b6e` — color de marca, headers, bordes de sección.
- `--paper: #eef1f5` — fondo general.
- `--card: #ffffff` — tarjetas y tablas.
- `--green / --amber / --red / --gray-chip` (+ sus `-bg`) — semáforo de impacto/tendencia.
- `--accent: #c99a3f` (dorado) — detalles en el masthead y footer.

**Tipografía** (Google Fonts):
- `Archivo` (700/800) — títulos y `.section-title`.
- `Archivo Narrow` (600/700) — etiquetas uppercase, fechas de timeline, headers de tabla.
- `Inter` (400-700) — cuerpo de texto.

**Layout**: `masthead` con gradiente navy → `.summary` superpuesta (banda blanca con barras de color a la izquierda) → `grid2` (dos columnas) repetido dos veces → `.agenda` (timeline horizontal) → `.footer-band` (navy) → `.legend` (gris claro).

**Impresión**: `@page { size: letter; margin: 10mm 11mm; }` + bloque `@media print` que reduce proporcionalmente toda la tipografía (actualmente ronda 7.8-8.6px base según la densidad de contenido de cada edición) y fuerza el salto de página con la clase `.page2-start` justo antes de "Noticias de Alto Impacto".

---

## 7. Checklist final antes de entregar

- [ ] ¿Toda cifra de mercado tiene fecha y fuente?
- [ ] ¿Se distinguieron hechos confirmados de expectativas/pendientes de confirmación?
- [ ] ¿El PDF generado cuenta exactamente 2 páginas (verificado por regex, no a ojo)?
- [ ] ¿La captura de verificación visual no muestra texto cortado ni desbordado?
- [ ] ¿Se publicó el Artifact HTML con favicon y título correctos?
- [ ] ¿Se envió el PDF como archivo directo al usuario (no solo el link)?
- [ ] ¿Se incluyeron los dos posts de LinkedIn en el chat (largo + corto con hashtags)?
- [ ] ¿La leyenda final incluye el disclaimer de no-asesoría y la lista real de fuentes usadas ese día?
