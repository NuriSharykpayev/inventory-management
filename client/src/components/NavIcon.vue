<script setup>
import { computed } from 'vue'

const props = defineProps({
  name: { type: String, required: true }
})

// Stroke-only path data on a 24x24 grid; rendered at 20px
const ICONS = {
  overview: ['M4 4h7v7H4z', 'M13 4h7v7h-7z', 'M4 13h7v7H4z', 'M13 13h7v7h-7z'],
  inventory: ['M21 8l-9-5-9 5v8l9 5 9-5z', 'M3 8l9 5 9-5', 'M12 13v8'],
  orders: ['M7 3h10a1 1 0 0 1 1 1v17l-3-2-3 2-3-2-3 2V4a1 1 0 0 1 1-1z', 'M9 8h6', 'M9 12h6'],
  finance: ['M12 3v18', 'M16 7.5c-.8-1-2.2-1.5-4-1.5-2.2 0-4 1-4 2.8 0 4 8 2.2 8 6.2 0 1.8-1.8 2.8-4 2.8-1.8 0-3.2-.6-4-1.6'],
  demand: ['M3 17l6-6 4 4 8-8', 'M15 7h6v6'],
  reports: ['M4 20V10', 'M10 20V4', 'M16 20v-7', 'M22 20H2'],
  menu: ['M4 6h16', 'M4 12h16', 'M4 18h16'],
  'chevron-left': ['M15 6l-6 6 6 6'],
  'chevron-right': ['M9 6l6 6-6 6']
}

const paths = computed(() => ICONS[props.name] || [])
</script>

<template>
  <svg
    class="nav-icon"
    width="20"
    height="20"
    viewBox="0 0 24 24"
    fill="none"
    stroke="currentColor"
    stroke-width="1.75"
    stroke-linecap="round"
    stroke-linejoin="round"
    aria-hidden="true"
  >
    <path v-for="d in paths" :key="d" :d="d" />
  </svg>
</template>

<style scoped>
.nav-icon {
  flex-shrink: 0;
}
</style>
