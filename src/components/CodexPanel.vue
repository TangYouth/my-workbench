<script setup lang="ts">
import { computed, onMounted, ref, shallowRef } from 'vue'

type ConfigItem = {
  id: string
  name: string
  state: string
  authBackupFileName: string
  configBackupFileName: string
}

type SkillItem = {
  id: string
  name: string
  directoryHandle: DirectoryHandle
  content: string
}

type DirectoryHandle = {
  name?: string
  entries: () => AsyncIterableIterator<[string, FileSystemHandle]>
  getDirectoryHandle?: (
    name: string,
    options?: { create?: boolean },
  ) => Promise<DirectoryHandle>
  getFileHandle: (
    name: string,
    options?: { create?: boolean },
  ) => Promise<FileSystemFileHandle>
  queryPermission?: (descriptor: { mode: 'read' | 'readwrite' }) => Promise<PermissionState>
  requestPermission?: (descriptor: { mode: 'read' | 'readwrite' }) => Promise<PermissionState>
}

type FileSystemHandle = {
  kind: 'file' | 'directory'
  name: string
}

type FileSystemFileHandle = FileSystemHandle & {
  getFile: () => Promise<File>
  createWritable: () => Promise<FileSystemWritableFileStream>
}

type FileSystemWritableFileStream = {
  write: (data: string | Blob | BufferSource) => Promise<void>
  close: () => Promise<void>
}

const CONFIG_BASE_FILES = ['auth.json', 'config.toml'] as const
const HANDLE_DB_NAME = 'codex-workbench'
const HANDLE_STORE_NAME = 'handles'
const HANDLE_KEY = 'codex-directory'

const codexDirHandle = shallowRef<DirectoryHandle | null>(null)
const skillsDirHandle = shallowRef<DirectoryHandle | null>(null)
const configs = ref<ConfigItem[]>([])
const skillItems = ref<SkillItem[]>([])
const currentConfigId = ref('')
const selectedConfigId = ref('')
const configStatus = ref('等待授权读取 Codex 配置目录')
const configError = ref('')
const skillStatus = ref('等待授权读取 Codex Skills 目录')
const skillError = ref('')
const scanDebug = ref({
  auth: [] as string[],
  config: [] as string[],
  files: 0,
})
const isLoadingConfigs = ref(false)
const isSwitchingConfig = ref(false)
const isLoadingSkills = ref(false)
const skillKeyword = ref('')
const activeSkillId = ref('')
const selectedSkill = ref<SkillItem | null>(null)
const copyStatus = ref('')
let copyStatusTimer: number | undefined

const defaultCodexPath = computed(() => {
  const platform = window.navigator.platform.toLowerCase()
  const userAgent = window.navigator.userAgent.toLowerCase()

  if (platform.includes('win') || userAgent.includes('windows')) {
    return '%USERPROFILE%\\.codex\\'
  }

  return '~/.codex/'
})

const defaultSkillsPath = computed(() => {
  const platform = window.navigator.platform.toLowerCase()
  const userAgent = window.navigator.userAgent.toLowerCase()

  if (platform.includes('win') || userAgent.includes('windows')) {
    return '%USERPROFILE%\\.codex\\skills\\'
  }

  return '~/.codex/skills/'
})

const selectedConfig = computed(() => {
  return configs.value.find((item) => item.id === selectedConfigId.value) ?? null
})

const activeConfig = computed<ConfigItem>(() => ({
  id: currentConfigId.value || 'unknown',
  name: currentConfigId.value || '未知配置',
  state: codexDirHandle.value ? 'LIVE' : 'LOCKED',
  authBackupFileName: '',
  configBackupFileName: '',
}))

const filteredSkills = computed(() => {
  const keyword = skillKeyword.value.trim().toLowerCase()

  if (!keyword) {
    return skillItems.value
  }

    return skillItems.value.filter((item) => {
    return item.name.toLowerCase().includes(keyword)
  })
})

const supportsDirectoryPicker = () => {
  return typeof window !== 'undefined' && 'showDirectoryPicker' in window
}

const authorizeCodexDirectory = async () => {
  configError.value = ''

  if (!supportsDirectoryPicker()) {
    configError.value = '当前浏览器不支持目录授权，请使用支持 File System Access API 的 Chromium 浏览器。'
    return
  }

  try {
    const picker = (window as unknown as {
      showDirectoryPicker: (options?: {
        id?: string
        mode?: 'read' | 'readwrite'
        startIn?: 'desktop' | 'documents' | 'downloads' | 'music' | 'pictures' | 'videos'
      }) => Promise<DirectoryHandle>
    }).showDirectoryPicker

    const handle = await picker({
      id: 'codex-config-directory',
      mode: 'readwrite',
    })

    const permission = await requestReadWritePermission(handle)

    if (permission !== 'granted') {
      configError.value = '未获得读写权限，无法读取或切换 Codex 配置。'
      return
    }

    await persistDirectoryHandle(handle)
    codexDirHandle.value = await resolveCodexDirectory(handle)
    configStatus.value = `已授权：${codexDirHandle.value.name || defaultCodexPath.value}`
    await scanConfigBackups()
    await scanSkills()
  } catch (error) {
    if (error instanceof DOMException && error.name === 'AbortError') {
      configStatus.value = '已取消目录授权'
      return
    }

    configError.value = getErrorMessage(error)
  }
}

const resolveCodexDirectory = async (handle: DirectoryHandle) => {
  if (handle.name === '.codex' || !handle.getDirectoryHandle) {
    return handle
  }

  try {
    return await handle.getDirectoryHandle('.codex')
  } catch {
    return handle
  }
}

const resolveSkillsDirectory = async () => {
  if (!codexDirHandle.value?.getDirectoryHandle) {
    return null
  }

  try {
    return await codexDirHandle.value.getDirectoryHandle('skills')
  } catch {
    return null
  }
}

const persistDirectoryHandle = async (handle: DirectoryHandle) => {
  const db = await openHandleDatabase()

  await new Promise<void>((resolve, reject) => {
    const transaction = db.transaction(HANDLE_STORE_NAME, 'readwrite')
    const store = transaction.objectStore(HANDLE_STORE_NAME)
    const request = store.put(handle, HANDLE_KEY)

    request.onsuccess = () => resolve()
    request.onerror = () => reject(request.error)
  })
}

const loadPersistedDirectoryHandle = async () => {
  const db = await openHandleDatabase()

  return await new Promise<DirectoryHandle | null>((resolve, reject) => {
    const transaction = db.transaction(HANDLE_STORE_NAME, 'readonly')
    const store = transaction.objectStore(HANDLE_STORE_NAME)
    const request = store.get(HANDLE_KEY)

    request.onsuccess = () => resolve((request.result as DirectoryHandle | undefined) ?? null)
    request.onerror = () => reject(request.error)
  })
}

const openHandleDatabase = async () => {
  return await new Promise<IDBDatabase>((resolve, reject) => {
    const request = window.indexedDB.open(HANDLE_DB_NAME, 1)

    request.onupgradeneeded = () => {
      request.result.createObjectStore(HANDLE_STORE_NAME)
    }

    request.onsuccess = () => resolve(request.result)
    request.onerror = () => reject(request.error)
  })
}

const requestReadWritePermission = async (handle: DirectoryHandle) => {
  if (!handle.queryPermission || !handle.requestPermission) {
    return 'granted' as PermissionState
  }

  const currentPermission = await handle.queryPermission({ mode: 'readwrite' })

  if (currentPermission === 'granted') {
    return currentPermission
  }

  return handle.requestPermission({ mode: 'readwrite' })
}

const scanConfigBackups = async () => {
  if (!codexDirHandle.value) {
    return
  }

  isLoadingConfigs.value = true
  configError.value = ''

  try {
    const files = new Set<string>()

    for await (const [name, handle] of codexDirHandle.value.entries()) {
      if (handle.kind === 'file') {
        files.add(name)
      }
    }

    const fileNames = [...files]
    const authBackups = extractBackupFiles(fileNames, 'auth.json')
    const configBackups = extractBackupFiles(fileNames, 'config.toml')
    scanDebug.value = {
      auth: [...authBackups.keys()].sort(),
      config: [...configBackups.keys()].sort(),
      files: files.size,
    }
    const pairedNames = [...authBackups.keys()]
      .filter((name) => configBackups.has(name))
      .sort((first, second) => {
        if (first === 'recent') return -1
        if (second === 'recent') return 1
        return first.localeCompare(second)
      })

    configs.value = pairedNames.map((name) => ({
      id: name,
      name,
      state: name === selectedConfigId.value ? 'SELECTED' : 'READY',
      authBackupFileName: authBackups.get(name)!,
      configBackupFileName: configBackups.get(name)!,
    }))

    currentConfigId.value = await detectCurrentConfigName(configs.value)

    if (!configs.value.some((item) => item.id === selectedConfigId.value)) {
      selectedConfigId.value = currentConfigId.value || configs.value[0]?.id || ''
    }

    configStatus.value = configs.value.length
      ? `已发现 ${configs.value.length} 组可切换配置`
      : `未发现成对配置。已扫描 ${files.size} 个文件`
  } catch (error) {
    configError.value = getErrorMessage(error)
  } finally {
    isLoadingConfigs.value = false
  }
}

const scanSkills = async () => {
  skillError.value = ''
  isLoadingSkills.value = true

  try {
    skillsDirHandle.value = await resolveSkillsDirectory()

    if (!skillsDirHandle.value) {
      skillItems.value = []
      skillStatus.value = `未找到 Skills 目录：${defaultSkillsPath.value}`
      return
    }

    const nextSkills: SkillItem[] = []

    for await (const [name, handle] of skillsDirHandle.value.entries()) {
      if (handle.kind !== 'directory') {
        continue
      }

      const directoryHandle = handle as unknown as DirectoryHandle

      try {
        const skillFileHandle = await directoryHandle.getFileHandle('SKILL.md')
        const skillFile = await skillFileHandle.getFile()
        const content = await skillFile.text()

        nextSkills.push({
          id: name,
          name,
          directoryHandle,
          content,
        })
      } catch {
        // 只有包含 SKILL.md 的文件夹才算 skill
      }
    }

    skillItems.value = nextSkills.sort((first, second) => first.name.localeCompare(second.name))

    if (!skillItems.value.some((item) => item.id === activeSkillId.value)) {
      activeSkillId.value = skillItems.value[0]?.id ?? ''
    }

    skillStatus.value = skillItems.value.length
      ? `已发现 ${skillItems.value.length} 个 skills`
      : '未发现包含 SKILL.md 的 skills 文件夹'
  } catch (error) {
    skillError.value = getErrorMessage(error)
  } finally {
    isLoadingSkills.value = false
  }
}

const openSkillModal = (skill: SkillItem) => {
  activeSkillId.value = skill.id
  selectedSkill.value = skill
}

const closeSkillModal = () => {
  selectedSkill.value = null
  copyStatus.value = ''
}

const copySelectedSkillContent = async () => {
  if (!selectedSkill.value) {
    return
  }

  const text = selectedSkill.value.content

  try {
    if (!navigator.clipboard?.writeText) {
      throw new Error('当前浏览器不支持复制')
    }

    await navigator.clipboard.writeText(text)
    copyStatus.value = '已复制'

    if (copyStatusTimer) {
      window.clearTimeout(copyStatusTimer)
    }

    copyStatusTimer = window.setTimeout(() => {
      copyStatus.value = ''
    }, 1500)
  } catch (error) {
    copyStatus.value = ''
    skillError.value = getErrorMessage(error)
  }
}

const extractBackupFiles = (files: string[], baseName: string) => {
  const backups = new Map<string, string>()

  files.forEach((fileName) => {
    const backupName = getBackupNameFromFile(fileName, baseName)

    if (!backupName) {
      return
    }

    backups.set(backupName, fileName)
  })

  return backups
}

const getBackupNameFromFile = (fileName: string, baseName: string) => {
  const normalizedFileName = normalizeFileName(fileName)
  const lowerFileName = normalizedFileName.toLowerCase()
  const lowerPrefix = `${baseName}.`.toLowerCase()

  if (!lowerFileName.startsWith(lowerPrefix)) {
    return ''
  }

  const match = normalizedFileName.match(
    new RegExp(`^${escapeRegExp(baseName)}\\.\\s*(.+?)\\.bak(?:\\.(?:json|toml))?$`, 'i'),
  )

  if (!match?.[1]) {
    return ''
  }

  return match[1]
    .replace(/\.(json|toml)$/i, '')
    .replace(/\s+/g, '')
    .trim()
}

const normalizeFileName = (fileName: string) => {
  return fileName.normalize('NFC').trim()
}

const escapeRegExp = (value: string) => {
  return value.replace(/[.*+?^${}()|[\]\\]/g, '\\$&')
}

const switchCodexConfig = async () => {
  if (!codexDirHandle.value || !selectedConfig.value) {
    return
  }

  isSwitchingConfig.value = true
  configError.value = ''

  try {
    const permission = await requestReadWritePermission(codexDirHandle.value)

    if (permission !== 'granted') {
      configError.value = '未获得读写权限，无法切换配置。'
      return
    }

    for (const baseFile of CONFIG_BASE_FILES) {
      await backupCurrentFile(baseFile)
      await replaceFromBackup(
        baseFile,
        baseFile === 'auth.json'
          ? selectedConfig.value.authBackupFileName
          : selectedConfig.value.configBackupFileName,
      )
    }

    currentConfigId.value = selectedConfig.value.name
    configStatus.value = `已切换到 ${selectedConfig.value.name}，重启 Codex 桌面端后生效`
    await scanConfigBackups()
  } catch (error) {
    configError.value = getErrorMessage(error)
  } finally {
    isSwitchingConfig.value = false
  }
}

const backupCurrentFile = async (baseFile: (typeof CONFIG_BASE_FILES)[number]) => {
  if (!codexDirHandle.value) {
    return
  }

  const currentHandle = await codexDirHandle.value.getFileHandle(baseFile)
  const currentFile = await currentHandle.getFile()
  const currentContent = await currentFile.text()
  const backupHandle = await codexDirHandle.value.getFileHandle(`${baseFile}.recent.bak`, {
    create: true,
  })

  await writeFile(backupHandle, currentContent)
}

const replaceFromBackup = async (
  baseFile: (typeof CONFIG_BASE_FILES)[number],
  backupFileName: string,
) => {
  if (!codexDirHandle.value) {
    return
  }

  const backupHandle = await codexDirHandle.value.getFileHandle(backupFileName)
  const backupFile = await backupHandle.getFile()
  const backupContent = await backupFile.text()
  const targetHandle = await codexDirHandle.value.getFileHandle(baseFile, { create: true })

  await writeFile(targetHandle, backupContent)
}

const detectCurrentConfigName = async (items: ConfigItem[]) => {
  if (!codexDirHandle.value || !items.length) {
    return ''
  }

  const currentAuthHandle = await codexDirHandle.value.getFileHandle('auth.json')
  const currentAuthFile = await currentAuthHandle.getFile()
  const currentAuthContent = await currentAuthFile.text()

  for (const item of items) {
    const backupHandle = await codexDirHandle.value.getFileHandle(item.authBackupFileName)
    const backupFile = await backupHandle.getFile()
    const backupContent = await backupFile.text()

    if (backupContent === currentAuthContent) {
      return item.id
    }
  }

  return items.find((item) => item.id === 'recent')?.id ?? items[0]?.id ?? ''
}

const writeFile = async (handle: FileSystemFileHandle, content: string) => {
  const writable = await handle.createWritable()

  await writable.write(content)
  await writable.close()
}

const getErrorMessage = (error: unknown) => {
  if (error instanceof Error) {
    return error.message
  }

  return '发生未知错误'
}

onMounted(() => {
  configStatus.value = `默认配置路径：${defaultCodexPath.value}`
  void restorePersistedCodexDirectory()
})

const restorePersistedCodexDirectory = async () => {
  try {
    const handle = await loadPersistedDirectoryHandle()

    if (!handle) {
      return
    }

    const permission = await requestReadWritePermission(handle)

    if (permission !== 'granted') {
      return
    }

    codexDirHandle.value = await resolveCodexDirectory(handle)
    configStatus.value = `已恢复授权：${codexDirHandle.value.name || defaultCodexPath.value}`
    await scanConfigBackups()
    await scanSkills()
  } catch {
    // 保持静默，页面会回退到授权入口
  }
}

if (typeof window !== 'undefined') {
  window.addEventListener('beforeunload', () => {
    if (copyStatusTimer) {
      window.clearTimeout(copyStatusTimer)
    }
  })
}
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

        <div class="rules-popover">
          <button type="button" class="panel__symbol" aria-label="查看配置切换规则">?</button>
          <div class="rules-popover__content">
            <p class="rules-popover__alert">修改配置后需要重启Codex桌面端。</p>
            <p>默认读取路径：Linux/macOS 为 ~/.codex/，Windows 为 %USERPROFILE%\\.codex\\。</p>
            <p>配置项必须同时存在 auth.json.xxx.bak 与 config.toml.xxx.bak，展示名称为 xxx。</p>
            <p>切换时会先把当前 auth.json 与 config.toml 保存为 recent 备份，再写入目标配置。</p>
          </div>
        </div>
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

          <div class="config-path">
            <span>PATH</span>
            <strong>{{ defaultCodexPath }}</strong>
          </div>
        </section>

        <section class="module-block module-block--list">
          <div class="module-block__header">
            <h3>可用配置列表</h3>
            <span>X</span>
          </div>

          <div v-if="!codexDirHandle" class="permission-panel">
            <p>{{ configStatus }}</p>
            <button type="button" class="terminal-action" @click="authorizeCodexDirectory">
              授权 .codex 目录
            </button>
          </div>

          <div v-else class="config-list">
            <button
              v-for="config in configs"
              :key="config.id"
              type="button"
              class="config-card"
              :class="{ 'config-card--active': config.id === selectedConfigId }"
              @click="selectedConfigId = config.id"
            >
              <div class="config-card__meta">
                <span class="config-card__radio"></span>
                <span class="config-card__name">{{ config.name }}</span>
              </div>
              <span v-if="config.id === selectedConfigId" class="config-card__badge">
                SELECTED
              </span>
            </button>

            <p v-if="!configs.length && !isLoadingConfigs" class="config-list__empty">
              未找到成对的 bak 配置。
            </p>
          </div>
        </section>

        <div class="scan-debug">
          <span>AUTH: {{ scanDebug.auth.length ? scanDebug.auth.join(', ') : 'none' }}</span>
          <span>CONFIG: {{ scanDebug.config.length ? scanDebug.config.join(', ') : 'none' }}</span>
        </div>

        <p class="config-status" :class="{ 'config-status--error': configError }">
          {{ configError || configStatus }}
        </p>

        <div class="switch-button-wrap">
          <button
            type="button"
            class="switch-button"
            :disabled="!selectedConfig || !codexDirHandle || isSwitchingConfig"
            @click="switchCodexConfig"
          >
            ≫ {{ isSwitchingConfig ? '切换中...' : '切换配置' }}
          </button>
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

          <div v-if="!codexDirHandle" class="permission-panel permission-panel--cyan">
            <p>{{ skillStatus }}</p>
            <button type="button" class="terminal-action terminal-action--cyan" @click="authorizeCodexDirectory">
              授权 .codex 目录
            </button>
          </div>

          <div v-else class="skill-list">
            <button
              v-for="skill in filteredSkills"
              :key="skill.id"
              type="button"
              class="skill-row"
              :class="{ 'skill-row--active': skill.id === activeSkillId }"
              @click="openSkillModal(skill)"
            >
              <div class="skill-row__meta">
                <span class="skill-row__bullet"></span>
                <span class="skill-row__name">{{ skill.name }}</span>
              </div>
            </button>

            <p v-if="!filteredSkills.length && !isLoadingSkills" class="skill-list__empty">
              {{ skillError || skillStatus || '未找到匹配的 skills。' }}
            </p>
          </div>
        </section>
      </div>
    </section>

    <Teleport to="body">
      <div v-if="selectedSkill" class="skill-modal" @click.self="closeSkillModal">
        <section class="skill-modal__panel" role="dialog" aria-modal="true">
          <header class="skill-modal__header">
            <div>
              <p class="skill-modal__eyebrow">SKILL.md</p>
              <h3>{{ selectedSkill.name }}</h3>
            </div>
            <div class="skill-modal__actions">
              <span v-if="copyStatus" class="skill-modal__copied">{{ copyStatus }}</span>
              <button
                type="button"
                class="skill-modal__copy"
                aria-label="复制 Skill 内容"
                @click="copySelectedSkillContent"
              >
                复制
              </button>
              <button type="button" class="skill-modal__close" aria-label="关闭 Skill 内容" @click="closeSkillModal">
                X
              </button>
            </div>
          </header>

          <pre class="skill-modal__content">{{ selectedSkill.content }}</pre>
        </section>
      </div>
    </Teleport>
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
  padding: 0.7rem 0.95rem 0.6rem;
  border-bottom: 1px solid rgba(255, 255, 255, 0.08);
}

.panel__title-row {
  display: flex;
  align-items: center;
  gap: 0.65rem;
}

.panel__index {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  width: 1.75rem;
  height: 1.75rem;
  border-radius: 0.45rem;
  border: 1px solid currentColor;
  font-size: 0.9rem;
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
  margin-bottom: 0.1rem;
  color: rgba(226, 255, 234, 0.4);
  font-size: 0.6rem;
  letter-spacing: 0.24em;
}

.panel__title {
  font-size: clamp(1.05rem, 1.45vw, 1.55rem);
  font-weight: 700;
}

.panel__symbol {
  display: grid;
  place-items: center;
  width: 2.05rem;
  height: 2.05rem;
  border: 1px solid currentColor;
  border-radius: 0.55rem;
  background: transparent;
  color: inherit;
  font: inherit;
  font-size: 1rem;
}

.rules-popover {
  position: relative;
}

.rules-popover__content {
  position: absolute;
  top: calc(100% + 0.75rem);
  right: 0;
  z-index: 4;
  width: min(25rem, 72vw);
  padding: 0.9rem;
  border: 1px solid rgba(61, 255, 126, 0.48);
  border-radius: 0.75rem;
  background:
    linear-gradient(180deg, rgba(3, 20, 9, 0.98), rgba(1, 8, 6, 0.98)),
    #020605;
  color: rgba(235, 255, 241, 0.86);
  opacity: 0;
  pointer-events: none;
  transform: translateY(-0.35rem);
  transition:
    opacity 0.16s ease,
    transform 0.16s ease;
  box-shadow:
    0 0 0 1px rgba(67, 255, 128, 0.12) inset,
    0 1rem 2.5rem rgba(0, 0, 0, 0.42),
    0 0 2rem rgba(32, 255, 98, 0.12);
}

.rules-popover:hover .rules-popover__content,
.rules-popover:focus-within .rules-popover__content {
  opacity: 1;
  pointer-events: auto;
  transform: translateY(0);
}

.rules-popover__content p {
  margin: 0;
  font-size: 0.78rem;
  line-height: 1.65;
}

.rules-popover__content p + p {
  margin-top: 0.55rem;
}

.rules-popover__alert {
  padding: 0.65rem 0.75rem;
  border: 1px solid rgba(61, 255, 126, 0.5);
  border-radius: 0.55rem;
  background: rgba(28, 107, 49, 0.35);
  color: #67ff94;
  font-weight: 700;
}

.panel__body {
  display: flex;
  flex: 1;
  min-height: 0;
  flex-direction: column;
  padding: 0.7rem 0.95rem 0.95rem;
}

.panel__body--config,
.panel__body--skills {
  overflow: hidden;
}

.module-block {
  padding: 0.62rem;
  border: 1px solid rgba(89, 255, 153, 0.15);
  border-radius: 0.95rem;
  background: rgba(2, 14, 9, 0.7);
}

.module-block + .module-block,
.switch-button,
.module-block--flat {
  margin-top: 0.58rem;
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
  margin-bottom: 0.5rem;
  color: rgba(235, 255, 241, 0.9);
}

.module-block__header h3 {
  font-size: 0.9rem;
}

.module-block__header span {
  color: rgba(93, 255, 153, 0.42);
}

.config-list,
.skill-list {
  display: grid;
  gap: 0.55rem;
}

.config-list,
.skill-list {
  min-height: 0;
  overflow: auto;
  padding-right: 0.2rem;
}

.config-list {
  flex: 1;
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
  padding: 0.62rem 0.78rem;
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
  background: radial-gradient(circle at center, #20ff62 0 45%, transparent 47% 100%);
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

.config-path,
.config-status {
  display: flex;
  align-items: center;
  gap: 0.6rem;
  margin-top: 0.48rem;
  color: rgba(213, 255, 226, 0.72);
  font-size: 0.72rem;
}

.scan-debug {
  display: flex;
  flex-wrap: wrap;
  gap: 0.5rem 0.9rem;
  margin-top: 0.42rem;
  color: rgba(123, 255, 166, 0.55);
  font-size: 0.68rem;
  letter-spacing: 0.08em;
}

.config-path span {
  color: #67ff94;
  letter-spacing: 0.18em;
}

.config-path strong {
  font-weight: 600;
}

.config-status {
  flex: 0 0 auto;
}

.config-status--error {
  color: #ff8f8f;
}

.permission-panel,
.config-list__empty,
.skill-list__empty {
  padding: 0.9rem;
  border: 1px dashed rgba(61, 255, 126, 0.28);
  border-radius: 0.8rem;
  color: rgba(219, 254, 226, 0.78);
}

.permission-panel p {
  margin: 0 0 0.6rem;
  font-size: 0.85rem;
}

.terminal-action {
  width: 100%;
  padding: 0.75rem 0.9rem;
  border: 1px solid rgba(67, 255, 128, 0.48);
  border-radius: 0.65rem;
  background: rgba(22, 88, 37, 0.5);
  color: #67ff94;
  font: inherit;
}

.permission-panel--cyan {
  border-color: rgba(36, 231, 255, 0.28);
  color: rgba(219, 254, 255, 0.78);
}

.terminal-action--cyan {
  border-color: rgba(36, 231, 255, 0.48);
  background: rgba(8, 72, 86, 0.5);
  color: #24eaff;
}

.search-box {
  display: flex;
  align-items: center;
  gap: 0.8rem;
  width: 100%;
  margin-bottom: 0.62rem;
  padding: 0.62rem 0.78rem;
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

.panel--cyan .skill-row__name {
  color: #dcfcff;
}

.panel--cyan .skill-row--active .skill-row__name {
  color: #24eaff;
}

.skill-list__empty {
  border-color: rgba(36, 231, 255, 0.25);
  color: rgba(219, 254, 255, 0.74);
}

.switch-button-wrap {
  flex: 0 0 auto;
  margin-top: 0.58rem;
  padding-top: 0.2rem;
  background: linear-gradient(180deg, rgba(2, 13, 9, 0), rgba(2, 13, 9, 0.94) 30%);
}

.switch-button {
  width: 100%;
  margin-top: 0;
  padding: 0.72rem 0.9rem;
  border: 1px solid rgba(67, 255, 128, 0.6);
  border-radius: 0.85rem;
  background: linear-gradient(90deg, rgba(18, 100, 37, 0.95), rgba(11, 61, 28, 0.94));
  color: #effff2;
  font: inherit;
  font-size: 1rem;
  box-shadow:
    0 0 0 1px rgba(117, 255, 159, 0.14) inset,
    0 0 30px rgba(31, 168, 70, 0.16);
}

.switch-button:disabled {
  cursor: not-allowed;
  opacity: 0.42;
}

.skill-modal {
  position: fixed;
  inset: 0;
  z-index: 20;
  display: grid;
  place-items: center;
  padding: 2rem;
  background:
    linear-gradient(rgba(0, 0, 0, 0.72), rgba(0, 0, 0, 0.82)),
    repeating-linear-gradient(
      0deg,
      rgba(36, 231, 255, 0.04) 0 1px,
      transparent 1px 12px
    );
}

.skill-modal__panel {
  display: flex;
  flex-direction: column;
  width: min(58rem, 92vw);
  max-height: min(42rem, 86vh);
  border: 1px solid rgba(36, 231, 255, 0.58);
  border-radius: 0.9rem;
  background:
    linear-gradient(180deg, rgba(3, 18, 22, 0.98), rgba(1, 7, 10, 0.98)),
    #02080a;
  box-shadow:
    0 0 0 1px rgba(36, 231, 255, 0.12) inset,
    0 1.5rem 4rem rgba(0, 0, 0, 0.58),
    0 0 2.5rem rgba(36, 231, 255, 0.16);
  overflow: hidden;
}

.skill-modal__header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 1rem;
  padding: 1rem 1.1rem;
  border-bottom: 1px solid rgba(36, 231, 255, 0.22);
}

.skill-modal__actions {
  display: flex;
  align-items: center;
  gap: 0.55rem;
}

.skill-modal__eyebrow {
  margin: 0 0 0.2rem;
  color: rgba(36, 231, 255, 0.62);
  font-size: 0.68rem;
  letter-spacing: 0.22em;
}

.skill-modal__header h3 {
  margin: 0;
  color: #24eaff;
  font-size: 1.25rem;
}

.skill-modal__close {
  display: grid;
  place-items: center;
  width: 2.2rem;
  height: 2.2rem;
  border: 1px solid rgba(36, 231, 255, 0.6);
  border-radius: 0.55rem;
  background: rgba(8, 72, 86, 0.35);
  color: #24eaff;
  font: inherit;
}

.skill-modal__copy {
  padding: 0.5rem 0.85rem;
  border: 1px solid rgba(36, 231, 255, 0.6);
  border-radius: 0.55rem;
  background: rgba(8, 72, 86, 0.35);
  color: #24eaff;
  font: inherit;
}

.skill-modal__copied {
  color: #67ff94;
  font-size: 0.72rem;
  letter-spacing: 0.12em;
}

.skill-modal__content {
  flex: 1;
  min-height: 0;
  margin: 0;
  padding: 1.1rem;
  overflow: auto;
  color: rgba(232, 255, 255, 0.9);
  font: inherit;
  font-size: 0.88rem;
  line-height: 1.75;
  white-space: pre-wrap;
}

.skill-modal__content::-webkit-scrollbar {
  width: 0.5rem;
}

.skill-modal__content::-webkit-scrollbar-thumb {
  border-radius: 999px;
  background: rgba(36, 231, 255, 0.32);
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
