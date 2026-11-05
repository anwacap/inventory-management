<template>
  <aside class="sidebar" :class="{ collapsed: isCollapsed }">
    <!-- Header -->
    <div class="sidebar-header">
      <button
        class="toggle-btn"
        :aria-label="isCollapsed ? 'Expand sidebar' : 'Collapse sidebar'"
        @click="toggleSidebar"
      >
        <span class="toggle-icon">{{ isCollapsed ? '→' : '←' }}</span>
      </button>
      <div v-if="!isCollapsed" class="sidebar-brand">
        <span class="brand-title">{{ t('nav.companyName') }}</span>
        <span class="brand-subtitle">{{ t('nav.subtitle') }}</span>
      </div>
    </div>

    <!-- Navigation -->
    <nav class="sidebar-nav" role="navigation" aria-label="Main navigation">
      <ul>
        <li v-for="item in navItems" :key="item.path">
          <SidebarLink
            :to="item.path"
            :label="item.label"
            :icon="item.icon"
            :collapsed="isCollapsed"
          />
        </li>
      </ul>
    </nav>

    <!-- Footer -->
    <div class="sidebar-footer">
      <button class="theme-toggle" @click="toggleDarkMode" :title="isDarkMode ? 'Light mode' : 'Dark mode'">
        {{ isDarkMode ? '☀️' : '🌙' }}
      </button>
    </div>
  </aside>
</template>

<script setup>
import { ref, onMounted } from 'vue'
import { useI18n } from '../composables/useI18n'
import SidebarLink from './SidebarLink.vue'

defineProps({
  navItems: {
    type: Array,
    required: true
  }
})

const { t } = useI18n()
const isCollapsed = ref(false)
const isDarkMode = ref(false)

const toggleSidebar = () => {
  isCollapsed.value = !isCollapsed.value
  localStorage.setItem('sidebar-collapsed', JSON.stringify(isCollapsed.value))
}

const toggleDarkMode = () => {
  isDarkMode.value = !isDarkMode.value
  localStorage.setItem('dark-mode', JSON.stringify(isDarkMode.value))
  applyTheme()
}

const applyTheme = () => {
  if (isDarkMode.value) {
    document.documentElement.setAttribute('data-theme', 'dark')
  } else {
    document.documentElement.removeAttribute('data-theme')
  }
}

onMounted(() => {
  const savedCollapsed = localStorage.getItem('sidebar-collapsed')
  if (savedCollapsed) {
    isCollapsed.value = JSON.parse(savedCollapsed)
  }

  const savedTheme = localStorage.getItem('dark-mode')
  if (savedTheme) {
    isDarkMode.value = JSON.parse(savedTheme)
    applyTheme()
  }
})
</script>

<style scoped>
.sidebar {
  --sidebar-width: 260px;
  --sidebar-collapsed-width: 80px;

  position: fixed;
  left: 0;
  top: 0;
  height: 100vh;
  width: var(--sidebar-width);
  background: var(--color-bg-sidebar, #ffffff);
  border-right: 1px solid var(--color-border-sidebar, #e2e8f0);
  display: flex;
  flex-direction: column;
  transition: width 0.3s ease;
  z-index: 1001;
  overflow-y: auto;
}

.sidebar.collapsed {
  width: var(--sidebar-collapsed-width);
}

.sidebar-header {
  padding: 16px;
  border-bottom: 1px solid var(--color-border-sidebar, #e2e8f0);
  display: flex;
  align-items: center;
  gap: 12px;
  min-height: 70px;
}

.toggle-btn {
  background: none;
  border: none;
  color: var(--color-text-sidebar, #64748b);
  cursor: pointer;
  padding: 6px 8px;
  font-size: 18px;
  border-radius: 6px;
  transition: background 0.2s;
  flex-shrink: 0;
}

.toggle-btn:hover {
  background: var(--color-bg-hover, #f1f5f9);
}

.sidebar-brand {
  display: flex;
  flex-direction: column;
  gap: 2px;
  white-space: nowrap;
}

.brand-title {
  font-weight: 700;
  color: var(--color-text-sidebar, #0f172a);
  font-size: 14px;
}

.brand-subtitle {
  font-size: 11px;
  color: var(--color-text-muted, #64748b);
}

.sidebar-nav {
  flex: 1;
  padding: 12px 8px;
  overflow-y: auto;
}

.sidebar-nav ul {
  list-style: none;
  margin: 0;
  padding: 0;
}

.sidebar-footer {
  padding: 12px 8px;
  border-top: 1px solid var(--color-border-sidebar, #e2e8f0);
  display: flex;
  gap: 8px;
}

.theme-toggle {
  background: none;
  border: 1px solid var(--color-border-sidebar, #e2e8f0);
  padding: 6px 12px;
  border-radius: 6px;
  cursor: pointer;
  font-size: 14px;
  color: var(--color-text-sidebar, #64748b);
  width: 100%;
  transition: all 0.2s;
}

.theme-toggle:hover {
  background: var(--color-bg-hover, #f1f5f9);
}

@media (max-width: 768px) {
  .sidebar {
    width: var(--sidebar-collapsed-width);
  }

  .sidebar-brand {
    display: none;
  }
}
</style>
