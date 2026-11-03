---
name: ui-redesign
description: Redesign a Vue 3 application's UI into a modern SaaS-style interface — left vertical sidebar navigation replacing a top nav bar, a design-token system for consistent spacing/color/typography, and polished cards, tables, and states. Use when asked to modernize, restyle, or "SaaS-ify" a Vue 3 frontend, or to move navigation into a sidebar.
---

# Vue 3 SaaS UI Redesign

This skill turns a Vue 3 app with a top navigation bar into a modern SaaS layout: a fixed left sidebar, a slim top bar, a token-based design system, and consistently styled cards, tables, and states. Work through the phases in order.

## Principles

- **Presentation only.** Routes, API calls, reactive state, i18n keys, props, and emitted events stay exactly as they are. If a change requires touching logic, stop and ask.
- **Project rules win.** Read the project's `CLAUDE.md` files first. Existing palette, "no emojis", subagent-delegation, or styling rules override the defaults in this skill.
- **One layer at a time.** Tokens → shell → shared primitives → individual views. Verify the app still runs between layers.
- **Keep shared class names.** Views usually depend on global classes (`.card`, `.badge`, ...). Restyle them in place; never rename them.

## Phase 1: Audit (read-only)

Find and note:

1. **App shell** — usually `src/App.vue` or `src/layouts/*.vue`. Locate the nav markup, header widgets (profile menu, language switcher, notifications), and anything rendered between the header and `<router-view />` (filter bars, banners).
2. **Router** — `src/main.js` or `src/router/index.js`. List every route and its nav label.
3. **Active-state logic** — manual `$route.path === '/x'` comparisons vs. built-in `router-link-active`.
4. **Global styles** — unscoped `<style>` in `App.vue`, `src/assets/*.css`, or `main.css`. List the shared classes and how heavily views use them:
   ```bash
   for c in page-header stats-grid stat-card card card-header badge loading error; do
     printf "%-12s %s\n" "$c" "$(grep -rho "class=\"[^\"]*\b$c\b" src/views src/components | wc -l)"
   done
   ```
5. **Existing palette and fonts** — hex values in use, so the tokens can be derived from them rather than invented.
6. **Must-not-break list** — every nav item, every header widget and the modals it opens, every contextual bar.

## Phase 2: Design tokens

Add CSS custom properties to the global stylesheet, deriving colors from the existing palette:

```css
:root {
  /* Spacing — 4px base */
  --space-1: 4px;  --space-2: 8px;  --space-3: 12px; --space-4: 16px;
  --space-5: 20px; --space-6: 24px; --space-8: 32px; --space-10: 40px; --space-12: 48px;

  /* Radius & elevation */
  --radius-sm: 6px; --radius-md: 8px; --radius-lg: 12px;
  --shadow-sm: 0 1px 2px rgba(15, 23, 42, 0.06);
  --shadow-md: 0 4px 12px rgba(15, 23, 42, 0.08);

  /* Type scale */
  --text-xs: 12px; --text-sm: 13px; --text-base: 14px; --text-lg: 16px;
  --text-xl: 20px; --text-2xl: 24px; --text-3xl: 30px;
  --leading-tight: 1.25; --leading-normal: 1.5;

  /* Semantic color */
  --color-bg: #f8fafc;
  --color-surface: #ffffff;
  --color-surface-muted: #f1f5f9;
  --color-border: #e2e8f0;
  --color-text: #0f172a;
  --color-text-muted: #64748b;
  --color-primary: #2563eb;
  --color-primary-soft: #eff6ff;
  --color-success: #16a34a; --color-success-soft: #f0fdf4;
  --color-warning: #ca8a04; --color-warning-soft: #fefce8;
  --color-danger:  #dc2626; --color-danger-soft:  #fef2f2;
  --color-info:    #2563eb; --color-info-soft:    #eff6ff;

  /* Sidebar (dark rail on light content is the classic SaaS look) */
  --sidebar-bg: #0f172a;
  --sidebar-text: #94a3b8;
  --sidebar-text-active: #ffffff;
  --sidebar-item-hover: rgba(255, 255, 255, 0.06);
  --sidebar-item-active: rgba(255, 255, 255, 0.10);

  /* Layout */
  --sidebar-width: 248px;
  --sidebar-collapsed: 72px;
  --topbar-height: 56px;
  --content-max: 1440px;
  --z-sidebar: 40; --z-topbar: 30; --z-backdrop: 45; --z-modal: 100;
}
```

Then replace hardcoded values in the existing global primitives with tokens. Keep selectors unchanged so views pick up the new look automatically.

## Phase 3: Shell layout

This is the core of the redesign. Replace the top nav with a grid shell.

### Template

```vue
<template>
  <div class="app-shell" :class="{ 'is-collapsed': collapsed, 'is-drawer-open': drawerOpen }">
    <aside class="sidebar" id="app-sidebar">
      <div class="sidebar-brand">
        <span class="brand-mark" aria-hidden="true">A</span>
        <span class="brand-name">{{ t('nav.companyName') }}</span>
      </div>

      <nav class="sidebar-nav" aria-label="Main">
        <router-link
          v-for="item in navItems"
          :key="item.to"
          :to="item.to"
          class="nav-item"
          :exact-active-class="item.exact ? 'is-active' : undefined"
          :active-class="item.exact ? '' : 'is-active'"
          :title="collapsed ? item.label : undefined"
        >
          <NavIcon :name="item.icon" />
          <span class="nav-label">{{ item.label }}</span>
        </router-link>
      </nav>

      <div class="sidebar-footer">
        <!-- existing profile menu moves here -->
        <button class="collapse-toggle" :aria-expanded="!collapsed" @click="toggleCollapsed">
          <NavIcon :name="collapsed ? 'chevron-right' : 'chevron-left'" />
          <span class="nav-label">Collapse</span>
        </button>
      </div>
    </aside>

    <div class="sidebar-backdrop" @click="drawerOpen = false" />

    <div class="shell-main">
      <header class="topbar">
        <button class="menu-toggle" aria-controls="app-sidebar"
                :aria-expanded="drawerOpen" @click="drawerOpen = !drawerOpen">
          <NavIcon name="menu" />
        </button>
        <h1 class="topbar-title">{{ currentTitle }}</h1>
        <div class="topbar-actions"><!-- language switcher, notifications --></div>
      </header>
      <!-- existing contextual bars (e.g. filter bar) go here, inside the content column -->
      <main class="main-content"><router-view /></main>
    </div>
  </div>
</template>
```

Define `navItems` as one array (`{ to, label, icon, exact }`) built from the existing nav labels and i18n keys. Use `exact: true` for `/`. Derive `currentTitle` from the matching nav item for `$route.path`. vue-router adds `aria-current="page"` to the exact-active link automatically.

With more than about six items, group them under small uppercase section labels (for example "Operations", "Finance").

### Script essentials

```js
const STORAGE_KEY = 'sidebar-collapsed'
const readCollapsed = () => { try { return localStorage.getItem(STORAGE_KEY) === '1' } catch { return false } }
const collapsed = ref(readCollapsed())
const drawerOpen = ref(false)

const toggleCollapsed = () => {
  collapsed.value = !collapsed.value
  try { localStorage.setItem(STORAGE_KEY, collapsed.value ? '1' : '0') } catch { /* storage blocked: keep in-memory state */ }
}

// The mobile drawer must close on navigation, or it covers the new page
watch(() => route.path, () => { drawerOpen.value = false })

const onKeydown = (e) => { if (e.key === 'Escape') drawerOpen.value = false }
onMounted(() => window.addEventListener('keydown', onKeydown))
onUnmounted(() => window.removeEventListener('keydown', onKeydown))
```

### Icons

Create a small `components/NavIcon.vue`: a `name` prop mapped to inline SVG path data, 20×20, `stroke="currentColor"`, `stroke-width="1.75"`, `fill="none"`, `aria-hidden="true"`. Do not add an icon library or emojis. Labels stay visible next to icons, except in the collapsed rail where the link's `title` provides them.

### Shell CSS

```css
.app-shell { display: grid; grid-template-columns: var(--sidebar-width) 1fr; min-height: 100vh; background: var(--color-bg); }
.app-shell.is-collapsed { grid-template-columns: var(--sidebar-collapsed) 1fr; }

.sidebar {
  position: sticky; top: 0; height: 100vh; z-index: var(--z-sidebar);
  display: flex; flex-direction: column; gap: var(--space-6);
  padding: var(--space-4) var(--space-3);
  background: var(--sidebar-bg); color: var(--sidebar-text); overflow-y: auto;
}
.sidebar-nav { display: flex; flex-direction: column; gap: var(--space-1); flex: 1; }
.sidebar-footer { border-top: 1px solid rgba(255, 255, 255, 0.08); padding-top: var(--space-3); }

.nav-item {
  position: relative; display: flex; align-items: center; gap: var(--space-3);
  height: 40px; padding: 0 var(--space-3); border-radius: var(--radius-md);
  color: inherit; text-decoration: none; font-size: var(--text-sm); font-weight: 500;
}
.nav-item:hover { background: var(--sidebar-item-hover); color: var(--sidebar-text-active); }
.nav-item.is-active { background: var(--sidebar-item-active); color: var(--sidebar-text-active); }
.nav-item.is-active::before {
  content: ''; position: absolute; left: calc(-1 * var(--space-3)); top: 8px; bottom: 8px;
  width: 3px; border-radius: 0 3px 3px 0; background: var(--color-primary);
}
.nav-item:focus-visible, .collapse-toggle:focus-visible, .menu-toggle:focus-visible {
  outline: 2px solid var(--color-primary); outline-offset: 2px;
}
.is-collapsed .nav-label, .is-collapsed .brand-name { display: none; }
.is-collapsed .nav-item { justify-content: center; padding: 0; }

.shell-main { min-width: 0; display: flex; flex-direction: column; } /* min-width: 0 stops wide tables/charts from blowing out the grid column */
.topbar {
  position: sticky; top: 0; z-index: var(--z-topbar); height: var(--topbar-height);
  display: flex; align-items: center; gap: var(--space-4); padding: 0 var(--space-8);
  background: var(--color-surface); border-bottom: 1px solid var(--color-border);
}
.topbar-title { font-size: var(--text-lg); font-weight: 600; margin: 0; }
.topbar-actions { margin-left: auto; display: flex; align-items: center; gap: var(--space-3); }
.main-content { width: 100%; max-width: var(--content-max); margin: 0 auto; padding: var(--space-8); }

.menu-toggle, .sidebar-backdrop { display: none; }

@media (max-width: 1023px) {
  .app-shell, .app-shell.is-collapsed { grid-template-columns: 1fr; }
  .sidebar {
    position: fixed; inset: 0 auto 0 0; width: var(--sidebar-width);
    transform: translateX(-100%); transition: transform 0.2s ease;
  }
  .is-drawer-open .sidebar { transform: none; }
  .is-drawer-open .sidebar-backdrop {
    display: block; position: fixed; inset: 0; z-index: var(--z-backdrop); background: rgba(15, 23, 42, 0.4);
  }
  .is-drawer-open .sidebar { z-index: calc(var(--z-backdrop) + 1); }
  .menu-toggle { display: inline-flex; }
  .collapse-toggle { display: none; } /* the drawer is always full width */
  .is-collapsed .nav-label, .is-collapsed .brand-name { display: inline; }
  .topbar, .main-content { padding-left: var(--space-4); padding-right: var(--space-4); }
}
```

Delete the old top-nav CSS once the shell works.

## Phase 4: Polishing shared components

Restyle the existing global primitives using tokens:

```css
.page-header { display: flex; align-items: flex-end; justify-content: space-between; gap: var(--space-4); margin-bottom: var(--space-6); }
.page-header h2 { font-size: var(--text-2xl); line-height: var(--leading-tight); margin: 0; }
.page-header p { color: var(--color-text-muted); margin: var(--space-1) 0 0; }

.stats-grid { display: grid; grid-template-columns: repeat(auto-fit, minmax(220px, 1fr)); gap: var(--space-4); margin-bottom: var(--space-6); }
.stat-card { background: var(--color-surface); border: 1px solid var(--color-border); border-radius: var(--radius-lg); padding: var(--space-5); box-shadow: var(--shadow-sm); }
.stat-label { font-size: var(--text-xs); font-weight: 600; text-transform: uppercase; letter-spacing: 0.04em; color: var(--color-text-muted); }
.stat-value { font-size: var(--text-3xl); font-weight: 700; margin-top: var(--space-2); font-variant-numeric: tabular-nums; }

.card { background: var(--color-surface); border: 1px solid var(--color-border); border-radius: var(--radius-lg); box-shadow: var(--shadow-sm); padding: var(--space-6); margin-bottom: var(--space-6); }
.card-header { display: flex; align-items: center; justify-content: space-between; margin-bottom: var(--space-4); }
.card-title { font-size: var(--text-lg); font-weight: 600; }

.table-container { overflow-x: auto; }
th { position: sticky; top: 0; background: var(--color-surface-muted); font-size: var(--text-xs); text-transform: uppercase; letter-spacing: 0.04em; color: var(--color-text-muted); padding: var(--space-3) var(--space-4); text-align: left; }
td { padding: var(--space-3) var(--space-4); border-top: 1px solid var(--color-border); font-size: var(--text-sm); }
tbody tr:hover { background: var(--color-surface-muted); }
td.num, th.num { text-align: right; font-variant-numeric: tabular-nums; }

.badge { display: inline-flex; align-items: center; padding: 2px var(--space-2); border-radius: 999px; font-size: var(--text-xs); font-weight: 600; }
.badge.success { background: var(--color-success-soft); color: var(--color-success); }
.badge.warning { background: var(--color-warning-soft); color: var(--color-warning); }
.badge.danger  { background: var(--color-danger-soft);  color: var(--color-danger); }
.badge.info    { background: var(--color-info-soft);    color: var(--color-info); }
```

**States:**
- **Loading** — replace bare "Loading..." text with skeleton blocks (`.skeleton { background: linear-gradient(90deg, var(--color-surface-muted), var(--color-border), var(--color-surface-muted)); background-size: 200% 100%; animation: shimmer 1.2s infinite; border-radius: var(--radius-sm); }`). Keep the existing `v-if="loading"` conditions.
- **Empty** — a centered muted message with a short hint inside the card.
- **Error** — `--color-danger-soft` background, `--color-danger` text, `--radius-md`, with the original error text.

Map any other existing status variants (for example `.badge.high`, `.badge.increasing`) to the four semantic colors.

## Phase 5: Sweeping the views

For each view in `src/views/`:

1. Replace one-off `margin`, `padding`, `gap`, `border-radius`, and hex colors in scoped styles with tokens.
2. Use one grid gap (`--space-4` or `--space-6`) across all layout grids.
3. Replace ad-hoc headings and containers with the shared `.page-header` and `.card` where the structure already matches.
4. Make custom SVG charts fluid (`width: 100%` plus `viewBox`) because the content column is narrower than before.
5. Do not change script blocks, data bindings, or `v-for` keys.

If the project requires delegating `.vue` edits to a subagent (for example `vue-expert`), do so. Pass along the token list, the shell spec above, and the must-not-break list from Phase 1. Split the views across a few calls rather than sending everything in one.

## Verification checklist

- [ ] `npm run build` succeeds with no new warnings.
- [ ] Every route renders, and exactly one nav item is active on each route. Check `/` especially.
- [ ] Header widgets (profile menu, language switcher) and the modals they open still work.
- [ ] Contextual bars (filters) still drive the data on every view.
- [ ] Widths of 1440, 1024 and 375px: no horizontal page scroll; the drawer opens and closes with the hamburger, the backdrop, Escape, and navigation.
- [ ] Collapse state persists across reloads.
- [ ] Keyboard: Tab reaches every nav item with a visible focus ring.
- [ ] i18n: switch locales and check that labels still translate and nothing overflows the sidebar.
- [ ] Before and after screenshots of each route via Playwright MCP if available; otherwise ask the user to check visually.
- [ ] Run the project's existing test suite.

## Pitfalls

- **`/` is always "active"** with `router-link-active`, because every path starts with `/`. Use the exact-active class for the root route.
- **Grid blow-out** — wide tables or SVGs push the sidebar off-screen unless `.shell-main` has `min-width: 0`.
- **Z-index order** — sidebar < topbar < backdrop < drawer < modals. Existing modals must stay above the drawer.
- **Leftover CSS** — remove the old top-nav rules. Selectors such as `.nav-tabs a.active` can match new markup by accident.
- **Global vs. scoped** — shared primitives must stay unscoped. Moving them into `<style scoped>` silently unstyles every view.
- **Sticky table headers** only work if no ancestor between the table and the scroll container has `overflow: hidden`.
- **Hardcoded labels** — reuse existing i18n keys for nav labels. Don't hardcode new strings in one language.

## Worked example: this repository

| Before (`client/src/App.vue`) | After |
|---|---|
| `header.top-nav` with `.logo` | `.sidebar-brand` at the top of the sidebar |
| `nav.nav-tabs` with six `router-link`s and manual `$route.path` active checks | `navItems` array → `.nav-item` links: Overview (`/`, exact), Inventory, Orders, Finance (`/spending`), Demand Forecast, Reports |
| `<LanguageSwitcher />` in the top nav | `.topbar-actions` |
| `<ProfileMenu />` in the top nav | `.sidebar-footer`; keep its `show-profile-details` / `show-tasks` emits wired to the same modals |
| `<FilterBar />` full width under the header | inside `.shell-main`, between `.topbar` and `.main-content` |
| Global primitives in App.vue's unscoped `<style>` | tokenized in place; class names unchanged |

Notes for this repo:
- The Reports nav label is hardcoded (`Reports`) while the others use `t('nav.*')`. Add a `nav.reports` key to `client/src/locales/en.js` and `ja.js` rather than copying the hardcoded string.
- `Backlog.vue` exists but isn't routed. Don't add it to the sidebar unless asked.
- Project rules apply: slate palette, no emojis in the UI, custom SVG charts, and `.vue` changes go through the `vue-expert` agent.
