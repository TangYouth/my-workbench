<script setup lang="ts">
import CodexPanel from './CodexPanel.vue'
import TodoPanel from './TodoPanel.vue'

defineProps<{
  activePanel: 'codex' | 'todo'
}>()
</script>

<template>
  <CodexPanel v-if="activePanel === 'codex'" />
  <TodoPanel v-else />
</template>
