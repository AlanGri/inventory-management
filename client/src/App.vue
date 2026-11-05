<template>
  <div class="app" :class="{ 'drawer-open': drawerOpen, rail: isRail }">
    <aside id="app-sidebar" class="sidebar">
      <div class="sidebar-head">
        <div class="brand">
          <span class="brand-mark" aria-hidden="true"></span>
          <div class="logo">
            <h1>{{ t('nav.companyName') }}</h1>
            <span class="subtitle">{{ t('nav.subtitle') }}</span>
          </div>
        </div>
        <button class="icon-btn drawer-close" aria-label="Close menu" @click="drawerOpen = false">
          <Icon name="x" />
        </button>
      </div>
      <nav class="nav-tabs">
        <router-link
          v-for="item in navItems"
          :key="item.to"
          :to="item.to"
          :class="{ active: $route.path === item.to }"
          :title="item.label"
        >
          <Icon :name="item.icon" />
          <span class="nav-label">{{ item.label }}</span>
        </router-link>
      </nav>
      <div class="sidebar-footer">
        <button
          class="collapse-toggle"
          :aria-pressed="collapsed ? 'true' : 'false'"
          :aria-label="collapsed ? 'Expand sidebar' : 'Collapse sidebar'"
          :title="collapsed ? 'Expand sidebar' : 'Collapse sidebar'"
          @click="collapsed = !collapsed"
        >
          <Icon :name="collapsed ? 'panel-left-open' : 'panel-left-close'" />
          <span class="nav-label">Collapse</span>
        </button>
        <LanguageSwitcher />
        <ProfileMenu
          @show-profile-details="showProfileDetails = true"
          @show-tasks="showTasks = true"
        />
      </div>
    </aside>
    <div v-if="drawerOpen" class="drawer-backdrop" @click="drawerOpen = false"></div>

    <div class="main-area">
      <header class="mobile-topbar">
        <button
          class="icon-btn"
          aria-label="Open menu"
          aria-controls="app-sidebar"
          :aria-expanded="drawerOpen ? 'true' : 'false'"
          @click="drawerOpen = !drawerOpen"
        >
          <Icon name="menu" />
        </button>
        <span class="brand-mark" aria-hidden="true"></span>
        <span class="mobile-logo">{{ t('nav.companyName') }}</span>
      </header>
      <FilterBar />
      <main class="main-content">
        <router-view v-slot="{ Component }">
          <transition name="page" mode="out-in">
            <component :is="Component" :key="$route.path" />
          </transition>
        </router-view>
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
import Icon from './components/Icon.vue'

export default {
  name: 'App',
  components: {
    FilterBar,
    ProfileMenu,
    ProfileDetailsModal,
    TasksModal,
    LanguageSwitcher,
    Icon
  },
  setup() {
    const { currentUser } = useAuth()
    const { t } = useI18n()
    const showProfileDetails = ref(false)
    const showTasks = ref(false)
    const apiTasks = ref([])
    const route = useRoute()
    const drawerOpen = ref(false)

    // Desktop collapse preference survives reloads; storage can throw in private mode
    const readCollapsed = () => {
      try { return localStorage.getItem('sidebar-collapsed') === '1' } catch { return false }
    }
    const collapsed = ref(readCollapsed())
    watch(collapsed, (v) => {
      try { localStorage.setItem('sidebar-collapsed', v ? '1' : '0') } catch { /* ignore */ }
    })
    const tabletMq = window.matchMedia('(min-width: 768px) and (max-width: 1023px)')
    const desktopMq = window.matchMedia('(min-width: 1024px)')
    const isTablet = ref(tabletMq.matches)
    const isDesktop = ref(desktopMq.matches)
    const syncMq = () => {
      isTablet.value = tabletMq.matches
      isDesktop.value = desktopMq.matches
    }
    tabletMq.addEventListener('change', syncMq)
    desktopMq.addEventListener('change', syncMq)
    onUnmounted(() => {
      tabletMq.removeEventListener('change', syncMq)
      desktopMq.removeEventListener('change', syncMq)
    })
    const isRail = computed(() => isTablet.value || (isDesktop.value && collapsed.value))

    const navItems = computed(() => [
      { to: '/', icon: 'dashboard', label: t('nav.overview') },
      { to: '/inventory', icon: 'box', label: t('nav.inventory') },
      { to: '/orders', icon: 'clipboard', label: t('nav.orders') },
      { to: '/spending', icon: 'wallet', label: t('nav.finance') },
      { to: '/demand', icon: 'trending-up', label: t('nav.demandForecast') },
      { to: '/reports', icon: 'file-chart', label: 'Reports' }
    ])

    watch(() => route.path, () => { drawerOpen.value = false })
    const onKeydown = (e) => { if (e.key === 'Escape') drawerOpen.value = false }
    onMounted(() => window.addEventListener('keydown', onKeydown))
    onUnmounted(() => window.removeEventListener('keydown', onKeydown))

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

    onMounted(loadTasks)

    return {
      t,
      navItems,
      drawerOpen,
      collapsed,
      isRail,
      showProfileDetails,
      showTasks,
      tasks,
      addTask,
      deleteTask,
      toggleTask
    }
  }
}
</script>

<style>
:root {
  --bg: #08090b;
  --sidebar: #0c0d10;
  --card: #111215;
  --card-hover: #16171b;
  --surface-2: #1b1c21;
  --border: rgba(255, 255, 255, 0.06);
  --text: #f4f4f5;
  --text-muted: #a1a1aa;
  /* --text-dim is below 4.5:1 on dark surfaces: use for icons/decoration, not body text */
  --text-dim: #71717a;
  --accent: #2be8b0;
  --accent-soft: rgba(43, 232, 176, 0.12);
  --success: #2be8b0;
  --info: #7dd3fc;
  --warning: #fbbf24;
  --danger: #fb7185;
  --success-tint: rgba(43, 232, 176, 0.12);
  --info-tint: rgba(125, 211, 252, 0.12);
  --warning-tint: rgba(251, 191, 36, 0.12);
  --danger-tint: rgba(251, 113, 133, 0.12);
  --font-display: 'Geist', ui-sans-serif, system-ui, sans-serif;
  --r-sm: 10px;
  --r-pill: 999px;
  --ease: cubic-bezier(0.22, 1, 0.36, 1);
  --s1: 4px;
  --s2: 8px;
  --s3: 12px;
  --s4: 16px;
  --s5: 24px;
  --s6: 32px;
  --r: 18px;
  --topbar-h: 56px;
  --sidebar-w: 240px;
  --rail-w: 72px;
  color-scheme: dark;
}

* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

body {
  font-family: var(--font-display);
  font-feature-settings: 'ss01', 'cv11';
  font-variant-numeric: tabular-nums;
  background: var(--bg);
  color: var(--text);
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
}

::selection {
  background: var(--accent-soft);
  color: var(--text);
}

/* Shell */
.app {
  min-height: 100vh;
  position: relative;
  isolation: isolate;
}

/* Atmosphere: faint dot grid fading out from the top, plus a soft mint glow behind the content */
.app::before,
.app::after {
  content: '';
  position: fixed;
  inset: 0;
  pointer-events: none;
  z-index: -1;
}

.app::before {
  background-image: radial-gradient(rgba(255, 255, 255, 0.07) 1px, transparent 1px);
  background-size: 22px 22px;
  mask-image: radial-gradient(ellipse 80% 60% at 60% 0%, #000 0%, transparent 75%);
}

.app::after {
  background:
    radial-gradient(600px 320px at 78% -60px, rgba(43, 232, 176, 0.10), transparent 70%),
    radial-gradient(520px 300px at 30% -120px, rgba(125, 211, 252, 0.07), transparent 70%);
}

.sidebar {
  position: fixed;
  inset: 0 auto 0 0;
  width: var(--sidebar-w);
  display: flex;
  flex-direction: column;
  gap: var(--s5);
  padding: var(--s5) var(--s3);
  background: rgba(12, 13, 16, 0.72);
  backdrop-filter: blur(18px) saturate(140%);
  -webkit-backdrop-filter: blur(18px) saturate(140%);
  border-right: 1px solid var(--border);
  z-index: 100;
  transition: width 0.3s var(--ease);
}

.collapse-toggle {
  display: none;
  align-items: center;
  gap: var(--s3);
  width: 100%;
  padding: 9px var(--s3);
  background: none;
  border: none;
  border-radius: var(--r-pill);
  color: var(--text-muted);
  font: inherit;
  font-size: 0.906rem;
  font-weight: 500;
  cursor: pointer;
  transition: background-color 0.2s var(--ease), color 0.2s var(--ease);
}

.collapse-toggle:hover {
  color: var(--text);
  background: rgba(255, 255, 255, 0.04);
}

.collapse-toggle:focus-visible {
  outline: 2px solid var(--accent);
  outline-offset: 2px;
}

@media (min-width: 1024px) {
  .collapse-toggle {
    display: flex;
  }
}

.sidebar-head {
  display: flex;
  align-items: flex-start;
  justify-content: space-between;
  padding: 0 var(--s2);
}

.brand {
  display: flex;
  align-items: center;
  gap: var(--s3);
  min-width: 0;
}

.brand-mark {
  flex-shrink: 0;
  width: 30px;
  height: 30px;
  border-radius: 9px;
  background:
    radial-gradient(circle at 30% 25%, rgba(255, 255, 255, 0.55), transparent 45%),
    conic-gradient(from 210deg, #2be8b0, #7dd3fc, #6366f1, #2be8b0);
  box-shadow: inset 0 0 0 1px rgba(255, 255, 255, 0.18);
}

.logo {
  display: flex;
  flex-direction: column;
  gap: 2px;
  min-width: 0;
}

.logo h1 {
  font-size: 0.975rem;
  font-weight: 600;
  color: var(--text);
  letter-spacing: -0.02em;
  line-height: 1.2;
}

.subtitle {
  font-size: 0.75rem;
  color: var(--text-muted);
}

.nav-tabs {
  flex: 1;
  display: flex;
  flex-direction: column;
  gap: var(--s1);
  overflow-y: auto;
}

.nav-tabs a {
  display: flex;
  align-items: center;
  gap: var(--s3);
  padding: 9px var(--s3);
  color: var(--text-muted);
  text-decoration: none;
  font-weight: 500;
  font-size: 0.906rem;
  letter-spacing: -0.01em;
  border-radius: var(--r-pill);
  transition: background-color 0.2s var(--ease), color 0.2s var(--ease);
}

.nav-tabs a svg {
  flex-shrink: 0;
  opacity: 0.8;
  transition: color 0.2s var(--ease), opacity 0.2s var(--ease);
}

.nav-tabs a:hover {
  color: var(--text);
  background: rgba(255, 255, 255, 0.04);
}

.nav-tabs a.active {
  color: var(--text);
  background: var(--surface-2);
  box-shadow: inset 0 1px 0 rgba(255, 255, 255, 0.05);
}

.nav-tabs a.active svg {
  color: var(--accent);
  opacity: 1;
}

.nav-tabs a:focus-visible,
.icon-btn:focus-visible {
  outline: 2px solid var(--accent);
  outline-offset: 2px;
}

.sidebar-footer {
  display: flex;
  flex-direction: column;
  gap: var(--s2);
  padding-top: var(--s4);
}

/* Child components' dropdowns open downward by default; from the sidebar bottom they must open upward. */
.sidebar .sidebar-footer .language-button,
.sidebar .sidebar-footer .profile-button {
  width: 100%;
}

.sidebar .sidebar-footer .profile-name {
  flex: 1;
  text-align: left;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.sidebar .sidebar-footer .language-label {
  flex: 1;
  text-align: left;
}

.sidebar .sidebar-footer .dropdown-menu {
  top: auto;
  bottom: calc(100% + var(--s2));
  left: 0;
  right: auto;
}

.icon-btn {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  width: 40px;
  height: 40px;
  background: var(--surface-2);
  border: none;
  border-radius: var(--r-pill);
  color: var(--text);
  cursor: pointer;
  transition: background-color 0.2s var(--ease);
}

.icon-btn:hover {
  background: #23242a;
}

.drawer-close,
.mobile-topbar,
.drawer-backdrop {
  display: none;
}

.main-area {
  margin-left: var(--sidebar-w);
  transition: margin-left 0.3s var(--ease);
  min-width: 0;
  min-height: 100vh;
  display: flex;
  flex-direction: column;
}

.main-content {
  flex: 1;
  max-width: 1600px;
  width: 100%;
  margin: 0 auto;
  padding: var(--s6) 40px 64px;
}

/* Icon rail */
/* Icon rail: forced at tablet width, opt-in on desktop via the collapse toggle */
.app.rail .sidebar {
  width: var(--rail-w);
  padding: var(--s4) var(--s3);
  align-items: stretch;
}

.app.rail .sidebar .subtitle,
.app.rail .sidebar .nav-label,
.app.rail .sidebar .sidebar-footer .language-label,
.app.rail .sidebar .sidebar-footer .profile-name,
.app.rail .sidebar .sidebar-footer .chevron {
  display: none;
}

.app.rail .sidebar-head {
  justify-content: center;
  padding: 0;
}

/* full name wraps letter-by-letter in the 72px rail */
.app.rail .logo {
  display: none;
}

.app.rail .nav-tabs a {
  justify-content: center;
  padding: var(--s3) 0;
}

.app.rail .sidebar .sidebar-footer .language-button,
.app.rail .sidebar .sidebar-footer .profile-button {
  justify-content: center;
  padding: var(--s2) 0;
}

.app.rail .sidebar .sidebar-footer .dropdown-menu {
  top: auto;
  bottom: 0;
  left: calc(100% + var(--s2));
  right: auto;
}

.app.rail .collapse-toggle {
  justify-content: center;
  padding: var(--s3) 0;
}

.app.rail .main-area {
  margin-left: var(--rail-w);
}

.app.rail .main-content {
  padding: var(--s5);
}

/* Off-canvas drawer */
@media (max-width: 767px) {
  .sidebar {
    transform: translateX(-100%);
    transition: transform 0.2s ease;
    z-index: 300;
  }

  .drawer-open .sidebar {
    transform: none;
  }

  .drawer-close {
    display: inline-flex;
  }

  .drawer-backdrop {
    display: block;
    position: fixed;
    inset: 0;
    background: rgba(0, 0, 0, 0.6);
    backdrop-filter: blur(4px);
    z-index: 250;
  }

  .mobile-topbar {
    display: flex;
    align-items: center;
    gap: var(--s3);
    height: var(--topbar-h);
    padding: 0 var(--s4);
    background: rgba(8, 9, 11, 0.8);
    backdrop-filter: blur(16px);
    -webkit-backdrop-filter: blur(16px);
    border-bottom: 1px solid var(--border);
    position: sticky;
    top: 0;
    z-index: 200;
  }

  .mobile-topbar .brand-mark {
    width: 24px;
    height: 24px;
    border-radius: 7px;
  }

  .mobile-logo {
    font-weight: 600;
    letter-spacing: -0.02em;
    color: var(--text);
    white-space: nowrap;
    overflow: hidden;
    text-overflow: ellipsis;
  }

  .main-area {
    margin-left: 0;
  }

  .main-content {
    padding: var(--s4);
  }
}

/* Route change: quick fade, then the view's sections rise in one after another */
.page-enter-active {
  transition: opacity 0.25s var(--ease);
}

.page-leave-active {
  transition: opacity 0.12s ease;
}

.page-enter-from,
.page-leave-to {
  opacity: 0;
}

@keyframes rise {
  from {
    opacity: 0;
    transform: translateY(10px);
  }
}

.main-content > * > * {
  animation: rise 0.6s var(--ease) both;
}

.main-content > * > *:nth-child(2) { animation-delay: 60ms; }
.main-content > * > *:nth-child(3) { animation-delay: 120ms; }
.main-content > * > *:nth-child(4) { animation-delay: 180ms; }
.main-content > * > *:nth-child(5) { animation-delay: 240ms; }
.main-content > * > *:nth-child(n + 6) { animation-delay: 300ms; }

@media (prefers-reduced-motion: reduce) {
  .sidebar,
  .main-area,
  .page-enter-active,
  .page-leave-active {
    transition: none;
  }

  .main-content > * > * {
    animation: none;
  }
}

/* Shared view classes */
.page-header {
  margin-bottom: 40px;
}

.page-header h2 {
  font-family: var(--font-display);
  font-size: clamp(2rem, 1.4rem + 2vw, 3rem);
  font-weight: 600;
  color: var(--text);
  margin-bottom: var(--s2);
  letter-spacing: -0.045em;
  line-height: 1.05;
}

.page-header p {
  color: var(--text-muted);
  font-size: 1rem;
  max-width: 60ch;
}

.stats-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(min(220px, 100%), 1fr));
  gap: var(--s3);
  margin-bottom: var(--s6);
}

/* KPIs read as type: quiet surface, big tight numerals, colour only when the value has a status */
.stat-card {
  position: relative;
  background: linear-gradient(180deg, rgba(255, 255, 255, 0.025), transparent 60%), var(--card);
  padding: 22px 22px 20px;
  border-radius: var(--r);
  box-shadow: inset 0 1px 0 rgba(255, 255, 255, 0.04);
  transition: background-color 0.25s var(--ease), transform 0.25s var(--ease);
  min-width: 0;
}

.stat-card:hover {
  background-color: var(--card-hover);
  transform: translateY(-1px);
}

.stat-label {
  color: var(--text-muted);
  font-size: 0.875rem;
  font-weight: 500;
  margin-bottom: var(--s3);
}

.stat-value {
  font-family: var(--font-display);
  font-size: clamp(1.75rem, 1.3rem + 1.2vw, 2.5rem);
  font-weight: 600;
  color: var(--text);
  letter-spacing: -0.04em;
  line-height: 1;
  overflow-wrap: anywhere;
}

.stat-card.warning .stat-value {
  color: var(--warning);
}

.stat-card.success .stat-value {
  color: var(--success);
}

.stat-card.danger .stat-value {
  color: var(--danger);
}

.stat-card.info .stat-value {
  color: var(--info);
}

.card {
  background: linear-gradient(180deg, rgba(255, 255, 255, 0.02), transparent 120px), var(--card);
  border-radius: var(--r);
  padding: var(--s5);
  box-shadow: inset 0 1px 0 rgba(255, 255, 255, 0.04);
  margin-bottom: var(--s4);
  min-width: 0;
}

.card-header {
  display: flex;
  flex-wrap: wrap;
  gap: var(--s2);
  justify-content: space-between;
  align-items: center;
  margin-bottom: var(--s5);
}

.card-title {
  font-family: var(--font-display);
  font-size: 1.125rem;
  font-weight: 600;
  color: var(--text);
  letter-spacing: -0.025em;
}

.table-container {
  overflow-x: auto;
  max-width: 100%;
  margin: 0 calc(var(--s2) * -1);
}

table {
  width: 100%;
  border-collapse: separate;
  border-spacing: 0;
}

th {
  text-align: left;
  padding: var(--s3);
  font-weight: 500;
  color: var(--text-muted);
  font-size: 0.8125rem;
  border-bottom: 1px solid var(--border);
  white-space: nowrap;
}

td {
  padding: 14px var(--s3);
  border-bottom: 1px solid var(--border);
  color: var(--text);
  font-size: 0.875rem;
}

td:first-child {
  white-space: nowrap;
}

tbody tr:last-child td {
  border-bottom: none;
}

tbody tr {
  transition: background-color 0.15s var(--ease);
}

tbody tr:hover td {
  background: rgba(255, 255, 255, 0.025);
}

tbody tr:hover td:first-child {
  border-radius: var(--r-sm) 0 0 var(--r-sm);
}

tbody tr:hover td:last-child {
  border-radius: 0 var(--r-sm) var(--r-sm) 0;
}

.badge {
  display: inline-flex;
  align-items: center;
  gap: 6px;
  padding: 4px 10px;
  border-radius: var(--r-pill);
  font-size: 0.75rem;
  font-weight: 500;
  white-space: nowrap;
}

.badge::before {
  content: '';
  width: 6px;
  height: 6px;
  border-radius: 50%;
  background: currentColor;
}

.badge.success,
.badge.increasing {
  background: var(--success-tint);
  color: var(--success);
}

.badge.warning,
.badge.medium {
  background: var(--warning-tint);
  color: var(--warning);
}

.badge.danger,
.badge.decreasing,
.badge.high {
  background: var(--danger-tint);
  color: var(--danger);
}

.badge.info,
.badge.low,
.badge.stable {
  background: var(--info-tint);
  color: var(--info);
}

.loading {
  text-align: center;
  padding: 3rem;
  color: var(--text-muted);
  font-size: 0.938rem;
}

.error {
  background: var(--danger-tint);
  color: var(--danger);
  padding: var(--s4) var(--s5);
  border-radius: var(--r-sm);
  margin: var(--s4) 0;
  font-size: 0.938rem;
}
</style>
