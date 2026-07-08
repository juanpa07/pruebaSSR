# Prueba SSR / SEO — `cc-text`

Comparación de **3 estrategias** de render para el web component `cc-text`, para decidir cuál
hace que un evaluador SEO detecte correctamente los headings (`<h1>`–`<h6>`) antes de fijar la
convención definitiva en las plantillas Twig de Drupal.

## El problema

`cc-text` genera el `<h1>`–`<h6>` (según su prop `fontsize`) **dentro de su Shadow DOM**.
Un evaluador SEO que mira el HTML (view-source) o que no atraviesa el Shadow DOM reporta
**"No H1 tag found"** aunque visualmente el heading se vea.

### Hallazgo importante (slot-heading.html "correcta" pero el evaluador falla)

Al verificar `slot-heading.html` con Backlinko (tools.backlinko.com, powered by Semrush)
el evaluador reporta "No H1" **aunque el `<h1>` SÍ está en el HTML crudo y en el DOM
renderizado** (verificado con curl, regex, Chrome `--dump-dom` y `querySelectorAll` → todos lo ven).

Dos anomalías distinguen ese `<h1>` de uno "normal", y por eso se añadieron 2 páginas de control:
1. El `<h1>` está **envuelto en un custom element** (`<cc-text>`), que un parser SEO ingenuo
   puede saltarse si no reconoce el tag.
2. `cc-text` pone `role="none"` en su host (para accesibilidad). Por spec ARIA eso NO debería
   podar el heading descendiente, pero **algunos evaluadores podan todo el subárbol**.

Las páginas `control.html` y `slot-no-role.html` aíslan cada variable. Evalúalas para saber
cuál rompe a Backlinko:
- Si `control.html` (headings desnudos) también falla → el problema es el evaluador (caché/JS).
- Si `control.html` pasa pero `slot-heading.html` falla → el envoltorio `<cc-text>` es el culpable.
- Si `slot-no-role.html` pasa pero `slot-heading.html` falla → el `role="none"` es el culpable.

## Las 3 variantes (misma URL base, 3 archivos)

| Archivo | Estrategia | `<hN>` en view-source (sin JS) | `<hN>` con JS (Googlebot) |
|---|---|---|---|
| `shadow-dom.html` | **Actual** — `cc-text` genera el heading en Shadow DOM | ❌ 0 | ⚠️ existe pero dentro del shadow root (la mayoría de evaluadores NO lo cuentan) |
| `light-dom.html` | `cc-text-lightdom` — genera el heading en **Light DOM** (`createRenderRoot() { return this }`) | ❌ 0 | ✅ el `<hN>` queda en el DOM de la página |
| `slot-heading.html` | **Patrón IDBLab** — el `<hN>` se escribe explícito en el slot; `cc-text` solo estiliza | ✅ 6 | ✅ 6 |

Sin redundancia en la variante light-dom: escribes `<cc-text-lightdom fontsize="h2">Título</cc-text-lightdom>`
y el componente decide el tag. En la variante slot-heading escribes el tag tú:
`<cc-text fontsize="h2"><h2>Título</h2></cc-text>` (como hace `idb-styled-text` de la competencia).

### Trade-offs

- **shadow-dom**: máxima encapsulación de estilos, peor SEO (heading escondido en el shadow).
- **light-dom**: SEO correcto si el crawler ejecuta JS (Googlebot moderno sí); **pierde la
  encapsulación de estilos del Shadow DOM** (los estilos del `.lit.ts` no aplican, hay que
  depender del CSS global). No aparece en view-source puro.
- **slot-heading (IDBLab)**: **mejor SEO** — el heading está en el HTML crudo, indexable por
  cualquier crawler, con o sin JS. Mantiene la encapsulación de estilos (el shadow estiliza vía
  `::slotted`). Coste: el autor debe escribir el tag semántico, que puede duplicar la intención
  de `fontsize`.

## Contenido del repo

```
pruebaSSR/
├─ shadow-dom.html                 # Variante 1 (actual)
├─ light-dom.html                  # Variante 2 (render en light DOM)
├─ slot-heading.html               # Variante 3 (patrón IDBLab)
├─ index.html                      # = shadow-dom.html (entrada por defecto)
├─ dist/
│  ├─ ebf-components.bundle.js      # Bundle real (cc-text incluido) — usado por variantes 1 y 3
│  ├─ cc-text-lightdom.bundle.js    # Variante experimental light-DOM — usada por variante 2
│  └─ ebf-tailwind.min.css          # CSS global de marca
└─ README.md
```

Los bundles se generaron desde `EBF-Website-StoryBook`:
```bash
npm run build:bundle    # → ebf-components.bundle.js (componente cc-text REAL)
npm run build:styles    # → ebf-tailwind.min.css
# cc-text-lightdom.bundle.js: variante experimental construida con esbuild (createRenderRoot=this).
# El archivo fuente NO quedó en el design system; es solo para esta prueba.
```

## Cómo probar

Servir por HTTP (los ES modules no cargan con `file://`):
```bash
cd pruebaSSR
python3 -m http.server 8080
```
Luego abrir `http://localhost:8080/shadow-dom.html`, `/light-dom.html`, `/slot-heading.html`.

## Cómo evaluar el SEO (el objetivo de la prueba)

Publica el repo (GitHub Pages: Settings → Pages → branch `main` / root) y pasa **las 3 URLs**
por el mismo evaluador SEO, comparando la sección de **Headings / H1-H6**:

1. **View-source (`Ctrl+U`)** — crawler sin JS. Solo `slot-heading.html` mostrará los `<hN>`.
2. **Evaluador con JS** (Google Rich Results / Search Console URL Inspection, Ahrefs, SEMrush,
   Screaming Frog con "JS rendering", seositecheckup, etc.):
   - `shadow-dom.html` → probablemente "No H1 found" (heading en shadow).
   - `light-dom.html` → debería detectar los headings (render en light DOM).
   - `slot-heading.html` → detecta los 6 headings siempre.

## Decisión esperada

| Si el equipo prioriza… | Elegir |
|---|---|
| SEO máximo, compatible con cualquier crawler (incl. sin JS) | **slot-heading** (patrón IDBLab) |
| Mantener `cc-text` como átomo que decide el tag, y basta con Googlebot (JS) | **light-dom** |
| (no recomendado para headings) | shadow-dom actual |

> Recomendación de partida: **slot-heading** replica exactamente lo que hace la competencia
> (`idb-styled-text`) y es el único que garantiza el heading en el HTML crudo. Confirma con el
> evaluador real sobre las 3 URLs antes de cambiar el design system.
