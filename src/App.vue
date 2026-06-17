<script setup lang="ts">
import { computed, onBeforeUnmount, onMounted, ref } from 'vue'
import WorkbenchCenter from './components/WorkbenchCenter.vue'

const navItems = [{ id: 'codex', label: 'Codex', icon: '>_' }]

const now = ref(new Date())
const onlineVisible = ref(true)
let clockTimer: number | undefined
let blinkTimer: number | undefined
let blinkResetTimer: number | undefined

const timeText = computed(() => {
  return new Intl.DateTimeFormat('zh-CN', {
    hour: '2-digit',
    minute: '2-digit',
    second: '2-digit',
    hour12: false,
  }).format(now.value)
})

const scheduleOnlineBlink = () => {
  const idleDelay = 3000 + Math.random() * 7000

  blinkTimer = window.setTimeout(() => {
    onlineVisible.value = false

    blinkResetTimer = window.setTimeout(() => {
      onlineVisible.value = true
      scheduleOnlineBlink()
    }, 320)
  }, idleDelay)
}

onMounted(() => {
  now.value = new Date()
  clockTimer = window.setInterval(() => {
    now.value = new Date()
  }, 1000)
  scheduleOnlineBlink()
})

onBeforeUnmount(() => {
  if (clockTimer) window.clearInterval(clockTimer)
  if (blinkTimer) window.clearTimeout(blinkTimer)
  if (blinkResetTimer) window.clearTimeout(blinkResetTimer)
})
</script>

<template>
  <div class="workbench-shell">
    <div class="workbench-shell__noise"></div>

    <header class="workbench-header">
      <div class="brand-block">
        <div class="brand-block__icon">
          <span>&gt;_</span>
        </div>
        <div>
          <p class="brand-block__eyebrow">SYSTEM PANEL</p>
          <h1 class="brand-block__title">我的工作台</h1>
        </div>
      </div>

      <div class="status-bar" :class="{ 'status-bar--hidden': !onlineVisible }">
        <span class="status-bar__dot"></span>
        <span class="status-bar__label">ONLINE</span>
        <span class="status-bar__divider"></span>
        <span class="status-bar__time">{{ timeText }}</span>
      </div>
    </header>

    <div class="workbench-body">
      <aside class="terminal-sidebar">
        <div class="terminal-sidebar__frame">
          <p class="terminal-sidebar__label">CHANNELS</p>

          <button
            v-for="item in navItems"
            :key="item.id"
            type="button"
            class="terminal-nav terminal-nav--active"
          >
            <span class="terminal-nav__icon">{{ item.icon }}</span>
            <span>{{ item.label }}</span>
          </button>
        </div>
      </aside>

      <WorkbenchCenter />
    </div>
  </div>
</template>

<style scoped>
:global(body) {
  margin: 0;
  min-width: 320px;
  background:
    radial-gradient(circle at top, rgba(25, 255, 102, 0.12), transparent 30%),
    radial-gradient(circle at right, rgba(0, 217, 255, 0.08), transparent 24%),
    #020605;
  color: #e8fff0;
  font-family:
    'SFMono-Regular',
    'Menlo',
    'Monaco',
    'Cascadia Mono',
    'Segoe UI Mono',
    'Roboto Mono',
    monospace;
}

:global(#app) {
  min-height: 100vh;
}

.workbench-shell {
  position: relative;
  display: grid;
  grid-template-rows: auto minmax(0, 1fr);
  height: 100svh;
  overflow: hidden;
  background-image:
    linear-gradient(rgba(27, 255, 129, 0.07) 1px, transparent 1px),
    linear-gradient(90deg, rgba(27, 255, 129, 0.07) 1px, transparent 1px);
  background-size: 56px 56px;
}

.workbench-shell__noise {
  position: absolute;
  inset: 0;
  pointer-events: none;
  background:
    linear-gradient(180deg, rgba(255, 255, 255, 0.02), transparent 20%, transparent 80%, rgba(255, 255, 255, 0.02)),
    radial-gradient(circle at 20% 20%, rgba(25, 255, 102, 0.08), transparent 18%);
  mix-blend-mode: screen;
}

.workbench-header,
.workbench-body {
  position: relative;
  z-index: 1;
}

.workbench-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 1.5rem;
  padding: 1rem 1.75rem 0.9rem;
  border-bottom: 1px solid rgba(109, 255, 154, 0.16);
}

.brand-block {
  display: flex;
  align-items: center;
  gap: 0.85rem;
}

.brand-block__icon {
  display: grid;
  place-items: center;
  width: 2.8rem;
  height: 2.8rem;
  border: 1px solid rgba(33, 255, 118, 0.7);
  border-radius: 0.8rem;
  color: #46ff83;
  font-size: 1.3rem;
  box-shadow:
    0 0 0 1px rgba(13, 70, 29, 0.85) inset,
    0 0 24px rgba(38, 255, 121, 0.18);
}

.brand-block__eyebrow {
  margin-bottom: 0.25rem;
  color: rgba(115, 255, 167, 0.6);
  font-size: 0.75rem;
  letter-spacing: 0.24em;
}

.brand-block__title {
  color: #37ff74;
  font-size: clamp(1.8rem, 2.6vw, 2.7rem);
  font-weight: 700;
  letter-spacing: 0.04em;
  text-shadow: 0 0 18px rgba(55, 255, 116, 0.22);
}

.status-bar {
  display: flex;
  align-items: center;
  gap: 0.9rem;
  color: #f3fff5;
  font-size: 1rem;
  transition: opacity 0.28s ease;
}

.status-bar--hidden {
  opacity: 0.28;
}

.status-bar__dot {
  width: 0.85rem;
  height: 0.85rem;
  border-radius: 999px;
  background: #27ff68;
  box-shadow: 0 0 18px rgba(39, 255, 104, 0.9);
}

.status-bar__label {
  color: #27ff68;
  letter-spacing: 0.08em;
}

.status-bar__divider {
  width: 1px;
  height: 1.25rem;
  background: rgba(255, 255, 255, 0.18);
}

.workbench-body {
  display: grid;
  grid-template-columns: 260px minmax(0, 1fr);
  gap: 1.5rem;
  padding: 1.5rem 1.75rem 1.6rem;
  min-height: 0;
  height: 100%;
  overflow: hidden;
}

.terminal-sidebar {
  min-width: 0;
}

.terminal-sidebar__frame {
  height: 100%;
  padding: 0.9rem;
  border: 1px solid rgba(67, 255, 130, 0.4);
  border-radius: 1.1rem;
  background: linear-gradient(180deg, rgba(3, 20, 9, 0.92), rgba(1, 9, 6, 0.9));
  box-shadow:
    0 0 0 1px rgba(27, 107, 54, 0.35) inset,
    0 0 35px rgba(0, 0, 0, 0.32);
}

.terminal-sidebar__label {
  margin-bottom: 0.9rem;
  color: rgba(154, 255, 188, 0.48);
  font-size: 0.72rem;
  letter-spacing: 0.28em;
}

.terminal-nav {
  display: flex;
  align-items: center;
  gap: 0.9rem;
  width: 100%;
  padding: 1.15rem 1rem;
  border: 1px solid rgba(39, 255, 104, 0.16);
  border-radius: 0.85rem;
  background: transparent;
  color: #9dffb9;
  font: inherit;
  text-align: left;
}

.terminal-nav--active {
  background: linear-gradient(90deg, rgba(22, 88, 37, 0.78), rgba(9, 40, 18, 0.92));
  color: #ecfff2;
  box-shadow:
    0 0 0 1px rgba(62, 255, 127, 0.14) inset,
    0 0 28px rgba(18, 181, 74, 0.12);
}

.terminal-nav__icon {
  color: #30ff6d;
  font-size: 1.2rem;
}

@media (max-width: 1080px) {
  .workbench-body {
    grid-template-columns: 1fr;
    min-height: auto;
    height: auto;
  }

  .terminal-sidebar__frame {
    min-height: auto;
  }
}

@media (max-width: 720px) {
  .workbench-header,
  .workbench-body {
    padding: 1.25rem;
  }

  .workbench-header {
    flex-direction: column;
    align-items: flex-start;
  }

  .status-bar {
    font-size: 0.92rem;
  }
}
</style>
