<template>
  <div class="app-shell">

    <!-- ===== Left Sidebar ===== -->
    <aside class="sidebar" :class="{ collapsed: sidebarCollapsed }">

      <!-- Logo row + collapse toggle -->
      <div class="sidebar-logo">
        <span class="logo-mark">CC</span>
        <div class="logo-text-group">
          <span class="logo-name">{{ t('nav.companyName') }}</span>
          <span class="logo-sub">{{ t('nav.subtitle') }}</span>
        </div>
        <button class="sidebar-toggle" @click="sidebarCollapsed = !sidebarCollapsed" :title="sidebarCollapsed ? 'Expand sidebar' : 'Collapse sidebar'">
          <!-- Right-pointing chevron rotates 180° when expanded -->
          <svg class="toggle-icon" viewBox="0 0 20 20" fill="currentColor" xmlns="http://www.w3.org/2000/svg">
            <path fill-rule="evenodd" d="M7.293 14.707a1 1 0 010-1.414L10.586 10 7.293 6.707a1 1 0 011.414-1.414l4 4a1 1 0 010 1.414l-4 4a1 1 0 01-1.414 0z" clip-rule="evenodd"/>
          </svg>
        </button>
      </div>

      <nav class="sidebar-nav">
        <router-link
          v-for="item in navItems"
          :key="item.to"
          :to="item.to"
          class="sidebar-link"
          :class="{ active: $route.path === item.to }"
          :title="sidebarCollapsed ? item.label : ''"
        >
          <!-- Two-letter abbreviation shown in collapsed mode -->
          <span class="nav-icon">{{ item.abbr }}</span>
          <span class="nav-label">{{ item.label }}</span>
        </router-link>
      </nav>

      <!-- Footer controls: hidden in collapsed mode -->
      <div class="sidebar-footer">
        <LanguageSwitcher />
        <ProfileMenu
          @show-profile-details="showProfileDetails = true"
          @show-tasks="showTasks = true"
        />
      </div>
    </aside>

    <!-- ===== Main Area (shifts with sidebar) ===== -->
    <div class="main-area" :style="{ marginLeft: sidebarCollapsed ? '56px' : '220px' }">
      <FilterBar />
      <main class="main-content">
        <router-view />
      </main>
    </div>

    <!-- Modals -->
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
import { ref, computed, onMounted, onUnmounted } from 'vue'
import { api } from './api'
import { useAuth } from './composables/useAuth'
import { useI18n } from './composables/useI18n'
import FilterBar from './components/FilterBar.vue'
import ProfileMenu from './components/ProfileMenu.vue'
import ProfileDetailsModal from './components/ProfileDetailsModal.vue'
import TasksModal from './components/TasksModal.vue'
import LanguageSwitcher from './components/LanguageSwitcher.vue'

export default {
  name: 'App',
  components: {
    FilterBar,
    ProfileMenu,
    ProfileDetailsModal,
    TasksModal,
    LanguageSwitcher
  },
  setup() {
    const { currentUser } = useAuth()
    const { t } = useI18n()
    const showProfileDetails = ref(false)
    const showTasks = ref(false)
    const apiTasks = ref([])
    const sidebarCollapsed = ref(false)

    // Nav items with 2-letter abbreviations used as icons when collapsed
    const navItems = computed(() => [
      { to: '/',           label: t('nav.overview'),       abbr: 'Ov' },
      { to: '/inventory',  label: t('nav.inventory'),      abbr: 'Iv' },
      { to: '/orders',     label: t('nav.orders'),         abbr: 'Or' },
      { to: '/spending',   label: t('nav.finance'),        abbr: 'Fi' },
      { to: '/demand',     label: t('nav.demandForecast'), abbr: 'Dm' },
      { to: '/restocking', label: t('nav.restocking'),     abbr: 'Rs' },
      { to: '/reports',    label: 'Reports',               abbr: 'Rp' },
    ])

    // Auto-collapse on narrow screens (<= 1024px)
    const checkScreenSize = () => {
      sidebarCollapsed.value = window.innerWidth <= 1024
    }

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
          const index = currentUser.value.tasks.findIndex(t => t.id === taskId)
          if (index !== -1) currentUser.value.tasks.splice(index, 1)
        } else {
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
          mockTask.status = mockTask.status === 'pending' ? 'completed' : 'pending'
        } else {
          const updatedTask = await api.toggleTask(taskId)
          const index = apiTasks.value.findIndex(t => t.id === taskId)
          if (index !== -1) apiTasks.value[index] = updatedTask
        }
      } catch (err) {
        console.error('Failed to toggle task:', err)
      }
    }

    onMounted(() => {
      loadTasks()
      checkScreenSize()
      window.addEventListener('resize', checkScreenSize)
    })

    onUnmounted(() => {
      window.removeEventListener('resize', checkScreenSize)
    })

    return {
      t,
      showProfileDetails,
      showTasks,
      tasks,
      sidebarCollapsed,
      navItems,
      addTask,
      deleteTask,
      toggleTask
    }
  }
}
</script>

<style>
/* ===== Reset ===== */
* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

body {
  font-family: 'Inter', -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Oxygen, Ubuntu, Cantarell, sans-serif;
  background: #f8fafc;
  color: #1e293b;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
}

/* ===== App shell ===== */
.app-shell {
  display: flex;
  min-height: 100vh;
}

/* ===== Sidebar ===== */
.sidebar {
  width: 220px;
  min-width: 220px;
  background: #0f172a;
  display: flex;
  flex-direction: column;
  position: fixed;
  top: 0;
  left: 0;
  bottom: 0;
  z-index: 100;
  overflow: hidden;        /* clip content during transition */
  border-right: 1px solid #1e293b;
  transition: width 0.22s ease, min-width 0.22s ease;
}

/* ===== Collapsed state ===== */
.sidebar.collapsed {
  width: 56px;
  min-width: 56px;
}

/* Hide text elements when collapsed */
.sidebar.collapsed .logo-text-group,
.sidebar.collapsed .nav-label,
.sidebar.collapsed .sidebar-footer {
  opacity: 0;
  pointer-events: none;
  width: 0;
  overflow: hidden;
}

/* Center the logo-mark when collapsed */
.sidebar.collapsed .sidebar-logo {
  justify-content: center;
  padding: 0.875rem 0;
  gap: 0;
}

/* Center links when collapsed */
.sidebar.collapsed .sidebar-link {
  justify-content: center;
  padding: 0.75rem 0;
}

/* Show abbr icons when collapsed */
.sidebar.collapsed .nav-icon {
  display: flex;
}

/* Flip toggle arrow direction when collapsed */
.sidebar.collapsed .toggle-icon {
  transform: rotate(180deg);
}

/* ===== Logo row ===== */
.sidebar-logo {
  display: flex;
  align-items: center;
  gap: 0.75rem;
  padding: 1.125rem 0.875rem 1.125rem 1rem;
  border-bottom: 1px solid #1e293b;
  flex-shrink: 0;
}

/* Blue monogram square — stands in as an icon without any library */
.logo-mark {
  width: 32px;
  height: 32px;
  min-width: 32px;
  background: #3b82f6;
  border-radius: 8px;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 0.75rem;
  font-weight: 700;
  color: #fff;
  letter-spacing: 0.025em;
  flex-shrink: 0;
}

.logo-text-group {
  display: flex;
  flex-direction: column;
  gap: 0.125rem;
  flex: 1;
  min-width: 0;
  overflow: hidden;
  transition: opacity 0.15s ease, width 0.22s ease;
}

.logo-name {
  font-size: 0.875rem;
  font-weight: 700;
  color: #f1f5f9;
  letter-spacing: -0.015em;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.logo-sub {
  font-size: 0.688rem;
  color: #475569;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

/* Collapse/expand toggle button */
.sidebar-toggle {
  flex-shrink: 0;
  margin-left: auto;
  width: 24px;
  height: 24px;
  display: flex;
  align-items: center;
  justify-content: center;
  background: transparent;
  border: 1px solid #334155;
  border-radius: 5px;
  color: #64748b;
  cursor: pointer;
  transition: background 0.15s ease, color 0.15s ease, border-color 0.15s ease;
  padding: 0;
}

.sidebar-toggle:hover {
  background: #1e293b;
  color: #e2e8f0;
  border-color: #475569;
}

.toggle-icon {
  width: 14px;
  height: 14px;
  transition: transform 0.22s ease;
}

/* ===== Nav links ===== */
.sidebar-nav {
  flex: 1;
  padding: 0.75rem 0.5rem;
  display: flex;
  flex-direction: column;
  gap: 0.125rem;
  overflow-y: auto;
  overflow-x: hidden;
}

.sidebar-link {
  display: flex;
  align-items: center;
  gap: 0.75rem;
  padding: 0.563rem 0.75rem;
  color: #94a3b8;
  text-decoration: none;
  font-size: 0.875rem;
  font-weight: 500;
  border-radius: 6px;
  transition: color 0.15s ease, background 0.15s ease;
  position: relative;
  white-space: nowrap;
  overflow: hidden;
}

.sidebar-link:hover {
  color: #e2e8f0;
  background: rgba(255, 255, 255, 0.06);
}

.sidebar-link.active {
  color: #ffffff;
  background: #1e293b;
}

/* Blue accent bar on active link */
.sidebar-link.active::before {
  content: '';
  position: absolute;
  left: 0;
  top: 50%;
  transform: translateY(-50%);
  width: 3px;
  height: 55%;
  background: #3b82f6;
  border-radius: 0 2px 2px 0;
}

/* 2-letter icon shown only in collapsed mode */
.nav-icon {
  display: none;
  width: 24px;
  min-width: 24px;
  height: 24px;
  align-items: center;
  justify-content: center;
  font-size: 0.688rem;
  font-weight: 700;
  letter-spacing: 0.02em;
  border-radius: 5px;
  background: rgba(255, 255, 255, 0.08);
  flex-shrink: 0;
}

.sidebar-link.active .nav-icon {
  background: rgba(59, 130, 246, 0.25);
  color: #93c5fd;
}

.nav-label {
  flex: 1;
  overflow: hidden;
  text-overflow: ellipsis;
  transition: opacity 0.15s ease;
}

/* ===== Sidebar footer ===== */
.sidebar-footer {
  padding: 0.75rem;
  border-top: 1px solid #1e293b;
  display: flex;
  align-items: center;
  justify-content: flex-end;
  gap: 0.5rem;
  flex-shrink: 0;
  transition: opacity 0.15s ease;
}

/* ===== Main area ===== */
.main-area {
  flex: 1;
  margin-left: 220px;  /* overridden by inline :style binding */
  display: flex;
  flex-direction: column;
  min-height: 100vh;
  min-width: 0;
  transition: margin-left 0.22s ease;
}

.main-content {
  flex: 1;
  max-width: 1600px;
  width: 100%;
  margin: 0 auto;
  padding: 1.5rem 2rem;
}

/* ===== Page header ===== */
.page-header {
  margin-bottom: 1.5rem;
}

.page-header h2 {
  font-size: 1.875rem;
  font-weight: 700;
  color: #0f172a;
  margin-bottom: 0.375rem;
  letter-spacing: -0.025em;
}

.page-header p {
  color: #64748b;
  font-size: 0.938rem;
}

/* ===== Stat cards ===== */
.stats-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(200px, 1fr));
  gap: 1rem;
  margin-bottom: 1.5rem;
}

.stat-card {
  background: white;
  padding: 1.25rem 1.5rem;
  border-radius: 12px;
  border: 1px solid #e2e8f0;
  transition: box-shadow 0.2s ease, border-color 0.2s ease;
}

.stat-card:hover {
  border-color: #cbd5e1;
  box-shadow: 0 4px 16px rgba(0, 0, 0, 0.06);
}

.stat-label {
  color: #64748b;
  font-size: 0.75rem;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.06em;
  margin-bottom: 0.5rem;
}

.stat-value {
  font-size: 2rem;
  font-weight: 700;
  color: #0f172a;
  letter-spacing: -0.03em;
  line-height: 1;
}

.stat-card.warning .stat-value { color: #ea580c; }
.stat-card.success .stat-value { color: #059669; }
.stat-card.danger  .stat-value { color: #dc2626; }
.stat-card.info    .stat-value { color: #2563eb; }

/* ===== Cards ===== */
.card {
  background: white;
  border-radius: 12px;
  padding: 1.25rem 1.5rem;
  border: 1px solid #e2e8f0;
  margin-bottom: 1.25rem;
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.04);
}

.card-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 1rem;
  padding-bottom: 0.875rem;
  border-bottom: 1px solid #f1f5f9;
}

.card-title {
  font-size: 1rem;
  font-weight: 700;
  color: #0f172a;
  letter-spacing: -0.015em;
}

/* ===== Tables ===== */
.table-container {
  overflow-x: auto;
}

table {
  width: 100%;
  border-collapse: collapse;
}

thead {
  background: #f8fafc;
  border-top: 1px solid #f1f5f9;
  border-bottom: 1px solid #e2e8f0;
}

th {
  text-align: left;
  padding: 0.625rem 0.75rem;
  font-weight: 600;
  color: #64748b;
  font-size: 0.688rem;
  text-transform: uppercase;
  letter-spacing: 0.07em;
}

td {
  padding: 0.625rem 0.75rem;
  border-top: 1px solid #f8fafc;
  color: #334155;
  font-size: 0.875rem;
}

tbody tr {
  transition: background-color 0.1s ease;
}

tbody tr:hover {
  background: #f8fafc;
}

/* ===== Badges ===== */
.badge {
  display: inline-block;
  padding: 0.25rem 0.625rem;
  border-radius: 5px;
  font-size: 0.688rem;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.04em;
}

.badge.success    { background: #dcfce7; color: #166534; }
.badge.warning    { background: #fef9c3; color: #854d0e; }
.badge.danger     { background: #fee2e2; color: #991b1b; }
.badge.info       { background: #dbeafe; color: #1e40af; }
.badge.increasing { background: #dcfce7; color: #166534; }
.badge.decreasing { background: #fee2e2; color: #991b1b; }
.badge.stable     { background: #e0e7ff; color: #3730a3; }
.badge.high       { background: #fee2e2; color: #991b1b; }
.badge.medium     { background: #fef9c3; color: #854d0e; }
.badge.low        { background: #dbeafe; color: #1e40af; }

/* ===== Loading / Error ===== */
.loading {
  text-align: center;
  padding: 3rem;
  color: #94a3b8;
  font-size: 0.938rem;
}

.error {
  background: #fef2f2;
  border: 1px solid #fecaca;
  color: #991b1b;
  padding: 1rem 1.25rem;
  border-radius: 10px;
  margin: 1rem 0;
  font-size: 0.875rem;
}
</style>
