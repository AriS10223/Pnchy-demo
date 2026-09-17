# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

`pnchy-demo` is a single-file PWA mockup of the Pnchy app — a community loyalty/punch-card platform for local merchants. The entire UI lives in `index.html`. There is no build step, no framework, and no dependencies to install.

To preview: open `index.html` directly in a browser, or serve it with any static server (e.g. `npx serve .`). On mobile it installs as a PWA via `manifest.json` + `sw.js`.

Deploys live via GitHub Pages straight off this repo (`CNAME` → `pnchy.store`), with no build/CI step — a push to `main` is an instant live deploy, so the `sw.js` `CACHE_NAME` version bump (see below) isn't just a local-testing nicety, it's what forces real installed-PWA users to pick up the change.

## Architecture

Everything is in one file (`index.html`): static markup for every screen, a CSS design-token system, and a plain-JS layer underneath that owns all state — there is no framework and nothing renders itself; every dynamic surface is repainted by a named render function called from state-changing code or from `showScreen()` on entry.

- **CSS** — CSS custom properties design system (`--cream`, `--sage`, `--blush`, etc.) defined in `:root`. All color tokens are here; never use raw hex in new styles.
- **HTML** — Screens inside `.screens > .screen`, three role apps plus shared entry screens. Only the `.active` screen is shown.
  - **Entry:** `splash` → `role-picker` (3 buttons: Customer / Employee / Business Owner).
  - **Customer:** `home`, `drops`, `drop-detail`, `passes`, `pass-qr`, `qr`, `success`, `leaderboard`, `profile`.
  - **Employee:** `emp-home`, `emp-scan`, `emp-drop`, `emp-ranks`, `emp-profile`.
  - **Owner:** `own-dashboard`, `own-drops`, `own-drop-config`, `own-drop-analytics`, `own-scan`, `own-team`, `own-ranks`, `own-profile`, plus the subscription-tier screens `own-plans`, `own-pnch-card`, `own-peak-hours`, `own-lapsed`, `own-ai-insights`, `own-campaigns` (see "Subscription tiers" below).
- **JavaScript** — At the bottom, one `<script>` block holds: navigation (`showScreen`/`SCREEN_ROLE`), map/QR logic, the canonical `MERCHANTS` data + punch-write path, and the Density Drop simulation engine (`SIM`/`WORLD`). See the subsections below for the state layer and engine — they're the parts most likely to matter for a future change.

### Screen navigation

`showScreen(name)` swaps `.active` on screens and picks the tab bar. Each screen's role is looked up in `SCREEN_ROLE`; `ROLE_TABBAR` maps the role to one of three tab bars (`tab-bar` customer / `tab-bar-emp` / `tab-bar-own`). `NO_CHROME` screens (`splash`, `role-picker`, `success`, `pass-qr`) show no tab bar. `enterRole(role)` jumps to a role's first screen; `switchRole()` returns to the picker (wired to every Profile tab). `currentScreenName` tracks the active screen so tick-driven code (the go-live toast) can avoid popping over a screen the tester is already on. Calling `switchTab(el, screenName)` delegates to `showScreen`.

### Employee/Owner content

A static port of the Pnchy MVP (`../pnchy-mvp`) staging screens, restyled onto the demo's tokens. Both roles represent **Bloom Coffee**; business names stay consistent with the demo's 5 merchants + coffee-pod leaderboard, all anchored to **State College, PA** addresses (see the `address` fields on `MERCHANTS` and the map-center comment near `initMap()`) — match that geography if you add a merchant or move the map center. Interactions with no static analog use inline overlays, never `alert()`/`confirm()`: `demoScan(role)` (scan-success flash), `ownerPayDrop()` ($19.99 payment-success overlay — and a real state change, see "Subscription tiers" below), `copyAccessCode()`, `toggleSwitch()`, `switchAPill()`.

Most of `emp-home`'s numbers are static illustrative content, but `SCANS TODAY` (`WORLD.empScansToday`) and `Recent Pnches` (`WORLD.recentPnches`, capped at `EMP_RECENT_PNCHES_MAX`) are real state — a stamp scan via `demoScan('emp')` increments/prepends them and `renderEmpHomeActivity()` repaints both, reset by `resetDemo()`.

### Ranks / leaderboard design system

All three roles' ranks screens (customer `leaderboard`, `emp-ranks`, `own-ranks`) share one visual system rather than each having its own: the customer leaderboard's own class family (`.lb-header`/`.lb-category-scroll`/`.lb-top3`/`.lb-podium-*`/`.lb-list`/`.lb-row`). A single `buildStepPodiumHTML()` builds the stepped 3-place podium for all three — `renderLbList()`, `renderEmpRankList()`, and `renderOwnRankList()` each shape their own top-3 rows (`{ icon, name, bg, me, scoreText }`) and call it. A `.me`-flagged row gets a gold ring (podium) or a "You" tag (list row) — the one piece of UI the customer screen never needs but employee/owner do. Don't reintroduce a separate podium/list style for emp/own ranks; change the shared `.lb-*` classes and all three screens move together.

### Map (Explore tab)

Leaflet 1.9.4 + leaflet-heat 0.2.0, loaded from CDN. `initMap()` is called lazily on first `showScreen('home')` and guards against double-init with `mapInitialized`. CartoDB Light tiles, pastel CSS filter (`saturate(0.55) brightness(1.08) hue-rotate(5deg) sepia(0.12)`). Merchant markers use `L.divIcon` with inline HTML, warmed (`pin-warm-*`) or made live while a Density Drop is building/live at that merchant. `heatLayer` is toggled with `toggleHeatmap()`.

### Bottom sheet

`openSheet(type)` populates `#sheet-header`, `#sheet-tags`, and `#sheet-punchcard` dynamically from `MERCHANTS`, then shows `#sheet-overlay`. Closed by clicking the overlay backdrop.

### QR screen

`generateQR()` builds a 9×9 fake QR grid. `startQRTimer()` counts down from 58 s and calls `generateQR()` on each cycle. Tapping the QR frame calls `showScreen('success')`.

### Merchant data and punches

`MERCHANTS` is the single canonical merchant data source (id, name, icon, category, punches/total, etc.) — it replaced an earlier split between two separately-maintained merchant objects that could disagree with each other, so `MERCHANTS` (plus its `MERCHANTS_INITIAL` deep-clone snapshot, used by `resetDemo()`) is the only merchant data to ever touch. `addPunch(key, n)` is the one place punch counts are incremented; it writes to `MERCHANTS[key].punches` and then calls `renderPunchSurfaces(key)`, which repaints every surface that shows a punch count — bottom sheet, profile loyalty cards, QR screen, success screen, map pins — from `MERCHANTS`. Never hardcode a punch count anywhere; always go through `addPunch`/`renderPunchSurfaces`.

### Density Drop engine

A compressed-clock simulation drives a Density Drop through building → live → expired. `SIM.nowMin` is the sim clock; `TICK_MIN`/`TICK_MS` compress it so **1 wall-second = 1 simulated minute**, meaning real MVP numbers (a 12-person threshold, a 60-minute window, a fixed dwell time) are used directly, unmodified — only the clock is fast. `WORLD.drop` is `null` until `armDrop()` creates one from `WORLD.ownerConfig` (guarded by the subscription-tier cap — see below).

The trigger is **passes issued, not raw foot traffic**: `armDrop()` builds `ambientQueue` via `buildAmbientQueue()`, which expands the fixed `BUILD_ARRIVALS` bucket-shape into individual simulated people, each with an `arrivedAtSim` and a `dwellCompleteAtSim` (`arrivedAtSim + DWELL_REQUIRED_MIN` — dwell time is fixed/Pnchy-set, never randomized per person). `advanceDrop()` (called every tick by `clockTick()`) marks each ambient person's pass `issued` once their dwell completes, incrementing `d.passesIssued`; `d.currentDensity` (raw headcount present) is tracked separately as a flavor stat and Discovery Lift input only — it never drives state transitions. Once `d.passesIssued` reaches `configSnapshot.threshold`, the **Bell rings**: state flips to `live`, `d.liveAtSim`/`d.liveEndsAtSim` are set, and Redemption opens as one **shared window** for every pass-holder (`isRedemptionOpen()`), not a personal per-pass countdown. A **grandfather rule** lets anyone already mid-dwell when the Bell rings keep counting and still receive a pass once their dwell completes, as long as they *arrived* before the Bell (checked in the `live` branch's ambient-trickle loop). If the configured time limit elapses before enough passes are issued, the drop **fizzles** (`state = 'expired'`, `liveAtSim` stays `null`) instead of going live — the one consolation-stamp path handles this for the tracked customer.

Personal dwell (`WORLD.drop.dwell`, `dwellProgress()`) is an independent rail from the ambient queue — it tracks how long "you" specifically have been dwelling. `issueMyPassIfEligible()` issues your own pass the moment you finish dwelling, whether that's during Gathering (before the Bell) or as a grandfathered late finisher during Redemption; either way it increments `d.passesIssued` too. `fillFraction(instance)` (`passesIssued / threshold`) is the one formula that drives every progress meter across every state. Reward passes are redeemed only through a real `demoScan('emp')` call while `isRedemptionOpen()` is true. Employees can pause a drop (auto-resumes after its window) via the emp-drop controls — pausing freezes the ambient queue's issuance, not the clock. `WORLD.history` accumulates since-launch funnel/analytics counters read by the owner drop-analytics screen, including the discovery-lift comparison against `WORLD.baselineDiscovery`. The full state-machine spec (with worked examples) lives in `../density-drop-spec.md`, one directory up — keep it in sync with `advanceDrop()`/`armDrop()` if you change the trigger, grandfather, or redemption rules.

**Hard rule: at most two recurring `setInterval`s may ever exist in this file** — `qrInterval` (QR countdown) and `simInterval` (this engine's clock, via `startClock()`/`stopClock()`). Every other displayed value — countdowns, meters, chips, funnels — must be *derived* from current state (`SIM.nowMin`, `WORLD.drop`, `MERCHANTS`, etc.) at render/tick time. Never give a new feature its own timer.

`renderDropSurfaces()` is the single entry point that keeps every drop-aware screen in sync with `WORLD.drop`/`WORLD.history` — it calls one render function per screen/zone (owner drop strip, owner analytics, map pins, customer drops list/detail/pass wallet/pass QR, employee drop card/controls, the home-screen drop alert). It's called on every clock tick and again from the relevant `showScreen()` transitions, so a screen is always correct whether the tester arrives mid-tick or navigates in from cold.

### Subscription tiers

`PLAN_TIERS` (`starter`/`standard`/`pro`, ordered by `PLAN_TIER_ORDER`) is the single source of truth for what each plan unlocks — cadence (`densityDropsPerCycle`), team size (`employeeCap`), leaderboard breadth (`leaderboardPeriods`), and feature flags (`peakHourAnalytics`, `lapsedCustomerNotif`, `aiInsights`, `sampleCampaigns`). `WORLD.planTier` (default `'pro'`, so the demo's out-of-box behavior is unchanged unless a tester visits Compare Plans) selects the active one via `currentPlan()`. Gating is **functional, not just cosmetic**: every gated screen/section calls `currentPlan()` and either renders real content or `buildLockedStateHTML(requiredTierKey, featureLabel)` — the one shared "🔒 needs \<tier\>" upsell card, which always routes to `own-plans`.

Density Drop cadence is tracked as a simple usage counter, not a fake calendar: `WORLD.dropsUsedThisCycle` (incremented by `armDrop()`) against `dropAllowanceThisCycle()` (`currentPlan().densityDropsPerCycle + WORLD.extraDropsPurchased`); `dropsRemainingThisCycle()` is what every cap check (`armDrop()`, `confirmForecast()`, the drop circle's lock state, the Activation Forecast) reads. `renderOwnDropsCycleGate()` keeps the own-drops screen's cycle indicator, extra-drop paywall (`ownerPayDrop()`, `$19.99`, real state: increments `WORLD.extraDropsPurchased`), and drop-circle lock in sync — call it (or `renderDropSurfaces()`, which includes it) after anything that changes the cap or the plan. `renderOwnTeamGate()` and `renderOwnRanksGate()` apply the same pattern to `own-team` (employee cap) and `own-ranks` (leaderboard period buttons); `renderOwnDashFeatureNav()` toggles the 🔒 badges on own-dashboard's feature nav-rows. **Every plan-dependent surface must be re-derived on plan change** — `selectPlanTier()` and `resetDemo()` are the two places `WORLD.planTier` changes, and both call the full set of gate-render functions plus `renderPlanBadges()`; if you add a new gated surface, wire its render call into both.

`screen-own-plans` (Compare Plans) is reached from "Manage plan →" (own-profile, own-dashboard's plan badge) or from any `buildLockedStateHTML` upsell. `comparePlansViewingTier` (browsing state, independent of `WORLD.planTier`) resets to the active plan every time the screen is entered; `selectPlanTier()` commits the viewed tab as the new `WORLD.planTier`. The five feature screens (`own-pnch-card`, `own-peak-hours`, `own-lapsed`, `own-ai-insights`, `own-campaigns`) each have one render function (`renderPnchCardScreen()`, etc.) called from `showScreen()`'s dispatch table, matching the demo's existing per-screen render-on-entry convention; `own-pnch-card` is the one screen with no plan gate (every tier gets a Pnch Card) but is cycle-gated instead via `WORLD.pnchCardEditedThisCycle`, writing straight to `MERCHANTS.coffee.total`/`.reward` — the one canonical merchant record — rather than any separate config object.

### Reset and session-only state

`resetDemo()` is the in-memory reset — it restores `MERCHANTS` from `MERCHANTS_INITIAL`, stops the clock and wipes `WORLD.drop`/`WORLD.history`/`WORLD.ownerConfig` back to their shipped defaults, resets the subscription-tier state (`WORLD.planTier` back to `'pro'`, cycle/purchase/reconfigure counters to zero) and re-derives every gate render from it, closes any open overlay/sheet, and re-arms the one-time coach marks (`WORLD.coach`). Coach marks are keyed by a shared `COACH_LANDING` lookup (screen name → `{ flag, el }`) and shown once per key per session by `maybeShowCoachMark(key)`. `showScreen()` calls it unconditionally on every navigation, which is enough to cover four of the five keys (`home`/`emp-home`/`own-dashboard`/`leaderboard`, dismissed via `dismissCoachMark(role)` for the three role ones); the fifth (`sheet`, the punch-card-sheet explainer) isn't a real screen, so `openSheet()` calls `maybeShowCoachMark('sheet')` directly, dismissed via `dismissCoachMarkByKey('sheet')`. **Never use `localStorage`/`sessionStorage` anywhere in this file** — every piece of state (`MERCHANTS`, `WORLD`, `SIM`, `reviewState`, etc.) is in-memory only and this matters more now than it used to, given how much of the app is state-driven rather than static markup.

### Wiring convention

Every interactive-looking element either does something real or has had `cursor:pointer` (and any hover affordance) deliberately removed — there should be no dead buttons that merely look clickable. Follow the same rule for any new element: wire it to a real handler, or de-affordance it.

**Service worker (`sw.js`):** Network-first strategy. `CACHE_NAME` is a version-suffixed string (`pnchy-demo-vN`) — bump the number on every deploy to force all clients to re-fetch.

## Design conventions

- **Fonts:** `DM Sans` for all body text; `Fraunces` (italic) for display headings and podium numbers; `Instrument Serif` for the splash tagline and CTA button.
- **Responsive:** `@media (max-width: 480px)` removes the fake phone chrome and fills the viewport using `100dvw` / `100dvh`. `env(safe-area-inset-bottom, 0px)` is baked into the `--nav-clear` token and the tab bar's `bottom` offset (with a `0px` fallback so an unsupported `env()` can't silently zero out the whole `calc()`), so no separate media-query override is needed for the home-indicator inset.
- **Animations:** `fadeUp`, `slideIn`, `slideInLeft`, `countUp`, `pulse`, `ripple`, `float`, `spin`, `shimmer`, plus the subtle CTA pulses `ctaPulseNeutral`/`ctaPulseGreen` — all defined in the `<style>` block. CTA pulses are opacity/box-shadow only (never scale/transform) and are applied via existing conditional classes (`.a-drop-btn.locked`, `.dd-pass-btn`) so they need no extra JS to turn off.
- **Tab bar:** a floating glass pill (`position: absolute`, centered, `border-radius: 999px`, warm-cream `backdrop-filter: blur()`), not a full-width bar — it hugs its own content instead of spanning the screen, so its width differs per role (customer 5 items, employee 4, owner 6). Every `.tab-item` is a direct flex child in reading order; there is no half-wrapper split around the center Stamp/Scan button any more; do not reintroduce one; it is a normal item, distinguished only by a permanent accent-green stroke. Inactive items are icon-only; the active item expands into a labelled chip (`.tab-item.active`), and `updateTabHighlight()` (JS) does pure class/`aria-selected` toggling — all the styling lives in CSS now, so don't reintroduce inline style writes there. Screens with no tab of their own (e.g. `drop-detail`, `own-drop-config`) borrow their parent section's chip via the `PARENT_TAB` map. Because the pill's width animates with the active label and it's centered via `translateX(-50%)`, every icon shifts horizontally on tab switches — this is an accepted trade-off of the hug-content design, not a bug. Scrollable content clears the pill via the shared `--nav-clear` token on `.scroll-content`; any new bottom-anchored element on a screen with tab-bar chrome must use `bottom: var(--nav-clear)` (or its own audited offset) instead of a bare pixel value.
