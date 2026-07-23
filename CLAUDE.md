# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

`pnchy-demo` is a single-file PWA mockup of the Pnchy app — a community loyalty/punch-card platform for local merchants. The entire UI lives in `index.html`. There is no build step, no framework, and no dependencies to install.

To preview: open `index.html` directly in a browser, or serve it with any static server (e.g. `npx serve .`). On mobile it installs as a PWA via `manifest.json` + `sw.js`.

## Architecture

Everything is in one file (`index.html`):

- **CSS** — CSS custom properties design system (`--cream`, `--sage`, `--blush`, etc.) defined in `:root`. All color tokens are here; never use raw hex in new styles.
- **HTML** — Screens inside `.screens > .screen`, three role apps plus shared entry screens. Only the `.active` screen is shown.
  - **Entry:** `splash` → `role-picker` (3 buttons: Customer / Employee / Business Owner).
  - **Customer:** `home`, `drops`, `qr`, `success`, `leaderboard`, `profile`.
  - **Employee:** `emp-home`, `emp-scan`, `emp-ranks`, `emp-profile`.
  - **Owner (Pro tier):** `own-dashboard`, `own-drops`, `own-scan`, `own-team`, `own-ranks`, `own-profile`.
- **JavaScript** — At the bottom, plain JS handles navigation, map, and QR logic.

**Screen navigation:** `showScreen(name)` swaps `.active` on screens and picks the tab bar. Each screen's role is looked up in `SCREEN_ROLE`; `ROLE_TABBAR` maps the role to one of three tab bars (`tab-bar` customer / `tab-bar-emp` / `tab-bar-own`). `NO_CHROME` screens (`splash`, `role-picker`, `success`) show no tab bar. `enterRole(role)` jumps to a role's first screen; `switchRole()` returns to the picker (wired to every Profile tab). Customer special-cases (`qr` → `generateQR()`/`startQRTimer()`, `home` → `initMap()`) are preserved inside `showScreen`. Calling `switchTab(el, screenName)` delegates to `showScreen`.

**Employee/Owner content** is a static port of the Pnchy MVP (`../pnchy-mvp`) staging screens, restyled onto the demo's tokens. Both roles represent **Bloom Coffee**; business names stay consistent with the demo's 5 merchants + coffee-pod leaderboard. Interactions with no static analog use inline overlays, never `alert()`/`confirm()`: `demoScan(role)` (scan-success flash), `ownerPayDrop()` ($20 payment-success overlay), `copyAccessCode()`, `toggleSwitch()`, `switchAPill()`.

**Map (Explore tab):** Leaflet 1.9.4 + leaflet-heat 0.2.0, loaded from CDN. `initMap()` is called lazily on first `showScreen('home')` and guards against double-init with `mapInitialized`. CartoDB Light tiles, pastel CSS filter (`saturate(0.55) brightness(1.08) hue-rotate(5deg) sepia(0.12)`). Merchant markers use `L.divIcon` with inline HTML. `heatLayer` is toggled with `toggleHeatmap()`.

**Bottom sheet:** `openSheet(type)` populates `#sheet-header`, `#sheet-tags`, and `#sheet-punchcard` dynamically from the `merchants` object, then shows `#sheet-overlay`. Closed by clicking the overlay backdrop.

**QR screen:** `generateQR()` builds a 9×9 fake QR grid. `startQRTimer()` counts down from 58 s and calls `generateQR()` on each cycle. Tapping the QR frame calls `showScreen('success')`.

**Service worker (`sw.js`):** Network-first strategy. Cache name is `pnchy-demo-v2` — bump this string to force all clients to re-fetch after a deploy.

## Design conventions

- **Fonts:** `DM Sans` for all body text; `Fraunces` (italic) for display headings and podium numbers; `Instrument Serif` for the splash tagline and CTA button.
- **Responsive:** `@media (max-width: 480px)` removes the fake phone chrome and fills the viewport using `100dvw` / `100dvh`. `env(safe-area-inset-*)` is applied on the tab bar for iPhone home indicator.
- **Animations:** `fadeUp`, `slideIn`, `slideInLeft`, `countUp`, `pulse`, `float`, `spin`, `shimmer` — all defined in the `<style>` block.
- **Elevated center tab (Stamp):** The QR button floats 20 px above the tab bar using `position: absolute; top: -20px`. Preserve this offset when restyling the tab bar.
