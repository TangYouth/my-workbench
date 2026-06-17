<script setup lang="ts">
import { computed, ref } from 'vue'

type ConfigItem = {
  id: string
  name: string
  state: string
}

type SkillItem = {
  id: string
  name: string
  tag: string
}

const configs: ConfigItem[] = [
  { id: 'default-dev', name: 'default-dev', state: 'ACTIVE' },
  { id: 'prod-safe', name: 'prod-safe', state: 'READY' },
  { id: 'debug-local', name: 'debug-local', state: 'READY' },
  { id: 'agent-fast', name: 'agent-fast', state: 'READY' },
  { id: 'agent-secure', name: 'agent-secure', state: 'READY' },
]

const skills: SkillItem[] = [
  { id: 'repo-analyze', name: 'repo-analyze', tag: '代码分析' },
  { id: 'code-review', name: 'code-review', tag: '代码评审' },
  { id: 'refactor', name: 'refactor', tag: '代码重构' },
  { id: 'bugfix', name: 'bugfix', tag: '调试修复' },
  { id: 'test-gen', name: 'test-gen', tag: '测试' },
  { id: 'doc-gen', name: 'doc-gen', tag: '文档' },
  { id: 'deploy-check', name: 'deploy-check', tag: '部署检查' },
  { id: 'prompt-debug', name: 'prompt-debug', tag: '调试辅助' },
]

const activeConfigId = ref('default-dev')
const skillKeyword = ref('')
const activeSkillId = ref('repo-analyze')

const activeConfig = computed<ConfigItem>(() => {
  return configs.find((item) => item.id === activeConfigId.value) ?? configs[0]!
})

const filteredSkills = computed(() => {
  const keyword = skillKeyword.value.trim().toLowerCase()

  if (!keyword) {
    return skills
  }

  return skills.filter((item) => {
    return item.name.toLowerCase().includes(keyword) || item.tag.includes(skillKeyword.value.trim())
  })
})
</script>

<template>
  <main class="workbench-main">
    <section class="panel panel--green">
      <header class="panel__header">
        <div class="panel__title-row">
          <span class="panel__index">01</span>
          <div>
            <p class="panel__eyebrow">CONFIG SWITCH</p>
            <h2 class="panel__title">配置切换</h2>
          </div>
        </div>
        <div class="panel__symbol">⟷</div>
      </header>

      <div class="panel__body panel__body--config">
        <section class="module-block module-block--active-config">
          <div class="module-block__header">
            <h3>当前活动配置</h3>
            <span>X</span>
          </div>

          <article class="config-card config-card--active">
            <div class="config-card__meta">
              <span class="config-card__radio"></span>
              <span class="config-card__name">{{ activeConfig.name }}</span>
            </div>
            <span class="config-card__badge">{{ activeConfig.state }}</span>
          </article>
        </section>

        <section class="module-block module-block--list">
          <div class="module-block__header">
            <h3>可用配置列表</h3>
            <span>X</span>
          </div>

          <div class="config-list">
            <button
              v-for="config in configs"
              :key="config.id"
              type="button"
              class="config-card"
              :class="{ 'config-card--active': config.id === activeConfigId }"
              @click="activeConfigId = config.id"
            >
              <div class="config-card__meta">
                <span class="config-card__radio"></span>
                <span class="config-card__name">{{ config.name }}</span>
              </div>
              <span v-if="config.id === activeConfigId" class="config-card__badge">
                {{ config.state }}
              </span>
            </button>
          </div>
        </section>

        <div class="switch-button-wrap">
          <button type="button" class="switch-button">≫ 切换配置</button>
        </div>
      </div>
    </section>

    <section class="panel panel--cyan">
      <header class="panel__header">
        <div class="panel__title-row">
          <span class="panel__index">02</span>
          <div>
            <p class="panel__eyebrow">SKILL INSPECTOR</p>
            <h2 class="panel__title">Skills 查看</h2>
          </div>
        </div>
        <div class="panel__symbol">&lt;/&gt;</div>
      </header>

      <div class="panel__body panel__body--skills">
        <label class="search-box">
          <span class="search-box__icon">⌕</span>
          <input v-model="skillKeyword" type="text" placeholder="搜索 skills..." />
        </label>

        <section class="module-block module-block--flat">
          <div class="module-block__header">
            <h3>Skills 列表</h3>
            <span>X</span>
          </div>

          <div class="skill-list">
            <button
              v-for="skill in filteredSkills"
              :key="skill.id"
              type="button"
              class="skill-row"
              :class="{ 'skill-row--active': skill.id === activeSkillId }"
              @click="activeSkillId = skill.id"
            >
              <div class="skill-row__meta">
                <span class="skill-row__bullet"></span>
                <span class="skill-row__name">{{ skill.name }}</span>
              </div>
              <span class="skill-row__tag">{{ skill.tag }}</span>
            </button>

            <p v-if="!filteredSkills.length" class="skill-list__empty">未找到匹配的 skills。</p>
          </div>
        </section>
      </div>
    </section>
  </main>
</template>

<style scoped>
.workbench-main {
  display: grid;
  grid-template-columns: repeat(2, minmax(0, 1fr));
  gap: 1.5rem;
  min-width: 0;
  min-height: 0;
  height: 100%;
}

.panel {
  position: relative;
  display: flex;
  flex-direction: column;
  min-width: 0;
  min-height: 0;
  border-radius: 1.15rem;
  background: linear-gradient(180deg, rgba(2, 13, 9, 0.98), rgba(2, 8, 7, 0.96));
  overflow: hidden;
}

.panel::before {
  content: '';
  position: absolute;
  inset: 0;
  border-radius: inherit;
  pointer-events: none;
}

.panel--green {
  border: 1px solid rgba(61, 255, 126, 0.5);
  box-shadow:
    0 0 0 1px rgba(18, 105, 47, 0.34) inset,
    0 0 36px rgba(22, 136, 54, 0.12);
}

.panel--green::before {
  box-shadow: 0 0 34px rgba(45, 255, 111, 0.14) inset;
}

.panel--cyan {
  border: 1px solid rgba(19, 233, 255, 0.62);
  box-shadow:
    0 0 0 1px rgba(9, 76, 92, 0.32) inset,
    0 0 36px rgba(0, 170, 255, 0.1);
}

.panel--cyan::before {
  box-shadow: 0 0 32px rgba(0, 213, 255, 0.12) inset;
}

.panel__header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 1rem;
  padding: 0.95rem 1rem 0.8rem;
  border-bottom: 1px solid rgba(255, 255, 255, 0.08);
}

.panel__title-row {
  display: flex;
  align-items: center;
  gap: 0.8rem;
}

.panel__index {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  width: 2rem;
  height: 2rem;
  border-radius: 0.45rem;
  border: 1px solid currentColor;
  font-size: 1rem;
}

.panel--green .panel__index,
.panel--green .panel__title,
.panel--green .panel__symbol {
  color: #38ff74;
}

.panel--cyan .panel__index,
.panel--cyan .panel__title,
.panel--cyan .panel__symbol {
  color: #23e7ff;
}

.panel__eyebrow {
  margin-bottom: 0.18rem;
  color: rgba(226, 255, 234, 0.4);
  font-size: 0.66rem;
  letter-spacing: 0.24em;
}

.panel__title {
  font-size: clamp(1.3rem, 1.8vw, 1.95rem);
  font-weight: 700;
}

.panel__symbol {
  display: grid;
  place-items: center;
  width: 2.45rem;
  height: 2.45rem;
  border: 1px solid currentColor;
  border-radius: 0.55rem;
  font-size: 1rem;
}

.panel__body {
  display: flex;
  flex: 1;
  min-height: 0;
  flex-direction: column;
  padding: 0.95rem 1rem 1rem;
}

.panel__body--config,
.panel__body--skills {
  overflow: hidden;
}

.module-block {
  padding: 0.8rem;
  border: 1px solid rgba(89, 255, 153, 0.15);
  border-radius: 0.95rem;
  background: rgba(2, 14, 9, 0.7);
}

.module-block + .module-block,
.switch-button,
.module-block--flat {
  margin-top: 0.8rem;
}

.module-block--flat {
  padding: 0;
  border: 0;
  background: transparent;
}

.module-block--active-config {
  flex: 0 0 auto;
}

.module-block--list {
  display: flex;
  flex: 1;
  min-height: 0;
  flex-direction: column;
  overflow: hidden;
}

.module-block__header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  margin-bottom: 0.7rem;
  color: rgba(235, 255, 241, 0.9);
}

.module-block__header h3 {
  font-size: 1rem;
}

.module-block__header span {
  color: rgba(93, 255, 153, 0.42);
}

.config-list,
.skill-list {
  display: grid;
  gap: 0.55rem;
}

.config-list {
  flex: 1;
  min-height: 0;
  overflow: auto;
  padding-right: 0.2rem;
}

.skill-list {
  min-height: 0;
  overflow: auto;
  padding-right: 0.2rem;
}

.config-list::-webkit-scrollbar,
.skill-list::-webkit-scrollbar {
  width: 0.45rem;
}

.config-list::-webkit-scrollbar-thumb,
.skill-list::-webkit-scrollbar-thumb {
  border-radius: 999px;
  background: rgba(119, 255, 167, 0.24);
}

.config-card,
.skill-row {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 1rem;
  width: 100%;
  padding: 0.75rem 0.9rem;
  border: 1px solid rgba(103, 255, 159, 0.12);
  border-radius: 0.8rem;
  background: rgba(4, 16, 11, 0.76);
  color: #effff4;
  font: inherit;
  text-align: left;
  transition:
    border-color 0.2s ease,
    box-shadow 0.2s ease,
    transform 0.2s ease;
}

.config-card:hover,
.skill-row:hover {
  transform: translateY(-1px);
}

.config-card--active {
  border-color: rgba(67, 255, 128, 0.42);
  background: linear-gradient(90deg, rgba(18, 82, 35, 0.9), rgba(7, 35, 17, 0.94));
  box-shadow:
    0 0 0 1px rgba(67, 255, 128, 0.12) inset,
    0 0 26px rgba(58, 227, 104, 0.12);
}

.config-card__meta,
.skill-row__meta {
  display: flex;
  align-items: center;
  gap: 0.8rem;
  min-width: 0;
}

.config-card__radio,
.skill-row__bullet {
  flex: 0 0 auto;
  width: 1.05rem;
  height: 1.05rem;
  border-radius: 999px;
  border: 1px solid currentColor;
}

.config-card--active .config-card__radio {
  background:
    radial-gradient(circle at center, #20ff62 0 45%, transparent 47% 100%);
  color: #20ff62;
  box-shadow: 0 0 16px rgba(32, 255, 98, 0.5);
}

.config-card__name,
.skill-row__name {
  font-size: 0.95rem;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.config-card__badge,
.skill-row__tag {
  flex: 0 0 auto;
  padding: 0.22rem 0.6rem;
  border: 1px solid currentColor;
  border-radius: 0.45rem;
  font-size: 0.82rem;
}

.config-card__badge {
  color: #7cff9f;
}

.search-box {
  display: flex;
  align-items: center;
  gap: 0.8rem;
  width: 100%;
  margin-bottom: 0.85rem;
  padding: 0.78rem 0.9rem;
  border: 1px solid rgba(255, 255, 255, 0.14);
  border-radius: 0.9rem;
  background: rgba(3, 8, 10, 0.88);
}

.search-box__icon {
  color: #f8ffff;
  font-size: 1.5rem;
}

.search-box input {
  width: 100%;
  border: 0;
  outline: 0;
  background: transparent;
  color: #efffff;
  font: inherit;
  font-size: 1rem;
}

.search-box input::placeholder {
  color: rgba(225, 241, 245, 0.52);
}

.panel--cyan .skill-row {
  border-color: rgba(73, 235, 255, 0.12);
  background: rgba(7, 12, 15, 0.74);
}

.panel--cyan .skill-row--active {
  border-color: rgba(36, 231, 255, 0.56);
  box-shadow:
    0 0 0 1px rgba(36, 231, 255, 0.12) inset,
    0 0 22px rgba(36, 231, 255, 0.12);
}

.panel--cyan .skill-row__bullet {
  width: 0.8rem;
  height: 0.8rem;
  border: 0;
  background: rgba(255, 255, 255, 0.66);
}

.panel--cyan .skill-row--active .skill-row__bullet {
  background: #22e8ff;
  box-shadow: 0 0 16px rgba(34, 232, 255, 0.7);
}

.panel--cyan .skill-row__name,
.panel--cyan .skill-row__tag {
  color: #dcfcff;
}

.panel--cyan .skill-row--active .skill-row__name,
.panel--cyan .skill-row--active .skill-row__tag {
  color: #24eaff;
}

.skill-list__empty {
  padding: 1rem;
  border: 1px dashed rgba(36, 231, 255, 0.25);
  border-radius: 0.8rem;
  color: rgba(219, 254, 255, 0.74);
}

.switch-button-wrap {
  flex: 0 0 auto;
  margin-top: 0.8rem;
  padding-top: 0.2rem;
  background: linear-gradient(180deg, rgba(2, 13, 9, 0), rgba(2, 13, 9, 0.94) 30%);
}

.switch-button {
  width: 100%;
  margin-top: 0;
  padding: 0.9rem 1rem;
  border: 1px solid rgba(67, 255, 128, 0.6);
  border-radius: 0.85rem;
  background: linear-gradient(90deg, rgba(18, 100, 37, 0.95), rgba(11, 61, 28, 0.94));
  color: #effff2;
  font: inherit;
  font-size: 1.15rem;
  box-shadow:
    0 0 0 1px rgba(117, 255, 159, 0.14) inset,
    0 0 30px rgba(31, 168, 70, 0.16);
}

@media (max-width: 1200px) {
  .workbench-main {
    grid-template-columns: 1fr;
  }
}

@media (max-width: 720px) {
  .workbench-main {
    min-height: auto;
  }

  .panel {
    min-height: auto;
  }

  .panel__header,
  .panel__body {
    padding-left: 1rem;
    padding-right: 1rem;
  }

  .panel__title {
    font-size: 1.45rem;
  }

  .config-card,
  .skill-row {
    align-items: flex-start;
    flex-direction: column;
  }
}
</style>
