<script setup lang="ts">
import { ref, onMounted, computed } from 'vue'
import { NButton, NTabs, NTabPane, useMessage } from 'naive-ui'
import { useI18n } from 'vue-i18n'
import MarkdownRenderer from '@/components/hermes/chat/MarkdownRenderer.vue'
import { fetchMemory, saveMemory, type MemoryData } from '@/api/hermes/skills'
import { useProfilesStore } from '@/stores/hermes/profiles'

const { t } = useI18n()
const message = useMessage()
const profilesStore = useProfilesStore()
const loading = ref(false)
const data = ref<MemoryData | null>(null)
const activeTab = ref<'memory' | 'user' | 'soul'>('memory')
const editingSection = ref<'memory' | 'user' | 'soul' | null>(null)
const editContent = ref('')
const saving = ref(false)

onMounted(loadMemory)

async function loadMemory() {
  loading.value = true
  try {
    if (!profilesStore.activeProfileName || profilesStore.profiles.length === 0) {
      await profilesStore.fetchProfiles()
    }
    data.value = await fetchMemory()
  } catch (err: any) {
    console.error('Failed to load memory:', err)
    message.error(t('memory.loadFailed'))
  } finally {
    loading.value = false
  }
}

function startEdit() {
  if (!activeTab.value) return
  editingSection.value = activeTab.value
  editContent.value = data.value?.[activeTab.value] || ''
}

function cancelEdit() {
  editingSection.value = null
  editContent.value = ''
}

async function handleSave() {
  if (!editingSection.value) return
  saving.value = true
  try {
    await saveMemory(editingSection.value, editContent.value)
    await loadMemory()
    editingSection.value = null
    editContent.value = ''
    message.success(t('common.saved'))
  } catch (err: any) {
    message.error(`${t('common.saveFailed')}: ${err.message}`)
  } finally {
    saving.value = false
  }
}

function formatTime(ts: number | null): string {
  if (!ts) return ''
  return new Date(ts).toLocaleString([], {
    month: 'short',
    day: 'numeric',
    hour: '2-digit',
    minute: '2-digit',
  })
}

const memoryEmpty = computed(() => !data.value?.memory?.trim())
const userEmpty = computed(() => !data.value?.user?.trim())
const soulEmpty = computed(() => !data.value?.soul?.trim())

const displayMemory = computed(() => (data.value?.memory || '').replace(/§/g, '\n\n'))
const displayUser = computed(() => (data.value?.user || '').replace(/§/g, '\n\n'))
const displaySoul = computed(() => (data.value?.soul || '').replace(/§/g, '\n\n'))
</script>

<template>
  <div class="memory-view">
    <header class="page-header">
      <h2 class="header-title">{{ t('memory.title') }}</h2>
      <NButton size="small" quaternary @click="loadMemory">
        <template #icon>
          <svg width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
            <polyline points="23 4 23 10 17 10" />
            <path d="M20.49 15a9 9 0 1 1-2.12-9.36L23 10" />
          </svg>
        </template>
        {{ t('memory.refresh') }}
      </NButton>
    </header>

    <div class="memory-content">
      <div v-if="loading && !data" class="memory-loading">{{ t('common.loading') }}</div>
      <div v-else class="memory-tabs">
        <NTabs v-model:value="activeTab" type="line" @update:value="cancelEdit">
          <NTabPane name="memory" :tab="t('memory.myNotes')">
            <div class="memory-section">
              <div class="section-toolbar">
                <span v-if="data?.memory_mtime" class="section-mtime">{{ formatTime(data.memory_mtime) }}</span>
                <NButton v-if="editingSection !== 'memory'" size="tiny" quaternary @click="startEdit">
                  <template #icon>
                    <svg width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.5">
                      <path d="M11 4H4a2 2 0 0 0-2 2v14a2 2 0 0 0 2 2h14a2 2 0 0 0 2-2v-7" />
                      <path d="M18.5 2.5a2.121 2.121 0 0 1 3 3L12 15l-4 1 1-4 9.5-9.5z" />
                    </svg>
                  </template>
                  {{ t('common.edit') }}
                </NButton>
              </div>

              <!-- View mode -->
              <div v-if="editingSection !== 'memory'" class="section-body">
                <MarkdownRenderer v-if="!memoryEmpty" :content="displayMemory" />
                <p v-else class="empty-text">{{ t('memory.noNotes') }}</p>
              </div>

              <!-- Edit mode -->
              <div v-else class="section-edit">
                <textarea
                  v-model="editContent"
                  class="edit-textarea"
                  :placeholder="t('memory.notesPlaceholder')"
                  spellcheck="false"
                ></textarea>
                <div class="edit-actions">
                  <NButton size="small" @click="cancelEdit">{{ t('common.cancel') }}</NButton>
                  <NButton size="small" type="primary" :loading="saving" @click="handleSave">{{ t('common.save') }}</NButton>
                </div>
              </div>
            </div>
          </NTabPane>

          <NTabPane name="user" :tab="t('memory.userProfile')">
            <div class="memory-section">
              <div class="section-toolbar">
                <span v-if="data?.user_mtime" class="section-mtime">{{ formatTime(data.user_mtime) }}</span>
                <NButton v-if="editingSection !== 'user'" size="tiny" quaternary @click="startEdit">
                  <template #icon>
                    <svg width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.5">
                      <path d="M11 4H4a2 2 0 0 0-2 2v14a2 2 0 0 0 2 2h14a2 2 0 0 0 2-2v-7" />
                      <path d="M18.5 2.5a2.121 2.121 0 0 1 3 3L12 15l-4 1 1-4 9.5-9.5z" />
                    </svg>
                  </template>
                  {{ t('common.edit') }}
                </NButton>
              </div>

              <!-- View mode -->
              <div v-if="editingSection !== 'user'" class="section-body">
                <MarkdownRenderer v-if="!userEmpty" :content="displayUser" />
                <p v-else class="empty-text">{{ t('memory.noProfile') }}</p>
              </div>

              <!-- Edit mode -->
              <div v-else class="section-edit">
                <textarea
                  v-model="editContent"
                  class="edit-textarea"
                  :placeholder="t('memory.profilePlaceholder')"
                  spellcheck="false"
                ></textarea>
                <div class="edit-actions">
                  <NButton size="small" @click="cancelEdit">{{ t('common.cancel') }}</NButton>
                  <NButton size="small" type="primary" :loading="saving" @click="handleSave">{{ t('common.save') }}</NButton>
                </div>
              </div>
            </div>
          </NTabPane>

          <NTabPane name="soul" :tab="t('memory.soul')">
            <div class="memory-section">
              <div class="section-toolbar">
                <span v-if="data?.soul_mtime" class="section-mtime">{{ formatTime(data.soul_mtime) }}</span>
                <NButton v-if="editingSection !== 'soul'" size="tiny" quaternary @click="startEdit">
                  <template #icon>
                    <svg width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.5">
                      <path d="M11 4H4a2 2 0 0 0-2 2v14a2 2 0 0 0 2 2h14a2 2 0 0 0 2-2v-7" />
                      <path d="M18.5 2.5a2.121 2.121 0 0 1 3 3L12 15l-4 1 1-4 9.5-9.5z" />
                    </svg>
                  </template>
                  {{ t('common.edit') }}
                </NButton>
              </div>

              <!-- View mode -->
              <div v-if="editingSection !== 'soul'" class="section-body">
                <MarkdownRenderer v-if="!soulEmpty" :content="displaySoul" />
                <p v-else class="empty-text">{{ t('memory.noSoul') }}</p>
              </div>

              <!-- Edit mode -->
              <div v-else class="section-edit">
                <textarea
                  v-model="editContent"
                  class="edit-textarea"
                  :placeholder="t('memory.soulPlaceholder')"
                  spellcheck="false"
                ></textarea>
                <div class="edit-actions">
                  <NButton size="small" @click="cancelEdit">{{ t('common.cancel') }}</NButton>
                  <NButton size="small" type="primary" :loading="saving" @click="handleSave">{{ t('common.save') }}</NButton>
                </div>
              </div>
            </div>
          </NTabPane>
        </NTabs>
      </div>
    </div>
  </div>
</template>

<style scoped lang="scss">
@use '@/styles/variables' as *;

.memory-view {
  height: calc(100 * var(--vh));
  display: flex;
  flex-direction: column;
}

.memory-content {
  flex: 1;
  overflow: hidden;
  padding: 20px;
  display: flex;
  flex-direction: column;
}

.memory-loading {
  flex: 1;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 13px;
  color: $text-muted;
}

.memory-tabs {
  flex: 1;
  min-height: 0;
  display: flex;
  flex-direction: column;

  :deep(.n-tabs) {
    flex: 1;
    display: flex;
    flex-direction: column;
    min-height: 0;

    .n-tabs-nav {
      flex-shrink: 0;
    }

    .n-tabs-content {
      flex: 1;
      min-height: 0;

      .n-tab-pane {
        height: 100%;
      }
    }
  }
}

.memory-section {
  height: 100%;
  display: flex;
  flex-direction: column;
}

.section-toolbar {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 8px 0 4px;
  flex-shrink: 0;
}

.section-mtime {
  font-size: 11px;
  color: $text-muted;
}

.section-body {
  flex: 1;
  overflow-y: auto;
  padding: 16px;
  min-height: 0;
}

.empty-text {
  color: $text-muted;
  font-style: italic;
  font-size: 13px;
}

.section-edit {
  flex: 1;
  display: flex;
  flex-direction: column;
  padding: 12px 16px;
  min-height: 0;
}

.edit-textarea {
  flex: 1;
  width: 100%;
  min-height: 0;
  padding: 12px;
  border: 1px solid $border-color;
  border-radius: $radius-sm;
  background: $bg-input;
  color: $text-primary;
  font-family: $font-code;
  font-size: 13px;
  line-height: 1.6;
  resize: none;
  outline: none;

  &:focus {
    border-color: $accent-primary;
  }
}

.edit-actions {
  display: flex;
  justify-content: flex-end;
  gap: 8px;
  margin-top: 10px;
}
</style>
