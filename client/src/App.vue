<template>
  <div
    class="app-shell"
    :class="{
      'is-rail': isRail,
      'is-drawer-open': drawerOpen,
      'is-overlay-open': overlayOpen
    }"
  >
    <aside class="sidebar" id="app-sidebar">
      <div class="sidebar-brand">
        <span class="brand-mark" aria-hidden="true">C</span>
        <span class="brand-text">
          <span class="brand-name">{{ t('nav.companyName') }}</span>
          <span class="brand-subtitle">{{ t('nav.subtitle') }}</span>
        </span>
      </div>

      <nav class="sidebar-nav" aria-label="Main">
        <router-link
          v-for="item in navItems"
          :key="item.to"
          :to="item.to"
          class="nav-item"
          :exact-active-class="item.exact ? 'is-active' : undefined"
          :active-class="item.exact ? '' : 'is-active'"
          :title="isRail ? item.label : undefined"
        >
          <NavIcon :name="item.icon" />
          <span class="nav-label">{{ item.label }}</span>
        </router-link>
      </nav>

      <div class="sidebar-footer">
        <ProfileMenu
          @show-profile-details="showProfileDetails = true"
          @show-tasks="showTasks = true"
        />
        <button
          v-if="breakpoint !== 'mobile'"
          class="collapse-toggle"
          :aria-expanded="!isRail"
          :title="isRail ? t('nav.expand') : undefined"
          @click="onToggleSidebar"
        >
          <NavIcon :name="isRail ? 'chevron-right' : 'chevron-left'" />
          <span class="nav-label">{{ isRail ? t('nav.expand') : t('nav.collapse') }}</span>
        </button>
      </div>
    </aside>

    <div class="sidebar-backdrop" @click="closeNav" />

    <div class="shell-main">
      <header class="topbar">
        <button
          class="menu-toggle"
          aria-controls="app-sidebar"
          :aria-expanded="drawerOpen"
          @click="drawerOpen = !drawerOpen"
        >
          <NavIcon name="menu" />
        </button>
        <h1 class="topbar-title">{{ currentTitle }}</h1>
        <div class="topbar-actions">
          <LanguageSwitcher />
        </div>
      </header>
      <FilterBar />
      <main class="main-content">
        <router-view />
      </main>
    </div>

    <ProfileDetailsModal
      :is-open="showProfileDetails"
      @close="showProfileDetails = false"
    />

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

<script>
import { ref, onMounted, onUnmounted, computed, watch } from 'vue'
import { useRoute } from 'vue-router'
import { api } from './api'
import { useAuth } from './composables/useAuth'
import { useI18n } from './composables/useI18n'
import FilterBar from './components/FilterBar.vue'
import ProfileMenu from './components/ProfileMenu.vue'
import ProfileDetailsModal from './components/ProfileDetailsModal.vue'
import TasksModal from './components/TasksModal.vue'
import LanguageSwitcher from './components/LanguageSwitcher.vue'
import NavIcon from './components/NavIcon.vue'

const STORAGE_KEY = 'sidebar-collapsed'
const readCollapsed = () => {
  try { return localStorage.getItem(STORAGE_KEY) === '1' } catch { return false }
}

// Keep these in sync with the @media queries in the <style> block below
const DESKTOP_QUERY = '(min-width: 1280px)'
const TABLET_QUERY = '(min-width: 768px)'

export default {
  name: 'App',
  components: {
    FilterBar,
    ProfileMenu,
    ProfileDetailsModal,
    TasksModal,
    LanguageSwitcher,
    NavIcon
  },
  setup() {
    const { currentUser } = useAuth()
    const { t } = useI18n()
    const route = useRoute()
    const showProfileDetails = ref(false)
    const showTasks = ref(false)
    const apiTasks = ref([])

    // Merge mock tasks from currentUser with API tasks
    const tasks = computed(() => {
      return [...currentUser.value.tasks, ...apiTasks.value]
    })

    const loadTasks = async () => {
      try {
        apiTasks.value = await api.getTasks()
      } catch (err) {
        console.error('Failed to load tasks:', err)
      }
    }

    const addTask = async (taskData) => {
      try {
        const newTask = await api.createTask(taskData)
        // Add new task to the beginning of the array
        apiTasks.value.unshift(newTask)
      } catch (err) {
        console.error('Failed to add task:', err)
      }
    }

    const deleteTask = async (taskId) => {
      try {
        // Check if it's a mock task (from currentUser)
        const isMockTask = currentUser.value.tasks.some(t => t.id === taskId)

        if (isMockTask) {
          // Remove from mock tasks
          const index = currentUser.value.tasks.findIndex(t => t.id === taskId)
          if (index !== -1) {
            currentUser.value.tasks.splice(index, 1)
          }
        } else {
          // Remove from API tasks
          await api.deleteTask(taskId)
          apiTasks.value = apiTasks.value.filter(t => t.id !== taskId)
        }
      } catch (err) {
        console.error('Failed to delete task:', err)
      }
    }

    const toggleTask = async (taskId) => {
      try {
        // Check if it's a mock task (from currentUser)
        const mockTask = currentUser.value.tasks.find(t => t.id === taskId)

        if (mockTask) {
          // Toggle mock task status
          mockTask.status = mockTask.status === 'pending' ? 'completed' : 'pending'
        } else {
          // Toggle API task
          const updatedTask = await api.toggleTask(taskId)
          const index = apiTasks.value.findIndex(t => t.id === taskId)
          if (index !== -1) {
            apiTasks.value[index] = updatedTask
          }
        }
      } catch (err) {
        console.error('Failed to toggle task:', err)
      }
    }

    // Sidebar has three tiers so wide screens get labels, mid-size screens keep
    // content width (icon rail), and phones get an off-canvas drawer.
    const navItems = computed(() => [
      { to: '/', label: t('nav.overview'), icon: 'overview', exact: true },
      { to: '/inventory', label: t('nav.inventory'), icon: 'inventory' },
      { to: '/orders', label: t('nav.orders'), icon: 'orders' },
      { to: '/spending', label: t('nav.finance'), icon: 'finance' },
      { to: '/demand', label: t('nav.demandForecast'), icon: 'demand' },
      { to: '/reports', label: t('nav.reports'), icon: 'reports' }
    ])

    const currentTitle = computed(() => {
      const item = navItems.value.find(i => (i.exact ? route.path === i.to : route.path.startsWith(i.to)))
      return item ? item.label : t('nav.companyName')
    })

    const collapsed = ref(readCollapsed())
    const drawerOpen = ref(false)
    const overlayOpen = ref(false)
    const breakpoint = ref('desktop')

    // Rail is forced on tablets regardless of the saved preference; the saved
    // preference only applies on desktop. Never a rail on mobile (drawer is full width).
    const isRail = computed(() =>
      (breakpoint.value === 'tablet' && !overlayOpen.value) ||
      (breakpoint.value === 'desktop' && collapsed.value)
    )

    const toggleCollapsed = () => {
      collapsed.value = !collapsed.value
      try {
        localStorage.setItem(STORAGE_KEY, collapsed.value ? '1' : '0')
      } catch { /* storage blocked: keep in-memory state */ }
    }

    // On tablets the expansion is temporary, so it must not overwrite the saved desktop preference
    const onToggleSidebar = () => {
      if (breakpoint.value === 'tablet') overlayOpen.value = !overlayOpen.value
      else toggleCollapsed()
    }

    const closeNav = () => {
      drawerOpen.value = false
      overlayOpen.value = false
    }

    // The drawer/overlay would otherwise keep covering the page we just navigated to
    watch(() => route.path, closeNav)

    const desktopMql = window.matchMedia(DESKTOP_QUERY)
    const tabletMql = window.matchMedia(TABLET_QUERY)
    const updateBreakpoint = () => {
      breakpoint.value = desktopMql.matches ? 'desktop' : tabletMql.matches ? 'tablet' : 'mobile'
    }
    // A stale drawer/overlay state would be wrong in the new tier
    watch(breakpoint, closeNav)
    updateBreakpoint()

    const onKeydown = (e) => {
      if (e.key === 'Escape') closeNav()
    }

    onMounted(() => {
      loadTasks()
      desktopMql.addEventListener('change', updateBreakpoint)
      tabletMql.addEventListener('change', updateBreakpoint)
      window.addEventListener('keydown', onKeydown)
    })
    onUnmounted(() => {
      desktopMql.removeEventListener('change', updateBreakpoint)
      tabletMql.removeEventListener('change', updateBreakpoint)
      window.removeEventListener('keydown', onKeydown)
    })

    return {
      t,
      showProfileDetails,
      showTasks,
      tasks,
      addTask,
      deleteTask,
      toggleTask,
      navItems,
      currentTitle,
      breakpoint,
      isRail,
      drawerOpen,
      overlayOpen,
      onToggleSidebar,
      closeNav
    }
  }
}
</script>

<style>
:root {
  /* Spacing (4px base) */
  --space-1: 4px; --space-2: 8px; --space-3: 12px; --space-4: 16px;
  --space-5: 20px; --space-6: 24px; --space-8: 32px; --space-10: 40px; --space-12: 48px;

  /* Radius and elevation */
  --radius-sm: 6px; --radius-md: 8px; --radius-lg: 12px;
  --shadow-sm: 0 1px 2px rgba(15, 23, 42, 0.06);
  --shadow-md: 0 4px 12px rgba(15, 23, 42, 0.08);
  --shadow-lg: 0 10px 25px rgba(15, 23, 42, 0.18);

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
  --color-primary: #4f46e5;
  --color-primary-soft: #eef2ff;
  --color-success: #16a34a; --color-success-soft: #f0fdf4;
  --color-warning: #ca8a04; --color-warning-soft: #fefce8;
  /* Yellow #ca8a04 is too light for text on white/soft yellow; use this for text */
  --color-warning-text: #a16207;
  --color-danger: #dc2626; --color-danger-soft: #fef2f2;
  /* Info stays blue (status color) while primary is indigo */
  --color-info: #2563eb; --color-info-soft: #eff6ff;

  /* Sidebar */
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
  /* Sidebar sits above the topbar so its profile dropdown paints over the topbar and filter bar. Modals use 1000/2000. */
  --z-sidebar: 40; --z-topbar: 30; --z-backdrop: 45;
}

* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

body {
  font-family: 'Inter', -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Oxygen, Ubuntu, Cantarell, sans-serif;
  font-size: var(--text-base);
  line-height: var(--leading-normal);
  background: var(--color-bg);
  color: var(--color-text);
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
}

/* ---------- Shell ---------- */

.app-shell {
  display: grid;
  grid-template-columns: var(--sidebar-width) minmax(0, 1fr);
  min-height: 100vh;
  background: var(--color-bg);
}

/* Not .is-collapsed: rail state is derived in JS (saved preference on desktop, forced on tablet) */
.app-shell.is-rail {
  grid-template-columns: var(--sidebar-collapsed) minmax(0, 1fr);
}

/* Overlay/drawer sidebars are position: fixed, i.e. out of grid flow; pin the
   content to its column so it doesn't slide into the sidebar's column */
.shell-main {
  grid-column: 2;
  min-width: 0; /* stops wide tables/charts from blowing out the grid column */
  display: flex;
  flex-direction: column;
}

.sidebar {
  position: sticky;
  top: 0;
  height: 100vh;
  z-index: var(--z-sidebar);
  display: flex;
  flex-direction: column;
  gap: var(--space-6);
  padding: var(--space-4) var(--space-3);
  background: var(--sidebar-bg);
  color: var(--sidebar-text);
  /* No overflow here: the profile dropdown opens upward and wider than the rail,
     and any overflow value on the sidebar would clip it. Scrolling lives on .sidebar-nav. */
}

.sidebar-brand {
  display: flex;
  align-items: center;
  gap: var(--space-3);
  padding: 0 var(--space-3);
  height: 40px;
  color: var(--sidebar-text-active);
}

.brand-mark {
  flex-shrink: 0;
  width: 32px;
  height: 32px;
  display: inline-flex;
  align-items: center;
  justify-content: center;
  border-radius: var(--radius-md);
  background: var(--color-primary);
  color: #ffffff;
  font-weight: 700;
  font-size: var(--text-lg);
}

.brand-text {
  display: flex;
  flex-direction: column;
  min-width: 0;
}

.brand-name {
  font-size: var(--text-base);
  font-weight: 700;
  letter-spacing: -0.01em;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.brand-subtitle {
  font-size: var(--text-xs);
  color: var(--sidebar-text);
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.sidebar-nav {
  display: flex;
  flex-direction: column;
  gap: var(--space-1);
  flex: 1;
  min-height: 0; /* lets the flex child shrink so overflow-y can actually scroll */
  overflow-y: auto;
}

.sidebar-footer {
  display: flex;
  flex-direction: column;
  gap: var(--space-2);
  border-top: 1px solid rgba(255, 255, 255, 0.08);
  padding-top: var(--space-3);
}

.nav-item,
.collapse-toggle {
  position: relative;
  display: flex;
  align-items: center;
  gap: var(--space-3);
  height: 40px;
  padding: 0 var(--space-3);
  border-radius: var(--radius-md);
  color: inherit;
  text-decoration: none;
  font-family: inherit;
  font-size: var(--text-sm);
  font-weight: 500;
  white-space: nowrap;
}

.collapse-toggle {
  width: 100%;
  background: none;
  border: none;
  cursor: pointer;
}

.nav-item:hover,
.collapse-toggle:hover {
  background: var(--sidebar-item-hover);
  color: var(--sidebar-text-active);
}

.nav-item.is-active {
  background: var(--sidebar-item-active);
  color: var(--sidebar-text-active);
}

.nav-item.is-active::before {
  content: '';
  position: absolute;
  left: calc(-1 * var(--space-3));
  top: 8px;
  bottom: 8px;
  width: 3px;
  border-radius: 0 3px 3px 0;
  background: var(--color-primary);
}

.nav-item:focus-visible,
.collapse-toggle:focus-visible,
.menu-toggle:focus-visible {
  outline: 2px solid var(--color-primary);
  outline-offset: 2px;
}

.is-rail .nav-label,
.is-rail .brand-text {
  display: none;
}

.is-rail .nav-item,
.is-rail .collapse-toggle {
  justify-content: center;
  padding: 0;
}

.is-rail .sidebar-brand {
  justify-content: center;
  padding: 0;
}

.topbar {
  position: sticky;
  top: 0;
  z-index: var(--z-topbar);
  height: var(--topbar-height);
  display: flex;
  align-items: center;
  gap: var(--space-4);
  padding: 0 var(--space-8);
  background: var(--color-surface);
  border-bottom: 1px solid var(--color-border);
}

.topbar-title {
  font-size: var(--text-lg);
  font-weight: 600;
  color: var(--color-text);
}

.topbar-actions {
  margin-left: auto;
  display: flex;
  align-items: center;
  gap: var(--space-3);
}

.menu-toggle {
  display: none;
  align-items: center;
  justify-content: center;
  width: 36px;
  height: 36px;
  background: none;
  border: 1px solid var(--color-border);
  border-radius: var(--radius-md);
  color: var(--color-text);
  cursor: pointer;
}

.sidebar-backdrop {
  display: none;
}

.main-content {
  flex: 1;
  width: 100%;
  max-width: var(--content-max);
  margin: 0 auto;
  padding: var(--space-8);
}

/*
 * Responsive tiers (keep in sync with DESKTOP_QUERY / TABLET_QUERY in the script):
 *   >= 1280px   full sidebar, optional manual rail
 *   768-1279px  icon rail; toggle expands it as an overlay (content does not reflow)
 *   < 768px     off-canvas drawer
 */

/* Tablet overlay: fixed so the content column stays 72px-offset and doesn't reflow */
@media (max-width: 1279px) {
  /* Column stays rail-width even while the overlay is open; that is what keeps content from reflowing */
  .app-shell,
  .app-shell.is-rail,
  .app-shell.is-overlay-open {
    grid-template-columns: var(--sidebar-collapsed) minmax(0, 1fr);
  }

  .app-shell.is-overlay-open .sidebar {
    position: fixed;
    inset: 0 auto 0 0;
    width: var(--sidebar-width);
    z-index: calc(var(--z-backdrop) + 1);
    box-shadow: var(--shadow-lg);
  }

  .app-shell.is-overlay-open .sidebar-backdrop {
    display: block;
    position: fixed;
    inset: 0;
    z-index: var(--z-backdrop);
    background: rgba(15, 23, 42, 0.2);
  }
}

@media (max-width: 767px) {
  .app-shell,
  .app-shell.is-rail,
  .app-shell.is-overlay-open {
    grid-template-columns: minmax(0, 1fr);
  }

  .shell-main {
    grid-column: 1;
  }

  .sidebar {
    position: fixed;
    inset: 0 auto 0 0;
    width: var(--sidebar-width);
    transform: translateX(-100%);
    transition: transform 0.2s ease;
  }

  .is-drawer-open .sidebar {
    transform: none;
    z-index: calc(var(--z-backdrop) + 1);
  }

  .is-drawer-open .sidebar-backdrop {
    display: block;
    position: fixed;
    inset: 0;
    z-index: var(--z-backdrop);
    background: rgba(15, 23, 42, 0.4);
  }

  .menu-toggle {
    display: inline-flex;
  }

  .topbar,
  .main-content {
    padding-left: var(--space-4);
    padding-right: var(--space-4);
  }
}

/* ---------- Shared primitives ---------- */

.page-header {
  display: flex;
  align-items: flex-end;
  justify-content: space-between;
  gap: var(--space-4);
  margin-bottom: var(--space-6);
}

.page-header h2 {
  font-size: var(--text-2xl);
  font-weight: 700;
  line-height: var(--leading-tight);
  letter-spacing: -0.025em;
  color: var(--color-text);
  margin: 0;
}

.page-header p {
  color: var(--color-text-muted);
  font-size: var(--text-base);
  margin: var(--space-1) 0 0;
}

.stats-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(220px, 1fr));
  gap: var(--space-4);
  margin-bottom: var(--space-6);
}

.stat-card {
  background: var(--color-surface);
  padding: var(--space-5);
  border-radius: var(--radius-lg);
  border: 1px solid var(--color-border);
  box-shadow: var(--shadow-sm);
  transition: border-color 0.2s ease, box-shadow 0.2s ease;
}

.stat-card:hover {
  border-color: #cbd5e1;
  box-shadow: var(--shadow-md);
}

.stat-label {
  color: var(--color-text-muted);
  font-size: var(--text-xs);
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.04em;
  margin-bottom: var(--space-2);
}

.stat-value {
  font-size: var(--text-3xl);
  font-weight: 700;
  color: var(--color-text);
  letter-spacing: -0.025em;
  font-variant-numeric: tabular-nums;
}

.stat-card.warning .stat-value { color: var(--color-warning-text); }
.stat-card.success .stat-value { color: var(--color-success); }
.stat-card.danger .stat-value { color: var(--color-danger); }
.stat-card.info .stat-value { color: var(--color-info); }

.card {
  background: var(--color-surface);
  border-radius: var(--radius-lg);
  padding: var(--space-6);
  border: 1px solid var(--color-border);
  box-shadow: var(--shadow-sm);
  margin-bottom: var(--space-6);
}

.card-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: var(--space-4);
  padding-bottom: var(--space-3);
  border-bottom: 1px solid var(--color-border);
}

.card-title {
  font-size: var(--text-lg);
  font-weight: 600;
  color: var(--color-text);
  letter-spacing: -0.01em;
}

.table-container {
  overflow-x: auto;
}

table {
  width: 100%;
  border-collapse: collapse;
}

thead {
  background: var(--color-surface-muted);
}

th {
  text-align: left;
  padding: var(--space-3) var(--space-4);
  font-weight: 600;
  color: var(--color-text-muted);
  font-size: var(--text-xs);
  text-transform: uppercase;
  letter-spacing: 0.04em;
}

td {
  padding: var(--space-3) var(--space-4);
  border-top: 1px solid var(--color-border);
  color: var(--color-text);
  font-size: var(--text-sm);
}

tbody tr {
  transition: background-color 0.15s ease;
}

tbody tr:hover {
  background: var(--color-surface-muted);
}

td.num,
th.num {
  text-align: right;
  font-variant-numeric: tabular-nums;
}

.badge {
  display: inline-flex;
  align-items: center;
  padding: 2px var(--space-2);
  border-radius: 999px;
  font-size: var(--text-xs);
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.025em;
}

.badge.success,
.badge.increasing {
  background: var(--color-success-soft);
  color: var(--color-success);
}

.badge.warning,
.badge.medium {
  background: var(--color-warning-soft);
  color: var(--color-warning-text);
}

.badge.danger,
.badge.decreasing,
.badge.high {
  background: var(--color-danger-soft);
  color: var(--color-danger);
}

.badge.info,
.badge.low {
  background: var(--color-info-soft);
  color: var(--color-info);
}

/* Stable has no good/bad meaning: neutral slate rather than a status color */
.badge.stable {
  background: var(--color-surface-muted);
  color: var(--color-text-muted);
}

.loading {
  text-align: center;
  padding: var(--space-12);
  color: var(--color-text-muted);
  font-size: var(--text-base);
}

.error {
  background: var(--color-danger-soft);
  border: 1px solid #fecaca;
  color: var(--color-danger);
  padding: var(--space-4);
  border-radius: var(--radius-md);
  margin: var(--space-4) 0;
  font-size: var(--text-base);
}

.empty-state {
  text-align: center;
  padding: var(--space-10) var(--space-4);
  color: var(--color-text-muted);
  font-size: var(--text-base);
}

.skeleton {
  background: linear-gradient(90deg, var(--color-surface-muted), var(--color-border), var(--color-surface-muted));
  background-size: 200% 100%;
  animation: shimmer 1.2s infinite;
  border-radius: var(--radius-sm);
}

@keyframes shimmer {
  from { background-position: 200% 0; }
  to { background-position: -200% 0; }
}
</style>
