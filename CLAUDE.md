# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
npm run dev       # dev server at http://localhost:4321
npm run build     # production build → dist/
npm run preview   # preview the production build
```

No linter or test runner is configured.

## Stack

| Layer | Tool | Installed |
|---|---|---|
| Framework | Astro | 6.3.1 |
| Styles | Tailwind CSS v4 | ^4.1 |
| Animations | GSAP + SplitText + CustomEase | ^3.13 |
| 3D / WebGL | Three.js | ^0.184.0 |
| Fonts | DM Sans, DM Mono | Google Fonts |
| Plugins | @tailwindcss/typography, tailwind-scrollbar-hide | — |
| UMD Plugin | ScrambleTextPlugin | public/ScrambleTextPlugin.min.js |

## Architecture

Tres páginas, dos layouts, design tokens compartidos.

```
NoBrandStudio/
├── astro.config.mjs              — Vite plugin para Tailwind v4
├── tsconfig.json                 — strict + alias @/* → src/*
├── package.json
├── CLAUDE.md
├── .astro/                       — caché de tipos (no editar)
├── public/
│   ├── img1.jpg … img12.jpg     — imágenes de clientes del homepage
│   ├── ScrambleTextPlugin.min.js — plugin UMD cargado vía is:inline (no Rollup)
│   └── crt/
│       ├── monitor.glb           — modelo 3D del monitor CRT
│       └── default.jpg           — textura de referencia
└── src/
    ├── images/
    │   ├── work/                 — imágenes del studio (one/two/three.png)
    │   └── projects/             — imágenes del studio (one/two/three.png)
    ├── styles/
    │   └── global.css            — ÚNICA fuente de verdad: tokens, base, custom CSS
    ├── layouts/
    │   ├── Layout.astro          — homepage/404: Preloader + PageTransition + Menu, body class="page-home"
    │   └── StudioLayout.astro    — studio/lab: nav + footer propios de studio
    ├── pages/
    │   ├── index.astro           — homepage /
    │   ├── studio.astro          — studio /studio (Intro + Work + Projects + Education + Speaking)
    │   ├── lab.astro             — lab /lab (Intro + Projects + Education + Speaking)
    │   └── 404.astro             — página de error personalizada con CRT Three.js
    └── components/
        ├── Menu.astro            — nav fija del homepage: logo + scramble + hamburger + OverlayMenu
        ├── OverlayMenu.astro     — menú fullscreen animado (cortinas GSAP + SplitText)
        ├── PageTransition.astro  — animaciones de entrada/salida vía [data-split] attributes
        ├── Preloader.astro       — animación de carga; último frame = showreel-trigger (#showreel-trigger)
        ├── CRTDisplay.astro      — escena Three.js con monitor CRT y shader de efectos
        ├── Footer.astro          — footer del homepage, mix-blend-difference
        ├── ClientsSection.astro  — homepage: lista de clientes + animación GSAP hover
        └── studio/               — todos los componentes del studio (aislados)
            ├── assets/Logo.astro — texto del logo ("No Brand Studio.")
            ├── fundations/
            │   ├── containers/Wrapper.astro       — variant="standard" | "prose"
            │   ├── elements/Button.astro, Text.astro
            │   ├── head/BaseHead.astro, Fonts.astro, Meta.astro, Seo.astro, Favicons.astro
            │   └── icons/ArrowRight.astro, Plus.astro
            ├── global/Navigation.astro — nav del studio (logo · location · links · download)
            │           Footer.astro   — footer del studio
            └── landing/Intro.astro, Work.astro, Projects.astro, Education.astro, Speaking.astro
```

## Design tokens — único punto de cambio

Todo el sistema de diseño vive en `src/styles/global.css` dentro del bloque `@theme`. Cambiar una variable aquí afecta todas las páginas:

```css
@theme {
  /* Tipografía — cambia aquí para retipografiar el sitio completo */
  --font-sans: "DM Sans", sans-serif;   /* fuente principal */
  --font-mono: "DM Mono", monospace;    /* UI del homepage (nav, footer, listas) */

  /* Paleta de acento (verde-teal OKLCH) — studio y elementos de acento */
  --color-accent-50 … --color-accent-950

  /* Escala base / neutros (OKLCH) — tipografía y superficies del studio */
  --color-base-50 … --color-base-950
}
```

## Separación de estilos por página

El homepage requiere texto blanco sobre fondo blanco (efecto `mix-blend-difference`). Estos estilos están acotados bajo `.page-home`, clase aplicada en `Layout.astro`:

```css
/* Solo aplica en / (homepage) y 404 */
.page-home h1       { color: #fff; }
.page-home p,
.page-home a        { color: #fff; text-transform: uppercase; font-family: var(--font-mono); … }
```

El studio usa clases de utilidad Tailwind explícitas (`text-base-900`, `text-base-600`, etc.) que siempre ganan sobre `@layer base` en Tailwind v4.

## Tailwind v4 setup

No hay `tailwind.config.js`. Toda la configuración está en CSS:

```css
@import "tailwindcss";
@plugin "@tailwindcss/typography";
@plugin "tailwind-scrollbar-hide";
@custom-variant dark (&:where(.dark, .dark *));   /* dark mode por clase, no por OS */
@theme { … }
```

El Vite plugin (`@tailwindcss/vite`) está conectado en `astro.config.mjs`. No añadir `@astrojs/tailwind` (es para Tailwind v3).

**Scoped CSS vs Tailwind:** Los bloques `<style>` de Astro tienen mayor especificidad que las utilidades Tailwind. Si se necesita controlar `display` condicionalmente (ej. ocultar en mobile), usar `@media` dentro del `<style>` scoped en lugar de clases responsive (`lg:hidden`). Usar `visibility: hidden` en vez de `display: none` cuando el elemento debe ocupar espacio en el grid.

**Dark mode:** las clases `dark:*` solo se activan cuando existe un ancestro con clase `.dark`. Ningún layout lo añade actualmente.

## Navegación y transiciones

### Layout.astro (homepage + 404)
Incluye automáticamente: `Preloader`, `PageTransition`, `Menu`. Cualquier página que use este layout hereda el sistema de transiciones y el menú hamburguesa.

### PageTransition.astro
Anima elementos con el atributo `data-split="words|chars|lines"`:
- **Entrada:** SplitText lines/chars desde `yPercent: 105` → 0, activado por el evento `preloaderDone` o por `__preloaderDone` en sessionStorage
- **Salida:** SplitText lines/chars hacia `yPercent: -110`, usando el patrón `pending` counter para esperar a que todas las animaciones terminen antes de navegar
- También anima `#showreel-trigger` (fade + translateY) y `#menu-toggle` (fade + translateY)
- Excluye elementos dentro de `#overlay-menu`

### Preloader.astro
- Primera visita: animación completa de carga, al terminar emite `preloaderDone` y guarda `__preloaderDone` en sessionStorage
- Recargas: el preloader se oculta, sólo el último `<img>` queda visible como `#showreel-trigger` con `opacity: 0` para que PageTransition lo anime al entrar
- `lastImg.id = "showreel-trigger"` — este ID es usado por PageTransition y OverlayMenu para las transiciones

### OverlayMenu.astro
- Animación: 4 cortinas negras (`scaleY: 0 → 1`) + clip-path en `.overlay-items`
- Textos animados con SplitText (esperan `document.fonts.ready`)
- Al abrir: oculta `main, section, footer` con delay 0.4s
- Al cerrar: restaura `main, section, footer`
- Al navegar desde el overlay: maneja la salida del `#showreel-trigger` y `#menu-toggle` directamente (sin PageTransition)
- Los links con `href="#"` o `href="#_"` son placeholders; no disparan navegación

### Menu.astro — ScrambleTextPlugin
El plugin es UMD y Rollup no puede analizarlo estáticamente. Se carga como:
```astro
<script is:inline src="/ScrambleTextPlugin.min.js"></script>
```
Y se registra en el script bundleado como:
```ts
gsap.registerPlugin((window as any).ScrambleTextPlugin);
```
**No importar ScrambleTextPlugin vía `import` en ningún archivo — rompe el build de Vercel.**

### Hamburger (Menu.astro)
- Solo visible en mobile (`< 1024px`); oculto en desktop con `@media (min-width: 1024px) { .hamburger { display: none; } }` en el `<style>` scoped
- Clase `.is-open` añadida/removida via JS para animar las barras al icono X
- Empieza con `opacity: 0` y PageTransition lo anima al entrar

## CRTDisplay.astro — página 404

Componente Three.js que renderiza un monitor CRT 3D sobre fondo negro.

**Setup:**
- `camera.position.set(0, 0.15, Math.max(1, 768 / innerWidth))` — se acerca en mobile
- GLTFLoader carga `/crt/monitor.glb`
- Pantalla: `ShapeGeometry` con esquinas redondeadas y UVs manuales
- `displayPlane.position.set(-0.008, 0.005, 0.041)`, `rotation(-0.18, 0, 0)`, `scale(0.28, 0.235, 1)`

**Canvas texture (404):** 512×430px, fondo `#040404`, texto verde `#2bff72`, DM Mono.

**Shader:** chromatic aberration, vignette, scanlines, glitch via `hash()`. Uniforms: `map`, `glitchIntensity`, `time`, `iResolution`, `imageAspect`, `planeAspect`.

**Glitch periódico:** `triggerGlitch()` con `setTimeout(triggerGlitch, 1200 + Math.random() * 3000)`.

**Mouse tracking:** `mouse.x/y` → `gsap.utils.interpolate` → `monitorGroup.rotation.x/y`.

## Homepage — animación GSAP (`ClientsSection.astro`)

**`mouseover`:**
1. Se crea `div.client-img-wrapper` y se añade a `#clients-preview`
2. Su `clip-path` parte de `polygon(50% 50%, 50% 50%, 50% 50%, 50% 50%)` y anima al rectángulo completo (0.5 s, ease "hop")
3. La `<img>` interior: fade in (0.25 s) + scale 1.25 → 1 (1.25 s, ease "hop")

**`mouseout`:** fade out (0.5 s) → `onComplete` elimina el wrapper del DOM.

**Custom ease "hop":**
```
M0,0 C0.071,0.505 0.192,0.726 0.318,0.852 0.45,0.984 0.504,1 1,1
```

## Añadir o cambiar clientes (homepage)

Editar el array `clients` al inicio de `src/components/ClientsSection.astro`. El índice N del array usa `public/imgN.jpg`. Añadir la imagen a `public/` antes de añadir el nombre.

## CSS que no se puede renombrar

`.client-img-wrapper`, `.client-name`, `.clients-list`, `#clients-preview`, `.page-home` — referenciados por JavaScript o por selectores de `global.css`.

`#showreel-trigger`, `#menu-toggle`, `#overlay-menu` — referenciados por `PageTransition.astro` y `OverlayMenu.astro`.
