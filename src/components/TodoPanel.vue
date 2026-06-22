<script setup lang="ts">
import { computed, onMounted, ref, watch } from 'vue'

type TaskStatus = 'active' | 'done'
type TaskPriority = 'HIGH' | 'MEDIUM' | 'LOW'
type FilterId = 'all' | 'active' | 'done' | 'high'

type TaskItem = {
  id: number
  title: string
  status: TaskStatus
  priority: TaskPriority
  completedAt?: number
}

const filters: { id: FilterId; label: string }[] = [
  { id: 'all', label: '全部' },
  { id: 'active', label: '进行中' },
  { id: 'done', label: '已完成' },
  { id: 'high', label: '高优先级' },
]

const DONE_CLEANUP_DELAY = 120 * 60 * 60 * 1000
const STORAGE_KEY = 'codex-workbench.todo.tasks'

const tasks = ref<TaskItem[]>([])
const activeFilter = ref<FilterId>('all')
const selectedTaskId = ref(0)
const newTaskTitle = ref('')
let nextTaskId = 1
const totalTasks = computed(() => tasks.value.length)
const doneTasks = computed(() => tasks.value.filter((task) => task.status === 'done').length)
const activeTasks = computed(() => tasks.value.filter((task) => task.status === 'active').length)
const highPriorityTasks = computed(() => tasks.value.filter((task) => task.priority === 'HIGH').length)

const completionPercent = computed(() => {
  if (!totalTasks.value) {
    return 0
  }

  return Math.round((doneTasks.value / totalTasks.value) * 100)
})

const selectedTask = computed(() => {
  return tasks.value.find((task) => task.id === selectedTaskId.value) ?? tasks.value[0] ?? null
})

const filteredTasks = computed(() => {
  if (activeFilter.value === 'active') {
    return tasks.value.filter((task) => task.status === 'active')
  }

  if (activeFilter.value === 'done') {
    return tasks.value.filter((task) => task.status === 'done')
  }

  if (activeFilter.value === 'high') {
    return tasks.value.filter((task) => task.priority === 'HIGH')
  }

  return tasks.value
})

const getFilterCount = (filterId: FilterId) => {
  if (filterId === 'active') return activeTasks.value
  if (filterId === 'done') return doneTasks.value
  if (filterId === 'high') return highPriorityTasks.value
  return totalTasks.value
}

const addTask = () => {
  const title = newTaskTitle.value.trim()

  if (!title) {
    return
  }

  const task: TaskItem = {
    id: nextTaskId,
    title,
    status: 'active',
    priority: 'MEDIUM',
  }

  tasks.value.unshift(task)
  selectedTaskId.value = task.id
  newTaskTitle.value = ''
  nextTaskId += 1
}

const toggleTask = (task: TaskItem) => {
  task.status = task.status === 'done' ? 'active' : 'done'
  selectedTaskId.value = task.id

  if (task.status === 'done') {
    task.completedAt = Date.now()
    return
  }

  task.completedAt = undefined
}

const cyclePriority = (task: TaskItem) => {
  const order: TaskPriority[] = ['LOW', 'MEDIUM', 'HIGH']
  const nextIndex = (order.indexOf(task.priority) + 1) % order.length

  task.priority = order[nextIndex] ?? 'LOW'
  selectedTaskId.value = task.id
}

const deleteTask = (taskId: number) => {
  tasks.value = tasks.value.filter((task) => task.id !== taskId)

  if (selectedTaskId.value === taskId) {
    selectedTaskId.value = tasks.value[0]?.id ?? 0
  }
}

const loadTasks = () => {
  const storedTasks = window.localStorage.getItem(STORAGE_KEY)

  if (!storedTasks) {
    return
  }

  try {
    const parsedTasks = JSON.parse(storedTasks) as TaskItem[]

    if (!Array.isArray(parsedTasks)) {
      return
    }

    tasks.value = removeExpiredCompletedTasks(parsedTasks.filter(isValidTask))
    selectedTaskId.value = tasks.value[0]?.id ?? 0
    nextTaskId = Math.max(0, ...tasks.value.map((task) => task.id)) + 1
  } catch {
    tasks.value = []
  }
}

const removeExpiredCompletedTasks = (items: TaskItem[]) => {
  const now = Date.now()

  return items.filter((task) => {
    if (task.status !== 'done') {
      return true
    }

    return typeof task.completedAt === 'number' && now - task.completedAt < DONE_CLEANUP_DELAY
  })
}

const isValidTask = (value: unknown): value is TaskItem => {
  if (!value || typeof value !== 'object') {
    return false
  }

  const task = value as TaskItem

  return (
    typeof task.id === 'number' &&
    typeof task.title === 'string' &&
    (task.status === 'active' || task.status === 'done') &&
    (task.priority === 'HIGH' || task.priority === 'MEDIUM' || task.priority === 'LOW')
  )
}

watch(
  tasks,
  (nextTasks) => {
    window.localStorage.setItem(STORAGE_KEY, JSON.stringify(nextTasks))
  },
  { deep: true },
)

onMounted(() => {
  loadTasks()
})
</script>

<template>
  <main class="todo-workspace">
    <section class="panel todo-panel" aria-labelledby="todo-title">
      <header class="panel-header">
        <div class="panel-title-group">
          <span class="panel-index">03</span>
          <div>
            <p class="panel-kicker">TODO MANAGER</p>
            <h2 id="todo-title">待办列表</h2>
          </div>
        </div>
        <button type="button" class="symbol-button" aria-label="运行任务扫描"> &gt; </button>
      </header>

      <form class="task-entry" @submit.prevent="addTask">
        <input v-model="newTaskTitle" type="text" placeholder="输入新的待办事项..." />
        <button type="submit">
          <span>添加任务</span>
          <span aria-hidden="true">+</span>
        </button>
      </form>

      <nav class="filter-tabs" aria-label="任务筛选">
        <button
          v-for="filter in filters"
          :key="filter.id"
          type="button"
          :class="{ 'filter-tabs__item--active': activeFilter === filter.id }"
          class="filter-tabs__item"
          @click="activeFilter = filter.id"
        >
          <span>{{ filter.label }}</span>
          <strong>{{ getFilterCount(filter.id) }}</strong>
        </button>
      </nav>

      <div class="task-list" aria-live="polite">
        <article
          v-for="task in filteredTasks"
          :key="task.id"
          class="task-card"
          :class="{
            'task-card--selected': selectedTask?.id === task.id,
            'task-card--done': task.status === 'done',
          }"
          @click="selectedTaskId = task.id"
        >
          <button
            type="button"
            class="status-toggle"
            :aria-label="task.status === 'done' ? '标记为进行中' : '标记为完成'"
            @click.stop="toggleTask(task)"
          >
            <span v-if="task.status === 'done'">✓</span>
          </button>
          <h3>{{ task.title }}</h3>
          <button
            type="button"
            class="priority-badge"
            :class="`priority-badge--${task.priority.toLowerCase()}`"
            @click.stop="cyclePriority(task)"
          >
            {{ task.priority }}
          </button>
        </article>

        <p v-if="!filteredTasks.length" class="empty-state">当前筛选没有任务</p>
      </div>

      <footer class="task-summary">
        <span>共 <strong>{{ totalTasks }}</strong> 项任务</span>
        <span>进行中 <strong>{{ activeTasks }}</strong> 项</span>
        <span>已完成 <strong>{{ doneTasks }}</strong> 项</span>
      </footer>
    </section>

    <aside class="info-column">
      <section class="panel status-panel" aria-labelledby="weekly-title">
        <header class="panel-header">
          <div class="panel-title-group">
            <span class="panel-index panel-index--cyan">04</span>
            <div>
              <p class="panel-kicker">CURRENT STATUS</p>
              <h2 id="weekly-title">当前进度</h2>
            </div>
          </div>
          <button type="button" class="symbol-button symbol-button--cyan" aria-label="查看统计">▥</button>
        </header>

        <div class="metric-grid">
          <div class="metric-card">
            <span>总任务</span>
            <strong>{{ totalTasks }}</strong>
            <small>项</small>
          </div>
          <div class="metric-card metric-card--green">
            <span>已完成</span>
            <strong>{{ doneTasks }}</strong>
            <small>项</small>
          </div>
          <div class="metric-card metric-card--cyan">
            <span>进行中</span>
            <strong>{{ activeTasks }}</strong>
            <small>项</small>
          </div>
          <div class="metric-card">
            <span>完成率</span>
            <strong>{{ completionPercent }}%</strong>
          </div>
        </div>

        <div class="progress-row">
          <span>进度概览</span>
          <strong>{{ doneTasks }} / {{ totalTasks }} 已完成</strong>
        </div>
        <div class="progress-track">
          <span :style="{ width: `${completionPercent}%` }"></span>
        </div>
      </section>

      <section class="panel details-panel" aria-labelledby="details-title">
        <header class="panel-header">
          <div class="panel-title-group">
            <span class="panel-index">05</span>
            <div>
              <p class="panel-kicker">TASK DETAILS</p>
              <h2 id="details-title">任务详情</h2>
            </div>
          </div>
          <button type="button" class="symbol-button" aria-label="复制任务信息">▣</button>
        </header>

        <div v-if="selectedTask" class="detail-table">
          <div>
            <span>任务标题</span>
            <strong>{{ selectedTask.title }}</strong>
          </div>
          <div>
            <span>状态</span>
            <strong class="state-chip" :class="{ 'state-chip--done': selectedTask.status === 'done' }">
              {{ selectedTask.status === 'done' ? '已完成' : '进行中' }}
            </strong>
          </div>
          <div>
            <span>优先级</span>
            <strong
              class="priority-badge"
              :class="`priority-badge--${selectedTask.priority.toLowerCase()}`"
            >
              {{ selectedTask.priority }}
            </strong>
          </div>
        </div>

        <p v-else class="empty-state empty-state--details">暂无选中的任务</p>

        <div v-if="selectedTask" class="detail-actions">
          <button type="button" @click="toggleTask(selectedTask)">
            ✓ {{ selectedTask.status === 'done' ? '恢复' : '完成' }}
          </button>
          <button type="button" @click="cyclePriority(selectedTask)">✎ 调整</button>
          <button type="button" class="danger-action" @click="deleteTask(selectedTask.id)">⌫ 删除</button>
        </div>
      </section>
    </aside>
  </main>
</template>

<style scoped>
.todo-workspace {
  display: grid;
  grid-template-columns: minmax(460px, 1fr) minmax(340px, 0.78fr);
  gap: 1.5rem;
  min-width: 0;
  min-height: 0;
  overflow: hidden;
}

.panel {
  position: relative;
  border: 1px solid rgba(38, 255, 111, 0.45);
  border-radius: 18px;
  background:
    linear-gradient(180deg, rgba(2, 20, 13, 0.88), rgba(1, 8, 6, 0.94)),
    radial-gradient(circle at 20% 0%, rgba(40, 255, 119, 0.1), transparent 36%);
  box-shadow:
    0 0 0 1px rgba(11, 83, 41, 0.48) inset,
    0 0 32px rgba(19, 255, 92, 0.09);
}

.panel::before {
  position: absolute;
  inset: 0;
  border-radius: inherit;
  pointer-events: none;
  content: '';
  background: linear-gradient(135deg, rgba(71, 255, 136, 0.1), transparent 28%, transparent 70%, rgba(0, 229, 255, 0.08));
}

.todo-panel {
  display: grid;
  grid-template-rows: auto auto auto minmax(0, 1fr) auto;
  min-height: 0;
  padding: 1.5rem;
}

.info-column {
  display: grid;
  grid-template-rows: auto minmax(0, 1fr);
  gap: 1.5rem;
  min-width: 0;
  min-height: 0;
  overflow: hidden;
}

.status-panel,
.details-panel {
  padding: 1.35rem 1.5rem;
}

.panel-header,
.panel-title-group,
.task-entry,
.filter-tabs,
.task-summary,
.progress-row,
.detail-actions {
  position: relative;
  z-index: 1;
}

.panel-header {
  display: flex;
  align-items: flex-start;
  justify-content: space-between;
  gap: 1rem;
  margin-bottom: 1.45rem;
}

.panel-title-group {
  display: flex;
  align-items: center;
  gap: 0.85rem;
}

.panel-index {
  display: grid;
  place-items: center;
  width: 2.2rem;
  height: 2.2rem;
  border: 1px solid rgba(38, 255, 111, 0.82);
  border-radius: 8px;
  color: #2fff70;
  font-size: 1rem;
  font-weight: 800;
  text-shadow: 0 0 12px rgba(47, 255, 112, 0.75);
  box-shadow: 0 0 18px rgba(47, 255, 112, 0.12);
}

.panel-index--cyan {
  border-color: rgba(31, 229, 255, 0.82);
  color: #21e6ff;
  text-shadow: 0 0 12px rgba(33, 230, 255, 0.75);
}

.panel-kicker {
  margin: 0 0 0.2rem;
  color: rgba(185, 255, 205, 0.48);
  font-size: 0.72rem;
  letter-spacing: 0.24em;
}

.panel h2 {
  margin: 0;
  color: #31ff72;
  font-size: clamp(1.55rem, 2.1vw, 2.2rem);
  letter-spacing: 0;
  text-shadow: 0 0 18px rgba(49, 255, 114, 0.24);
}

.status-panel h2 {
  color: #21e6ff;
  text-shadow: 0 0 18px rgba(33, 230, 255, 0.3);
}

.symbol-button {
  display: grid;
  place-items: center;
  width: 2.55rem;
  height: 2.55rem;
  border: 1px solid rgba(45, 255, 116, 0.8);
  border-radius: 8px;
  background: rgba(2, 19, 11, 0.82);
  color: #2fff70;
  font-size: 1.35rem;
  line-height: 1;
  box-shadow: 0 0 18px rgba(47, 255, 112, 0.16);
}

.symbol-button--cyan {
  border-color: rgba(33, 230, 255, 0.76);
  color: #21e6ff;
  box-shadow: 0 0 18px rgba(33, 230, 255, 0.16);
}

.task-entry {
  display: grid;
  grid-template-columns: minmax(0, 1fr) 10rem;
  gap: 1rem;
  margin-bottom: 1.55rem;
}

.task-entry input {
  min-width: 0;
  height: 3.5rem;
  border: 1px solid rgba(64, 255, 131, 0.24);
  border-radius: 8px;
  background: rgba(0, 10, 8, 0.62);
  color: #effff4;
  outline: none;
  padding: 0 1.25rem;
  box-shadow: 0 0 0 1px rgba(9, 76, 38, 0.42) inset;
}

.task-entry input:focus {
  border-color: rgba(54, 255, 121, 0.76);
  box-shadow:
    0 0 0 1px rgba(42, 255, 111, 0.16) inset,
    0 0 24px rgba(42, 255, 111, 0.12);
}

.task-entry button,
.detail-actions button {
  border: 1px solid rgba(45, 255, 116, 0.68);
  border-radius: 8px;
  background: linear-gradient(90deg, rgba(22, 103, 45, 0.85), rgba(3, 41, 18, 0.9));
  color: #31ff72;
  font-weight: 700;
  box-shadow:
    0 0 0 1px rgba(45, 255, 116, 0.12) inset,
    0 0 22px rgba(45, 255, 116, 0.12);
}

.task-entry button {
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 0.95rem;
}

.filter-tabs {
  display: grid;
  grid-template-columns: repeat(4, minmax(0, 1fr));
  margin-bottom: 1.25rem;
  border: 1px solid rgba(45, 255, 116, 0.22);
  border-radius: 8px;
  overflow: hidden;
}

.filter-tabs__item {
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 0.55rem;
  min-width: 0;
  height: 2.8rem;
  border: 0;
  border-right: 1px solid rgba(45, 255, 116, 0.16);
  background: rgba(2, 19, 12, 0.62);
  color: rgba(222, 255, 230, 0.62);
}

.filter-tabs__item:last-child {
  border-right: 0;
}

.filter-tabs__item strong {
  min-width: 1.5rem;
  border-radius: 6px;
  background: rgba(154, 255, 187, 0.12);
  color: #dcffe6;
  font-size: 0.85rem;
}

.filter-tabs__item--active {
  background: linear-gradient(90deg, rgba(22, 107, 43, 0.78), rgba(4, 45, 18, 0.82));
  color: #34ff75;
  box-shadow: 0 0 24px rgba(40, 255, 109, 0.11) inset;
}

.task-list {
  position: relative;
  z-index: 1;
  display: grid;
  align-content: start;
  gap: 0.65rem;
  min-height: 0;
  overflow: auto;
  padding: 0.15rem 0.2rem 0.1rem 0;
}

.task-list::-webkit-scrollbar {
  width: 0.35rem;
}

.task-list::-webkit-scrollbar-thumb {
  border-radius: 999px;
  background: rgba(46, 255, 116, 0.32);
}

.task-card {
  display: grid;
  grid-template-columns: 2rem minmax(0, 1fr) auto;
  align-items: center;
  gap: 0.9rem;
  min-height: 3.85rem;
  border: 1px solid rgba(130, 255, 164, 0.13);
  border-radius: 10px;
  background: rgba(4, 16, 13, 0.76);
  color: #effff4;
  padding: 0.65rem 0.8rem;
  transition:
    border-color 0.18s ease,
    background 0.18s ease,
    transform 0.18s ease,
    box-shadow 0.18s ease;
}

.task-card:hover,
.task-card--selected {
  border-color: rgba(44, 255, 116, 0.42);
  background: rgba(6, 28, 18, 0.82);
  box-shadow: 0 0 22px rgba(32, 255, 99, 0.08);
}

.task-card:hover {
  transform: translateY(-1px);
}

.task-card h3 {
  margin: 0;
  min-width: 0;
  overflow: hidden;
  color: #f4fff7;
  font-size: 1rem;
  font-weight: 650;
  letter-spacing: 0;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.task-card--done h3 {
  color: rgba(223, 255, 231, 0.48);
  text-decoration: line-through;
}

.status-toggle {
  display: grid;
  place-items: center;
  width: 1.6rem;
  height: 1.6rem;
  border: 2px solid #28ff68;
  border-radius: 50%;
  background: transparent;
  color: #28ff68;
  line-height: 1;
  box-shadow: 0 0 12px rgba(40, 255, 104, 0.18);
}

.task-card:not(.task-card--done) .status-toggle {
  border-color: #21e6ff;
  box-shadow: 0 0 14px rgba(33, 230, 255, 0.18);
}

.priority-badge {
  display: inline-grid;
  place-items: center;
  min-width: 4.25rem;
  height: 1.8rem;
  border: 1px solid;
  border-radius: 6px;
  background: rgba(3, 15, 12, 0.86);
  font-size: 0.82rem;
  font-weight: 800;
}

button.priority-badge {
  padding: 0 0.6rem;
}

.priority-badge--high {
  border-color: rgba(33, 230, 255, 0.76);
  color: #21e6ff;
}

.priority-badge--medium {
  border-color: rgba(255, 187, 0, 0.68);
  color: #ffc928;
}

.priority-badge--low {
  border-color: rgba(41, 255, 111, 0.62);
  color: #31ff72;
}

.empty-state {
  margin: 1.5rem 0 0;
  color: rgba(224, 255, 232, 0.5);
  text-align: center;
}

.empty-state--details {
  position: relative;
  z-index: 1;
  align-self: start;
}

.task-summary {
  display: flex;
  flex-wrap: wrap;
  gap: 0.75rem 1.25rem;
  margin-top: 1.25rem;
  padding-top: 1.1rem;
  border-top: 1px solid rgba(58, 255, 126, 0.11);
  color: rgba(222, 255, 230, 0.68);
}

.task-summary strong {
  color: #31ff72;
}

.metric-grid {
  position: relative;
  z-index: 1;
  display: grid;
  grid-template-columns: repeat(4, minmax(0, 1fr));
  gap: 0.8rem;
  margin-bottom: 1.15rem;
}

.metric-card {
  min-height: 6.7rem;
  border: 1px solid rgba(33, 230, 255, 0.24);
  border-radius: 10px;
  background: rgba(5, 22, 23, 0.72);
  padding: 1rem;
  box-shadow: 0 0 20px rgba(33, 230, 255, 0.05) inset;
}

.metric-card--green {
  border-color: rgba(45, 255, 116, 0.24);
}

.metric-card--cyan {
  border-color: rgba(33, 230, 255, 0.28);
}

.metric-card span {
  display: block;
  margin-bottom: 0.75rem;
  color: rgba(237, 255, 244, 0.68);
  font-size: 0.82rem;
}

.metric-card strong {
  color: #21e6ff;
  font-size: 1.85rem;
  line-height: 1;
  text-shadow: 0 0 16px rgba(33, 230, 255, 0.26);
}

.metric-card--green strong {
  color: #31ff72;
  text-shadow: 0 0 16px rgba(49, 255, 114, 0.25);
}

.metric-card small {
  margin-left: 0.25rem;
  color: rgba(235, 255, 241, 0.7);
}

.progress-row {
  display: flex;
  justify-content: space-between;
  gap: 1rem;
  color: rgba(236, 255, 241, 0.72);
  font-size: 0.92rem;
}

.progress-row strong {
  color: #eafff0;
}

.progress-track {
  position: relative;
  z-index: 1;
  height: 0.9rem;
  margin-top: 0.8rem;
  overflow: hidden;
  border-radius: 999px;
  background: rgba(235, 255, 241, 0.08);
}

.progress-track span {
  display: block;
  height: 100%;
  border-radius: inherit;
  background: linear-gradient(90deg, #21e6ff, #31ff72);
  box-shadow: 0 0 18px rgba(33, 230, 255, 0.4);
  transition: width 0.2s ease;
}

.details-panel {
  display: grid;
  grid-template-rows: auto minmax(0, 1fr) auto;
  min-height: 0;
}

.detail-table {
  position: relative;
  z-index: 1;
  min-height: 0;
  overflow: auto;
}

.detail-table div {
  display: grid;
  grid-template-columns: 8.5rem minmax(0, 1fr);
  gap: 1rem;
  padding: 0.9rem 0;
  border-bottom: 1px solid rgba(79, 255, 137, 0.12);
}

.detail-table span {
  color: rgba(237, 255, 243, 0.64);
}

.detail-table strong {
  min-width: 0;
  color: #f3fff6;
  font-weight: 600;
}

.state-chip {
  color: #21e6ff;
}

.state-chip::before {
  display: inline-block;
  width: 0.75rem;
  height: 0.75rem;
  margin-right: 0.45rem;
  border-radius: 999px;
  background: #21e6ff;
  box-shadow: 0 0 12px rgba(33, 230, 255, 0.55);
  content: '';
}

.state-chip--done {
  color: #31ff72;
}

.state-chip--done::before {
  background: #31ff72;
  box-shadow: 0 0 12px rgba(49, 255, 114, 0.55);
}

.detail-actions {
  display: grid;
  grid-template-columns: repeat(3, minmax(0, 1fr));
  gap: 0.75rem;
  margin-top: 1rem;
}

.detail-actions button {
  min-width: 0;
  min-height: 3rem;
}

.detail-actions .danger-action {
  border-color: rgba(255, 68, 68, 0.72);
  background: rgba(39, 5, 5, 0.72);
  color: #ff5757;
}

@media (max-width: 1280px) {
  .todo-workspace,
  .info-column {
    overflow: visible;
  }

  .todo-workspace {
    grid-template-columns: 1fr;
  }
}

@media (max-width: 720px) {
  .todo-workspace {
    gap: 1rem;
  }

  .todo-panel,
  .status-panel,
  .details-panel {
    padding: 1rem;
  }

  .task-entry,
  .metric-grid,
  .detail-actions {
    grid-template-columns: 1fr;
  }

  .filter-tabs {
    grid-template-columns: repeat(2, minmax(0, 1fr));
  }

  .task-card {
    grid-template-columns: 2rem minmax(0, 1fr);
  }

  .task-card .priority-badge {
    grid-column: 2;
    justify-self: start;
  }

  .detail-table div {
    grid-template-columns: 1fr;
    gap: 0.35rem;
  }
}
</style>
