<template>
  <aside :class="['sidebar', { 'sidebar--collapsed': collapsed }]">
    <!-- Logo -->
    <div class="sidebar-logo">
      <span class="logo-mark">F</span>
      <transition name="fade">
        <div v-if="!collapsed" class="logo-text">
          <span class="logo-name">FactoryOS</span>
          <span class="logo-sub">Inventory Platform</span>
        </div>
      </transition>
    </div>

    <!-- Toggle button -->
    <button class="collapse-btn" @click="collapsed = !collapsed" :title="collapsed ? 'Expand sidebar' : 'Collapse sidebar'">
      <span v-html="collapsed ? icons.chevronRight : icons.chevronLeft" />
    </button>

    <!-- Nav -->
    <nav class="sidebar-nav">
      <div v-if="!collapsed" class="nav-section-label">Workspace</div>
      <router-link
        v-for="item in navItems"
        :key="item.to"
        :to="item.to"
        class="sidebar-link"
        :class="{ 'sidebar-link--exact': item.exact }"
        :title="collapsed ? item.label : ''"
      >
        <span class="sidebar-icon" v-html="item.icon" />
        <transition name="fade">
          <span v-if="!collapsed" class="sidebar-label">{{ item.label }}</span>
        </transition>
      </router-link>
    </nav>

    <!-- Footer slot -->
    <div class="sidebar-footer">
      <slot name="footer" />
    </div>
  </aside>
</template>

<script>
import { ref, onMounted, onUnmounted } from 'vue'

export default {
  name: 'Sidebar',
  setup() {
    const collapsed = ref(false)

    const icons = {
      grid: `<svg width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><rect x="3" y="3" width="7" height="7"/><rect x="14" y="3" width="7" height="7"/><rect x="3" y="14" width="7" height="7"/><rect x="14" y="14" width="7" height="7"/></svg>`,
      box: `<svg width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M21 16V8a2 2 0 0 0-1-1.73l-7-4a2 2 0 0 0-2 0l-7 4A2 2 0 0 0 3 8v8a2 2 0 0 0 1 1.73l7 4a2 2 0 0 0 2 0l7-4A2 2 0 0 0 21 16z"/></svg>`,
      clipboard: `<svg width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M16 4h2a2 2 0 0 1 2 2v14a2 2 0 0 1-2 2H6a2 2 0 0 1-2-2V6a2 2 0 0 1 2-2h2"/><rect x="8" y="2" width="8" height="4" rx="1" ry="1"/></svg>`,
      chart: `<svg width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><line x1="18" y1="20" x2="18" y2="10"/><line x1="12" y1="20" x2="12" y2="4"/><line x1="6" y1="20" x2="6" y2="14"/></svg>`,
      trend: `<svg width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><polyline points="23 6 13.5 15.5 8.5 10.5 1 18"/><polyline points="17 6 23 6 23 12"/></svg>`,
      document: `<svg width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M14 2H6a2 2 0 0 0-2 2v16a2 2 0 0 0 2 2h12a2 2 0 0 0 2-2V8z"/><polyline points="14 2 14 8 20 8"/><line x1="16" y1="13" x2="8" y2="13"/><line x1="16" y1="17" x2="8" y2="17"/></svg>`,
      refresh: `<svg width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><polyline points="23 4 23 10 17 10"/><polyline points="1 20 1 14 7 14"/><path d="M3.51 9a9 9 0 0 1 14.85-3.36L23 10M1 14l4.64 4.36A9 9 0 0 0 20.49 15"/></svg>`,
      chevronLeft: `<svg width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><polyline points="15 18 9 12 15 6"/></svg>`,
      chevronRight: `<svg width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><polyline points="9 18 15 12 9 6"/></svg>`,
    }

    const navItems = [
      { to: '/', label: 'Overview', icon: icons.grid, exact: true },
      { to: '/inventory', label: 'Inventory', icon: icons.box },
      { to: '/orders', label: 'Orders', icon: icons.clipboard },
      { to: '/spending', label: 'Finance', icon: icons.chart },
      { to: '/demand', label: 'Demand', icon: icons.trend },
      { to: '/reports', label: 'Reports', icon: icons.document },
      { to: '/restocking', label: 'Restocking', icon: icons.refresh },
    ]

    const handleResize = () => {
      collapsed.value = window.innerWidth < 1024
    }

    onMounted(() => {
      handleResize()
      window.addEventListener('resize', handleResize)
    })

    onUnmounted(() => {
      window.removeEventListener('resize', handleResize)
    })

    return {
      collapsed,
      icons,
      navItems,
    }
  }
}
</script>

<style scoped>
.sidebar {
  width: 240px;
  min-width: 240px;
  background: #0f172a;
  display: flex;
  flex-direction: column;
  height: 100vh;
  position: sticky;
  top: 0;
  overflow: visible; /* important: lets dropdown menus escape */
  transition: width 0.2s ease, min-width 0.2s ease;
  flex-shrink: 0;
  z-index: 100;
}

.sidebar--collapsed {
  width: 56px;
  min-width: 56px;
}

.sidebar-logo {
  display: flex;
  align-items: center;
  gap: 0.75rem;
  padding: 1.125rem 0.75rem 1rem;
  border-bottom: 1px solid rgba(255,255,255,0.06);
  min-height: 64px;
  overflow: hidden;
}

.logo-mark {
  width: 32px;
  height: 32px;
  background: #2563eb;
  color: #fff;
  border-radius: 8px;
  display: flex;
  align-items: center;
  justify-content: center;
  font-weight: 800;
  font-size: 1rem;
  flex-shrink: 0;
}

.logo-text {
  display: flex;
  flex-direction: column;
  overflow: hidden;
  white-space: nowrap;
}

.logo-name {
  color: #f1f5f9;
  font-weight: 700;
  font-size: 0.938rem;
  letter-spacing: -0.01em;
}

.logo-sub {
  color: #94a3b8;
  font-size: 0.75rem;
  margin-top: 1px;
}

.collapse-btn {
  position: absolute;
  top: 52px;
  right: -12px;
  width: 24px;
  height: 24px;
  background: #1e293b;
  border: 1px solid rgba(255,255,255,0.12);
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  color: #94a3b8;
  cursor: pointer;
  z-index: 10;
  transition: background 0.15s, color 0.15s;
}

.collapse-btn:hover {
  background: #334155;
  color: #f1f5f9;
}

.sidebar-nav {
  flex: 1;
  padding: 0.5rem 0.375rem;
  overflow-y: auto;
  overflow-x: hidden;
}

.nav-section-label {
  padding: 0.5rem 0.5rem 0.25rem;
  font-size: 0.688rem;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.08em;
  color: #475569;
  white-space: nowrap;
  overflow: hidden;
}

.sidebar-link {
  display: flex;
  align-items: center;
  gap: 0.625rem;
  padding: 0.5rem 0.625rem;
  border-radius: 6px;
  color: #94a3b8;
  text-decoration: none;
  font-size: 0.875rem;
  font-weight: 500;
  transition: background 0.15s, color 0.15s;
  margin-bottom: 2px;
  white-space: nowrap;
  overflow: hidden;
}

.sidebar-link:hover {
  background: rgba(255,255,255,0.05);
  color: #f1f5f9;
}

/* Vue Router v4 active class */
.sidebar-link.router-link-active {
  background: rgba(255,255,255,0.08);
  color: #f1f5f9;
}

/* Prevent / from matching all routes */
.sidebar-link--exact.router-link-active:not(.router-link-exact-active) {
  background: transparent;
  color: #94a3b8;
}
.sidebar-link--exact.router-link-exact-active {
  background: rgba(255,255,255,0.08);
  color: #f1f5f9;
}

.sidebar-icon {
  width: 16px;
  height: 16px;
  flex-shrink: 0;
  display: flex;
  align-items: center;
  justify-content: center;
}

/* Collapsed: center icons */
.sidebar--collapsed .sidebar-link {
  justify-content: center;
  padding: 0.5rem;
}

.sidebar-label {
  overflow: hidden;
  text-overflow: ellipsis;
}

.sidebar-footer {
  padding: 0.625rem 0.375rem;
  border-top: 1px solid rgba(255,255,255,0.06);
  display: flex;
  align-items: center;
  gap: 0.5rem;
  overflow: hidden;
  justify-content: flex-start;
}

.sidebar--collapsed .sidebar-footer {
  justify-content: center;
  flex-direction: column;
  gap: 0.375rem;
}

/* Fade transition for labels */
.fade-enter-active,
.fade-leave-active {
  transition: opacity 0.15s ease;
}
.fade-enter-from,
.fade-leave-to {
  opacity: 0;
}
</style>
