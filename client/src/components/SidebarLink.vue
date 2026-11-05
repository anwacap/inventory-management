<template>
  <router-link :to="to" class="sidebar-link" :class="{ active: isActive }" :aria-current="isActive ? 'page' : undefined">
    <span class="link-icon">{{ icon }}</span>
    <span v-if="!collapsed" class="link-label">{{ label }}</span>
  </router-link>
</template>

<script setup>
import { computed } from 'vue'
import { useRoute } from 'vue-router'

const props = defineProps({
  to: String,
  label: String,
  icon: String,
  collapsed: Boolean
})

const route = useRoute()
const isActive = computed(() => route.path === props.to || route.path.startsWith(props.to + '/'))
</script>

<style scoped>
.sidebar-link {
  display: flex;
  align-items: center;
  gap: 12px;
  padding: 10px 12px;
  border-radius: 8px;
  color: var(--color-text-sidebar, #64748b);
  text-decoration: none;
  font-size: 14px;
  font-weight: 500;
  transition: all 0.2s ease;
  white-space: nowrap;
  margin-bottom: 4px;
}

.sidebar-link:hover {
  background: var(--color-bg-hover, #f1f5f9);
  color: var(--color-text-primary, #0f172a);
}

.sidebar-link.active {
  background: var(--color-primary-light, #dbeafe);
  color: var(--color-primary, #2563eb);
  border-left: 3px solid var(--color-primary, #2563eb);
  padding-left: 9px;
}

.link-icon {
  font-size: 18px;
  flex-shrink: 0;
  display: inline-flex;
  align-items: center;
  justify-content: center;
  min-width: 24px;
}

.link-label {
  overflow: hidden;
  text-overflow: ellipsis;
  flex: 1;
}

[data-theme="dark"] .sidebar-link.active {
  background: #1e40af;
  color: #3b82f6;
}
</style>
