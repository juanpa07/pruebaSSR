# Prueba SSR / SEO — `cc-text` (Shadow DOM vs Light DOM)

Experimento para responder una pregunta concreta antes de escribir las plantillas Twig de Drupal:

> Cuando uso el web component `cc-text` así:
> ```html
> <cc-text color="graphite-deep-main" fontsize="h2" ...>This is a Title</cc-text>
> ```
> …¿un evaluador de SEO detecta que hay un elemento `<h2>` en la página?
> ¿O me toca escribir el heading explícito dentro del slot?
> ```html
> <cc-text color="graphite-deep-main" fontsize="h2" ...><h2>This is a Title</h2></cc-text>
> ```

## Contexto técnico (por qué importa)

`cc-text` es un componente Lit. Cuando le pasas `fontsize="h2"`, genera un `<h2>` **dentro de su Shadow DOM** y proyecta tu contenido con un `<slot>`. Es decir:

- El **texto** ("This is a Title") vive en el **Light DOM** → está en el HTML plano, es indexable por cualquier crawler.
- El **elemento `<h2>`** vive en el **Shadow DOM** → lo crea JavaScript al hidratar el componente.

Esto genera dos preguntas distintas de SEO:

| Qué ve el crawler | Forma 1: `<cc-text>Título</cc-text>` | Forma 2: `<cc-text><h2>Título</h2></cc-text>` |
|---|---|---|
| **Texto del título** | ✅ en Light DOM | ✅ en Light DOM |
| **Elemento `<h2>` semántico** | ⚠️ solo en Shadow DOM | ✅ en Light DOM |

- **Crawler sin JS (view-source):** no ejecuta el componente. En la Forma 1 solo verá `<cc-text>Título</cc-text>` — el texto sí, pero **ningún `<h2>`**. En la Forma 2 verá el `<h2>` tal cual escrito.
- **Crawler con JS (Googlebot moderno):** ejecuta el componente y el `<h2>` existe en el DOM — **pero dentro del shadow root**. La mayoría de evaluadores SEO y `document.querySelectorAll('h2')` **no atraviesan el Shadow DOM**, así que reportan "0 h2" en la Forma 1.

**Conclusión anticipada (a confirmar con la prueba):** si el evaluador SEO que uses cuenta headings desde el Light DOM (lo habitual), necesitarás la **Forma 2** (`<h2>` explícito en el slot) para que el heading semántico cuente. La Forma 1 garantiza el texto indexable pero no expone el nivel de heading fuera del shadow.

## Contenido de este repo

```
pruebaSSR/
├─ index.html                      # La página de prueba
├─ dist/
│  ├─ ebf-components.bundle.js     # Bundle real de los componentes (cc-text incluido)
│  └─ ebf-tailwind.min.css         # CSS global de marca (tokens @theme + utilidades globales)
└─ README.md
```

Ambos artefactos se generaron desde el proyecto `EBF-Website-StoryBook`:

```bash
npm run build:bundle   # → dist/ebf-components.bundle.js (ESM, 64 componentes, ~960 KB)
npm run build:styles   # → dist/ebf-tailwind.min.css   (CSS global, ~32 KB)
```

Son los artefactos **reales**, no una maqueta — los mismos que se usarán en Drupal.
El CSS global se carga con `<link>` en el `<head>`; el bundle JS con `<script type="module">`.
Nota: cada componente ya trae su propio CSS embebido en el Shadow DOM (vía su `.lit.ts`),
así que `cc-text` se ve estilizado aunque el CSS global es necesario para las clases
globales usadas dentro de los slots (p. ej. `text-accent-main`) y el contenido fuera del shadow.

## Qué contiene `index.html`

1. **Panel de diagnóstico en vivo** — cuenta `<h1>`–`<h6>` de dos formas: solo Light DOM (lo que ve un crawler típico) vs. recorriendo también los Shadow DOM (crawler avanzado). Se ejecuta con JS ya cargado.
2. **Sección A — el experimento**: las dos formas (slot plano vs `<h2>` explícito) lado a lado.
3. **Sección B — todas las variaciones** de cada propiedad de `cc-text` (`fontsize` ×20, `color` ×85, `fontfamily` ×10, `weight` ×8, `align` ×5, `fontstyle` ×2, `texttransform` ×4), generadas iterando los enums reales del componente.

## Cómo probar localmente

Los ES modules requieren servirse por HTTP (no `file://`). Con cualquiera de estos:

```bash
cd pruebaSSR
python3 -m http.server 8080        # → http://localhost:8080
# o
npx serve .
```

## Cómo publicar (GitHub Pages)

1. Crea el repo y sube estos archivos:
   ```bash
   cd /Users/juanpa/Git/pruebaSSR
   git init && git add . && git commit -m "Prueba SSR/SEO cc-text"
   git branch -M main
   git remote add origin git@github.com:<tu-usuario>/pruebaSSR.git
   git push -u origin main
   ```
2. En GitHub → **Settings → Pages** → Source: `Deploy from a branch` → Branch: `main` / `/ (root)` → Save.
3. Tu URL será `https://<tu-usuario>.github.io/pruebaSSR/`.

## Cómo ejecutar la prueba SEO (los 3 métodos que importan)

Una vez publicado, evalúa la **misma URL** de las 3 formas y compara:

### 1. View-source (crawler sin JS) — el caso más estricto
- Abre la página y haz `Ctrl+U` (ver código fuente).
- Busca `<h2>`: **no aparecerá** para la Forma 1 (el heading lo pone JS). El **texto** de la Forma 1 sí aparece dentro de `<cc-text>`.
- Para la Forma 2 verás el `<h2>` literal.
- ➡️ *Si tu SEO depende de crawlers sin JS, la Forma 2 es obligatoria.*

### 2. Evaluador SEO externo (Googlebot-like, con JS)
Prueba la URL publicada en alguno de estos y mira la sección "Encabezados / Headings (H1, H2…)":
- Google Rich Results Test / URL Inspection (Search Console) — renderiza con JS.
- Ahrefs, SEMrush, Screaming Frog (activa "JavaScript rendering"), Sitebulb, seositecheckup.com.
- ➡️ *La mayoría cuenta headings del Light DOM: la Forma 1 probablemente marque 0 h2, la Forma 2 los detectará.*

### 3. Panel de diagnóstico de esta misma página (referencia rápida)
- La tabla superior muestra el conteo **Light DOM** vs **Incl. Shadow DOM**.
- Es la demostración local del mismo principio que aplicará el evaluador externo.

## Cómo interpretar el resultado

| Resultado observado | Qué significa | Acción en Twig |
|---|---|---|
| El evaluador cuenta el h2 de la **Forma 1** | Atraviesa Shadow DOM (raro) | Puedes usar slot plano |
| El evaluador **NO** cuenta el h2 de la Forma 1 pero sí el de la **Forma 2** | Solo lee Light DOM (lo habitual) | **Usa `<h2>` explícito** dentro de `cc-text` |
| Ningún evaluador ve el texto | (No debería pasar) | Revisar que el contenido esté slotted, no en prop |

> **Hipótesis a validar:** para máxima compatibilidad SEO, escribir el heading semántico explícito en el slot:
> ```html
> <cc-text fontsize="h2" color="graphite-deep-main"><h2>{{ title }}</h2></cc-text>
> ```
> El `cc-text` seguiría aplicando estilos vía `::slotted`, y el `<h2>` quedaría en el Light DOM (indexable por todos).
> Confirma con los 3 métodos de arriba antes de decidir la convención definitiva.
