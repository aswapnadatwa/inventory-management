---
name: saas-redesign
description: This skill should be used when the user asks to "redesign the UI", "modernize the interface", "add a sidebar nav", "convert to SaaS style", "replace the top nav with a sidebar", "make it look like a SaaS app", or discusses layout overhaul, vertical navigation, or professional dashboard aesthetics for a Vue 3 application.
version: 1.0.0
---

# SaaS UI Redesign Skill

Transforms a Vue 3 app from a top-navigation-bar layout into a modern SaaS-style interface with a vertical sidebar, consistent spacing, and a polished professional look.

## Overview

This skill guides a full layout restructure of `App.vue` and the introduction of a `Sidebar.vue` component. The goal is a two-column shell: a fixed-width left sidebar for navigation + branding, and a flex-grow right panel containing the filter bar and `<router-view>`.

No view files (Dashboard.vue, Inventory.vue, etc.) should need structural changes — only global layout, global CSS, and the sidebar component are touched.

---

## When This Skill Applies

- User wants to replace the `.top-nav` / `.nav-tabs` pattern with a left sidebar
- User wants a "SaaS dashboard" aesthetic (Notion, Linear, Vercel, Stripe style)
- User asks for consistent spacing, card polish, or professional look across the whole app
- User is dissatisfied with the current horizontal tab navigation

---

## Step-by-Step Implementation

### Step 1 — Delegate to vue-expert

**MANDATORY**: Any creation or significant modification of `.vue` files must be delegated to the `vue-expert` subagent. Do not write `.vue` files directly in the main conversation.

Spawn `vue-expert` with the full context from this skill. Pass:
- The current `App.vue` content
- The route list from `main.js`
- The design tokens (colors, spacing) below
- The exact template structure to produce

---

### Step 2 — New App Shell Structure

Replace the current vertical flex layout (`top-nav` → `FilterBar` → `main-content`) with a horizontal flex layout:

```
.app  (display: flex; height: 100vh; overflow: hidden)
├── <Sidebar />            ← fixed 240px left column
└── .app-body              ← flex: 1; overflow-y: auto; display: flex; flex-direction: column
    ├── <FilterBar />       ← sticky header within the scrollable column
    └── <main.main-content> ← router-view lives here
```

**Key layout rules:**
- `html, body` must be `height: 100%; overflow: hidden` — scroll happens only inside `.app-body`
- Sidebar is `position: sticky; top: 0; height: 100vh; overflow-y: auto` — scrolls independently if nav grows long
- `.app-body` gets `overflow-y: auto` so page content scrolls without moving the sidebar

---

### Step 3 — Sidebar Component (`client/src/components/Sidebar.vue`)

Create `Sidebar.vue` as a new component. It replaces `<nav class="nav-tabs">` entirely.

**Template structure:**
```vue
<template>
  <aside class="sidebar">
    <div class="sidebar-logo">
      <span class="sidebar-logo-mark">F</span>
      <div class="sidebar-logo-text">
        <span class="sidebar-logo-name">FactoryOS</span>
        <span class="sidebar-logo-sub">Inventory Platform</span>
      </div>
    </div>

    <nav class="sidebar-nav">
      <div class="nav-section-label">Workspace</div>
      <router-link v-for="item in navItems" :key="item.to" :to="item.to" class="sidebar-link">
        <span class="sidebar-icon" v-html="item.icon" />
        <span>{{ item.label }}</span>
      </router-link>
    </nav>

    <div class="sidebar-footer">
      <!-- slot for ProfileMenu and LanguageSwitcher -->
      <slot name="footer" />
    </div>
  </aside>
</template>
```

**Nav items data** (defined in `setup()`):
```javascript
const navItems = [
  { to: '/',           label: 'Overview',         icon: svgDashboard },
  { to: '/inventory',  label: 'Inventory',        icon: svgBox },
  { to: '/orders',     label: 'Orders',           icon: svgClipboard },
  { to: '/spending',   label: 'Finance',          icon: svgChart },
  { to: '/demand',     label: 'Demand Forecast',  icon: svgTrend },
  { to: '/reports',    label: 'Reports',          icon: svgDocument },
  { to: '/restocking', label: 'Restocking',       icon: svgRefresh },
]
```

Use simple inline SVG strings for icons — no icon library dependency. 16×16 viewBox, stroke-based, `currentColor`.

**Active state**: Use `router-link-active` class (Vue Router sets it automatically). Style it with accent background + text color change.

---

### Step 4 — Design Tokens

Use these exact values throughout all new CSS. Do not introduce new colors.

```css
/* Layout */
--sidebar-width: 240px;
--sidebar-bg: #0f172a;        /* slate-900 */
--sidebar-text: #94a3b8;      /* slate-400 */
--sidebar-text-active: #f1f5f9; /* slate-100 */
--sidebar-active-bg: rgba(255,255,255,0.08);
--sidebar-hover-bg: rgba(255,255,255,0.05);
--sidebar-border: rgba(255,255,255,0.06);

/* Logo mark */
--logo-mark-bg: #2563eb;      /* blue-600 */
--logo-mark-color: #ffffff;

/* Body / content area */
--body-bg: #f8fafc;           /* slate-50 — existing body background */
--content-padding: 1.5rem 2rem;

/* Accent */
--accent: #2563eb;            /* blue-600 — consistent with existing .active color */
```

---

### Step 5 — Sidebar CSS (inside `Sidebar.vue` `<style scoped>`)

```css
.sidebar {
  width: var(--sidebar-width);
  min-width: var(--sidebar-width);
  background: var(--sidebar-bg);
  display: flex;
  flex-direction: column;
  height: 100vh;
  position: sticky;
  top: 0;
  overflow-y: auto;
}

.sidebar-logo {
  display: flex;
  align-items: center;
  gap: 0.75rem;
  padding: 1.25rem 1rem 1rem;
  border-bottom: 1px solid var(--sidebar-border);
  margin-bottom: 0.5rem;
}

.sidebar-logo-mark {
  width: 32px;
  height: 32px;
  background: var(--logo-mark-bg);
  color: var(--logo-mark-color);
  border-radius: 8px;
  display: flex;
  align-items: center;
  justify-content: center;
  font-weight: 800;
  font-size: 1rem;
  flex-shrink: 0;
}

.sidebar-logo-name {
  display: block;
  color: #f1f5f9;
  font-weight: 700;
  font-size: 0.938rem;
  letter-spacing: -0.01em;
}

.sidebar-logo-sub {
  display: block;
  color: var(--sidebar-text);
  font-size: 0.75rem;
  font-weight: 400;
  margin-top: 1px;
}

.sidebar-nav {
  flex: 1;
  padding: 0 0.5rem;
}

.nav-section-label {
  padding: 0.5rem 0.5rem 0.25rem;
  font-size: 0.688rem;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.08em;
  color: #475569;
}

.sidebar-link {
  display: flex;
  align-items: center;
  gap: 0.625rem;
  padding: 0.5rem 0.75rem;
  border-radius: 6px;
  color: var(--sidebar-text);
  text-decoration: none;
  font-size: 0.875rem;
  font-weight: 500;
  transition: background 0.15s ease, color 0.15s ease;
  margin-bottom: 2px;
}

.sidebar-link:hover {
  background: var(--sidebar-hover-bg);
  color: var(--sidebar-text-active);
}

.sidebar-link.router-link-active {
  background: var(--sidebar-active-bg);
  color: var(--sidebar-text-active);
}

.sidebar-icon {
  width: 16px;
  height: 16px;
  flex-shrink: 0;
  display: flex;
  align-items: center;
}

.sidebar-footer {
  padding: 0.75rem;
  border-top: 1px solid var(--sidebar-border);
  display: flex;
  align-items: center;
  gap: 0.5rem;
}
```

---

### Step 6 — Updated App.vue Template

Replace the existing template entirely:

```vue
<template>
  <div class="app">
    <Sidebar>
      <template #footer>
        <LanguageSwitcher />
        <ProfileMenu
          @show-profile-details="showProfileDetails = true"
          @show-tasks="showTasks = true"
        />
      </template>
    </Sidebar>

    <div class="app-body">
      <FilterBar />
      <main class="main-content">
        <router-view />
      </main>
    </div>

    <ProfileDetailsModal :is-open="showProfileDetails" @close="showProfileDetails = false" />
    <TasksModal
      :is-open="showTasks"
      :tasks="tasks"
      @close="showTasks = false"
      @add-task="addTask"
      @delete-task="deleteTask"
      @toggle-task="toggleTask"
    />
  </div>
</template>
```

Import and register `Sidebar` in the component's `components` object. Remove the old `nav` markup and the `.top-nav` / `.nav-container` / `.nav-tabs` CSS blocks.

---

### Step 7 — Updated Global CSS in App.vue

Replace layout-related global styles. Keep all non-layout globals (`.card`, `.badge`, `.stat-card`, `table`, etc.) unchanged.

**Replace:**
```css
.app {
  display: flex;
  flex-direction: column;
  min-height: 100vh;
}

/* Remove entirely: .top-nav, .nav-container, .nav-tabs, .logo, .subtitle */

.main-content {
  flex: 1;
  max-width: 1600px;
  width: 100%;
  margin: 0 auto;
  padding: 1.5rem 2rem;
}
```

**With:**
```css
html, body {
  height: 100%;
  overflow: hidden;
}

.app {
  display: flex;
  height: 100vh;
  overflow: hidden;
}

.app-body {
  flex: 1;
  display: flex;
  flex-direction: column;
  overflow-y: auto;
  background: #f8fafc;
  min-width: 0; /* prevent flex overflow */
}

.main-content {
  flex: 1;
  padding: 1.5rem 2rem;
}
```

---

### Step 8 — FilterBar Sticky Adjustment

The `FilterBar` now sits inside `.app-body` (the scrollable column) instead of below `.top-nav`. Update FilterBar's sticky behavior:

```css
/* In FilterBar.vue <style scoped> — change position/z-index if needed */
.filter-bar {
  position: sticky;
  top: 0;
  z-index: 50;   /* lower than old 100, no longer needs to clear a top nav */
  background: #ffffff;
  border-bottom: 1px solid #e2e8f0;
}
```

---

### Step 9 — ProfileMenu and LanguageSwitcher in Sidebar Footer

The `ProfileMenu` and `LanguageSwitcher` move from the top nav into the sidebar footer slot. They keep their existing internal styling — only their position in the DOM changes.

If `ProfileMenu` uses fixed/absolute positioning for its dropdown, verify the dropdown doesn't get clipped by `.sidebar`'s `overflow-y: auto`. If it does, change the sidebar to `overflow: visible` on the footer area, or use `position: fixed` for the dropdown.

---

## Quality Checklist

Before marking the redesign complete, verify each item:

- [ ] Sidebar is visible and 240px wide on all routes
- [ ] Active route is highlighted in the sidebar (dark bg + light text)
- [ ] No horizontal scrollbar on the page at 1280px viewport
- [ ] FilterBar sticks to top of the content area when scrolling
- [ ] ProfileMenu dropdown is not clipped by the sidebar
- [ ] Dark sidebar does not inherit global `body` light background
- [ ] All 7 nav links render and navigate correctly
- [ ] `LanguageSwitcher` and `ProfileMenu` are visible in sidebar footer
- [ ] Cards, tables, badges, stat-cards all look identical to before
- [ ] No console errors about missing props or slots

---

## Common Pitfalls

| Problem | Fix |
|---|---|
| Sidebar missing on some routes | Sidebar is in `App.vue`, not inside views — check it's outside `<router-view>` |
| Page does not scroll | Ensure `overflow-y: auto` is on `.app-body`, not on `.app` |
| Sidebar clips dropdown menus | Add `overflow: visible` to `.sidebar-footer` or use `position: fixed` on the dropdown |
| Content bleeds under sidebar | Confirm `min-width: 0` on `.app-body` to prevent flex child overflow |
| `router-link-active` fires on all routes | Dashboard uses `to="/"` — add `exact` prop: `<router-link to="/" exact ...>` (Vue Router v4: use `exact-active-class` or the `activeClass` prop) |
| FilterBar jumps when navigating | It's rendered once in `App.vue` — filter state should live in a composable, not in the view |

---

## File Change Summary

| File | Change |
|---|---|
| `client/src/App.vue` | Replace template + remove top-nav CSS + add app/app-body layout CSS |
| `client/src/components/Sidebar.vue` | **Create new** — sidebar layout + nav items + scoped styles |
| `client/src/components/FilterBar.vue` | Minor: update sticky `top`/`z-index` values |

No view files, no `main.js`, no `api.js`, no backend files need changes.
