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
| Animations | GSAP + CustomEase | ^3.13 |
| Fonts | DM Sans, DM Mono | Google Fonts |
| Plugins | @tailwindcss/typography, tailwind-scrollbar-hide | — |

## Architecture

Dos páginas, dos layouts, design tokens compartidos.

```
NoBrandStudio/
├── astro.config.mjs              — Vite plugin para Tailwind v4
├── tsconfig.json                 — strict + alias @/* → src/*
├── package.json
├── CLAUDE.md
├── .astro/                       — caché de tipos (no editar)
├── public/
│   └── img1.jpg … img12.jpg     — imágenes de clientes del homepage
└── src/
    ├── images/
    │   ├── work/                 — imágenes del studio (one/two/three.png)
    │   └── projects/             — imágenes del studio (one/two/three.png)
    ├── styles/
    │   └── global.css            — ÚNICA fuente de verdad: tokens, base, custom CSS
    ├── layouts/
    │   ├── Layout.astro          — homepage: Google Fonts, body class="page-home"
    │   └── StudioLayout.astro    — studio: Inter (rsms.me), nav + footer propios
    ├── pages/
    │   ├── index.astro           — homepage /
    │   └── studio.astro          — studio /studio
    └── components/
        ├── Nav.astro             — nav compartida del homepage; tiene link "Studio" → /studio
        │                           lleva estilos propios (<style> scoped) para mix-blend-difference
        ├── Footer.astro          — footer del homepage, mix-blend-difference
        ├── ClientsSection.astro  — homepage: lista de clientes + animación GSAP
        └── studio/               — todos los componentes del studio (aislados)
            ├── assets/Logo.astro — texto del logo ("No Brand Studio.")
            ├── fundations/
            │   ├── containers/Wrapper.astro
            │   ├── elements/Button.astro, Text.astro
            │   ├── head/BaseHead.astro, Fonts.astro, Meta.astro, Seo.astro, Favicons.astro
            │   └── icons/ArrowRight.astro, Plus.astro
            ├── global/Navigation.astro — nav propia del studio (logo → /, location, links de sección)
            │           Footer.astro
            └── landing/Intro.astro, Work.astro, Projects.astro, Education.astro, Speaking.astro
```

## Design tokens — único punto de cambio

Todo el sistema de diseño vive en `src/styles/global.css` dentro del bloque `@theme`. Cambiar una variable aquí afecta ambas páginas:

```css
@theme {
  /* Tipografía — cambia aquí para retipografiar el sitio completo */
  --font-sans: "DM Sans", sans-serif;   /* fuente principal de ambas páginas */
  --font-mono: "DM Mono", monospace;    /* UI del homepage (nav, footer, listas) */

  /* Paleta de acento (verde-teal OKLCH) — studio y elementos de acento */
  --color-accent-50 … --color-accent-950

  /* Escala base / neutros (OKLCH) — tipografía y superficies del studio */
  --color-base-50 … --color-base-950
}
```

## Separación de estilos por página

El homepage requiere texto blanco sobre fondo blanco (efecto `mix-blend-difference`). Para evitar que estos estilos contaminen el studio, están acotados bajo `.page-home`, clase aplicada en `Layout.astro`:

```css
/* Solo aplica en / (homepage) */
.page-home h1       { color: #fff; }
.page-home p,
.page-home a        { color: #fff; text-transform: uppercase; font-family: var(--font-mono); … }
```

El studio usa clases de utilidad Tailwind explícitas (`text-base-900`, `text-base-600`, etc.) que siempre ganan sobre `@layer base` en Tailwind v4, por lo que no necesita overrides adicionales.

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

El alias `@/*` → `src/*` está configurado en `tsconfig.json` y Astro lo propaga automáticamente a Vite.

**Dark mode:** las clases `dark:*` solo se activan cuando existe un ancestro con clase `.dark` en el HTML. Ningún layout lo añade actualmente, por lo que el sitio siempre renderiza en modo claro. Para activar dark mode en una página futura basta con añadir `class="dark"` al elemento `<html>`.

## Navegación entre páginas

`Nav.astro` es el componente compartido del homepage. Incluye un link "Studio" → `/studio`. Sus `<a>` llevan estilos en un `<style>` scoped (blanco, uppercase, DM Mono) para que el efecto `mix-blend-difference` funcione en cualquier fondo sin depender de `.page-home`.

El studio tiene su propia `studio/global/Navigation.astro` con layout de 4 columnas (logo · location · links · download). El logo enlaza a `/` para volver al homepage.

## Homepage — diseño visual

- **Fondo:** blanco (`#fff`)
- **Texto:** blanco (`#fff`) — legible porque `nav`, `footer` y `.clients-list` tienen `mix-blend-mode: difference`, que invierte el color contra lo que hay detrás
- **`h1`:** DM Sans, 3rem / 2rem móvil, weight 500
- **`p`, `a`:** DM Mono, 0.85rem, weight 550, uppercase — acotado a `.page-home` en `@layer base`
- **Breakpoint:** 1000px (custom, no es un preset Tailwind) — en media queries de `global.css`

## Homepage — animación GSAP (`ClientsSection.astro`)

El script corre íntegramente client-side dentro de `<script>` (Astro lo empaqueta como ES module — no necesita `DOMContentLoaded`).

**`mouseover`:**
1. Se crea `div.client-img-wrapper` y se añade a `#clients-preview`
2. Su `clip-path` parte de `polygon(50% 50%, 50% 50%, 50% 50%, 50% 50%)` y anima al rectángulo completo (0.5 s, ease "hop")
3. La `<img>` interior: fade in (0.25 s) + scale 1.25 → 1 (1.25 s, ease "hop")

**`mouseout`:** fade out (0.5 s) → `onComplete` elimina el wrapper del DOM.

**Estado:** `activeClientIndex` (entero, -1 = ninguno) evita animaciones duplicadas.

**Custom ease "hop":**
```
M0,0 C0.071,0.505 0.192,0.726 0.318,0.852 0.45,0.984 0.504,1 1,1
```
Registrado una vez con `CustomEase.create("hop", …)` — reutilizar el nombre `"hop"` en cualquier `gsap.to()` posterior.

## Añadir o cambiar clientes (homepage)

Editar el array `clients` al inicio de [src/components/ClientsSection.astro](src/components/ClientsSection.astro). El índice N del array usa `public/imgN.jpg`. Añadir la imagen a `public/` antes de añadir el nombre.

## Studio — componentes (`src/components/studio/`)

El `Text` component acepta `tag` y `variant` (escala tipográfica de `displayLG` a `textXS`). El `Wrapper` acepta `variant="standard"`. Todos los componentes de sección importan desde `@/components/studio/…`.

Las imágenes del studio están en `src/images/` (procesadas por `astro:assets` con optimización automática a WebP).

## CSS que no se puede renombrar

`.client-img-wrapper`, `.client-name`, `.clients-list`, `#clients-preview`, `.page-home` — referenciados por JavaScript o por selectores de `global.css`. Renombrarlos sin actualizar ambos archivos romperá la animación o los estilos del homepage.
