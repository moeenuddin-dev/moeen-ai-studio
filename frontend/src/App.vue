<script setup lang="ts">
import { ref, onMounted } from 'vue'
import { t, LANGS, useI18n } from '@/i18n'
import { useTheme } from '@/composables/useTheme'
import { useToast } from '@/composables/useToast'
import { useConfig } from '@/composables/useConfig'
import { useVoice } from '@/composables/useVoice'
import { useTasks } from '@/composables/useTasks'
import { useProgress } from '@/composables/useProgress'
import { useNavigation } from '@/composables/useNavigation'
import { appState } from '@/store'
import ConfigPanel from '@/components/ConfigPanel.vue'
import CreatePanel from '@/components/CreatePanel.vue'
import SimplePanel from '@/components/SimplePanel.vue'
import TaskListPanel from '@/components/TaskListPanel.vue'
import ProgressPage from '@/components/ProgressPage.vue'
import VoicePickerModal from '@/components/VoicePickerModal.vue'
import Toast from '@/components/Toast.vue'
import ConfirmModal from '@/components/ConfirmModal.vue'

const { switchLang } = useI18n()
const { themeIcon, themeLabel, cycleTheme } = useTheme()
const { visible: toastVisible, message: toastMessage, type: toastType } = useToast()
const { loadModels, renderWorkspaces } = useConfig()
const { initVoiceSelector } = useVoice()
const { loadTaskList, startTaskListTimer, stopTaskListTimer } = useTasks()
const { parseHash } = useNavigation()

function switchMainTab(tab: 'create' | 'list' | 'simple') {
  appState.view = tab
  location.hash = tab === 'list' ? '#/list' : tab === 'simple' ? '#/simple' : '#/create'
  if (tab === 'list') {
    loadTaskList()
    startTaskListTimer()
  } else {
    stopTaskListTimer()
  }
}

const isConfigLoaded = ref(false)

onMounted(async () => {
  // 解析 hash：直达进度页 / 列表页（刷新保留视图）
  const parsed = parseHash()
  if (parsed.view === 'progress' && parsed.taskId) {
    appState.view = 'progress'
    appState.progressTaskId = parsed.taskId
    appState.currentTaskId = parsed.taskId
    // 其余恢复逻辑由 ProgressPage 挂载时统一处理
  } else {
    appState.view = parsed.view
  }

  try {
    const cfg = await fetch('/api/config').then((r) => r.json())
    if (cfg.api_key) {
      appState.apiKeySource = cfg.source
    }
    // v6.1 问题反馈：记录应用版本（诊断信息用；缺失时 FeedbackPanel 自兜底拉取）
    if (cfg.app_version) {
      appState.appVersion = cfg.app_version
    }
    await renderWorkspaces()
    if (cfg.watermark !== undefined) {
      appState.watermarkEnabled = !!cfg.watermark.enabled
    }
    if (cfg.agnes_domain) {
      appState.agnesDomain = cfg.agnes_domain
    }
    await loadModels()
    isConfigLoaded.value = true
  } catch (e) {
    console.error('init config load error:', e)
  }

  try {
    await initVoiceSelector()
  } catch (e) {
    console.error('init voice selector error:', e)
  }

  // 自动重连运行中的任务（已在进度页时跳过，由 ProgressPage 恢复）
  if (appState.view !== 'progress') {
    autoReconnectRunningTask()
  }
})

async function autoReconnectRunningTask() {
  try {
    const d = await fetch('/api/tasks').then((r) => r.json())
    const running = (d.tasks || []).find((t: any) => t.status === 'running' || t.status === 'queued')
    if (running) {
      appState.currentTaskType = running.task_type || 'creative'
      appState.currentDirName = running.dir_name || running.task_id
      appState.progressTaskId = running.task_id
      appState.progressOrigin = 'create'
      appState.view = 'progress'
      location.hash = '#/progress/' + encodeURIComponent(running.task_id)
    }
  } catch {
    /* ignore */
  }
}
</script>

<template>
  <ProgressPage v-if="appState.view === 'progress'" />

  <div v-else class="flex justify-center gap-3 px-4">

    <!-- Main content -->
    <div class="max-w-4xl flex-1 min-w-0 py-8">
      <!-- Header -->
      <div class="text-center mb-10">
        <div class="flex items-center justify-end gap-2 mb-5">
          <button
            class="glass-input rounded-lg px-3 py-1.5 text-sm cursor-pointer text-ink flex items-center gap-1.5 hover:border-accent/40 transition"
            :title="themeLabel"
            :aria-label="themeLabel"
            @click="cycleTheme"
          >
            <svg class="w-4 h-4 shrink-0" fill="none" stroke="currentColor" viewBox="0 0 24 24" aria-hidden="true">
              <path stroke-linecap="round" stroke-linejoin="round" stroke-width="1.8" :d="themeIcon"></path>
            </svg>
            <span class="text-xs whitespace-nowrap">{{ themeLabel }}</span>
          </button>
          <select
            class="glass-input rounded-lg px-3 py-1.5 text-sm cursor-pointer text-ink"
            @change="switchLang(($event.target as HTMLSelectElement).value)"
          >
            <option v-for="l in LANGS" :key="l.code" :value="l.code">{{ l.label }}</option>
          </select>
        </div>
        <h1 class="text-2xl sm:text-4xl font-bold text-ink px-2" style="position: relative; z-index: 0">Moeen AI Studio</h1>
        <p class="text-muted mt-2 text-sm tracking-wide px-2">{{ t('subtitle') }}</p>
      </div>


      <!-- Config Panel -->
      <ConfigPanel />

      <!-- Main Tabs -->
      <div class="flex gap-2 mb-6">
        <button
          class="px-5 py-2.5 rounded-lg text-sm font-medium transition"
          :class="appState.view === 'create' ? 'tab-active' : 'tab-inactive'"
          @click="switchMainTab('create')"
        >
          {{ t('tabCreate') }}
        </button>
        <button
          class="px-5 py-2.5 rounded-lg text-sm font-medium transition"
          :class="appState.view === 'list' ? 'tab-active' : 'tab-inactive'"
          @click="switchMainTab('list')"
        >
          {{ t('tabList') }}
        </button>
        <button
          class="px-5 py-2.5 rounded-lg text-sm font-medium transition"
          :class="appState.view === 'simple' ? 'tab-active' : 'tab-inactive'"
          @click="switchMainTab('simple')"
        >
          {{ t('tabSimple') }} <span class="tab-badge">{{ t('tabNewBadge') }}</span>
        </button>
      </div>

      <!-- Create Panel -->
      <div v-show="appState.view === 'create'">
        <CreatePanel />
      </div>

      <!-- Simple Panel -->
      <div v-show="appState.view === 'simple'">
        <SimplePanel @go-list="switchMainTab('list')" />
      </div>

      <!-- List Panel -->
      <div v-show="appState.view === 'list'">
        <TaskListPanel />
      </div>


      </div>
      </div> 


  <!-- Voice Picker Modal -->
  <VoicePickerModal />

  <!-- Toast / Confirm -->
  <Toast :visible="toastVisible" :message="toastMessage" :type="toastType" />
  <ConfirmModal />
</template>
