<script setup lang="ts">
import { ref, computed } from 'vue'
import { useI18n } from 'vue-i18n'
import api from '@/api'
import { useAuthStore } from '@/stores/auth'
import { useThemeStore } from '@/stores/theme'
import { useToast } from '@/stores/toast'
import UserAvatar from '@/components/UserAvatar.vue'
import ChangeEmailModal from '@/components/profile/ChangeEmailModal.vue'
import type { User } from '@/types/api'

const { t } = useI18n()
const authStore = useAuthStore()
const themeStore = useThemeStore()
const toast = useToast()
const userDetails = ref<User | null>(null)
const loading = ref<boolean>(true)

const AVATAR_URL_PATTERN = /^https:\/\/cdn\.nodeimage\.com\/i\/.+\.(jpeg|jpg|png|webp)$/i

// 头像模式: 'style' = DiceBear 风格, 'customUrl' = 自定义图片
const avatarMode = ref<'style' | 'customUrl'>(
  authStore.user?.avatarUrl ? 'customUrl' : 'style'
)

// DiceBear 风格相关
const avatarStyles = [
  'adventurer', 'adventurerNeutral', 'avataaars', 'avataaarsNeutral',
  'bigEars', 'bigEarsNeutral', 'bigSmile', 'bottts', 'botttsNeutral',
  'croodles', 'croodlesNeutral', 'dylan', 'funEmoji', 'glass', 'icons',
  'identicon', 'initials', 'lorelei', 'loreleiNeutral', 'micah', 'miniavs',
  'notionists', 'notionistsNeutral', 'openPeeps', 'personas', 'pixelArt',
  'pixelArtNeutral', 'rings', 'shapes', 'thumbs'
] as const

const selectedAvatarStyle = ref<string>(authStore.user?.avatarStyle || '')
const savingAvatar = ref(false)
const avatarExpanded = ref(false)
const showChangeEmailModal = ref(false)

// 自定义头像 URL 相关
const customAvatarUrl = ref<string>(authStore.user?.avatarUrl || '')
const savingAvatarUrl = ref(false)

function getStyleLabel(style: string): string {
  return t(`profile.avatar.styles.${style}`)
}

const hasAvatarChanged = computed(() => {
  return selectedAvatarStyle.value !== authStore.user?.avatarStyle
})

const isUrlValid = computed(() => {
  if (!customAvatarUrl.value) return false
  return AVATAR_URL_PATTERN.test(customAvatarUrl.value)
})

const hasUrlChanged = computed(() => {
  return customAvatarUrl.value !== (authStore.user?.avatarUrl || '')
})

async function saveAvatarStyle(): Promise<void> {
  if (!hasAvatarChanged.value) return
  savingAvatar.value = true
  try {
    await api.users.update(authStore.user!.id, { avatarStyle: selectedAvatarStyle.value })
    if (authStore.user) {
      authStore.user.avatarStyle = selectedAvatarStyle.value
    }
    toast.success(t('profile.avatar.saveSuccess'))
  } catch (error) {
    console.error('Failed to save avatar style:', error)
    toast.error(t('profile.avatar.saveFailed'))
  } finally {
    savingAvatar.value = false
  }
}

async function saveCustomAvatarUrl(): Promise<void> {
  if (!hasUrlChanged.value || !customAvatarUrl.value) return
  if (!AVATAR_URL_PATTERN.test(customAvatarUrl.value)) return
  savingAvatarUrl.value = true
  try {
    const url = customAvatarUrl.value
    await api.users.update(authStore.user!.id, { avatarUrl: url })
    if (authStore.user) {
      authStore.user.avatarUrl = url
    }
    toast.success(t('profile.avatar.urlSaveSuccess'))
  } catch (error) {
    console.error('Failed to save avatar URL:', error)
    toast.error(t('profile.avatar.urlSaveFailed'))
  } finally {
    savingAvatarUrl.value = false
  }
}

async function clearCustomAvatarUrl(): Promise<void> {
  savingAvatarUrl.value = true
  try {
    await api.users.update(authStore.user!.id, { avatarUrl: null })
    if (authStore.user) {
      authStore.user.avatarUrl = null
    }
    customAvatarUrl.value = ''
    avatarMode.value = 'style'
    toast.success(t('profile.avatar.urlCleared'))
  } catch (error) {
    console.error('Failed to clear avatar URL:', error)
    toast.error(t('profile.avatar.urlSaveFailed'))
  } finally {
    savingAvatarUrl.value = false
  }
}

function switchToMode(mode: 'style' | 'customUrl'): void {
  if (mode === 'customUrl') {
    customAvatarUrl.value = authStore.user?.avatarUrl || ''
  }
  avatarMode.value = mode
}

async function loadUserDetails(): Promise<void> {
  try {
    const response = await api.users.get(authStore.user!.id)
    userDetails.value = (response as { user?: User }).user || null
  } catch (error) {
    console.error('Failed to load user details:', error)
  } finally {
    loading.value = false
  }
}

function handleEmailUpdated(email: string): void {
  if (authStore.user) {
    authStore.user.email = email
  }
  if (userDetails.value) {
    userDetails.value.email = email
  }
  showChangeEmailModal.value = false
}

loadUserDetails()

defineExpose({ loadUserDetails })
</script>

<template>
  <!-- 账户信息 -->
  <div class="card p-5">
    <div class="flex items-start justify-between mb-4">
      <h2 class="text-sm font-medium text-themed-secondary">{{ $t('profile.account.title') }}</h2>
      <UserAvatar 
        :username="authStore.user?.username || ''" 
        :email="authStore.user?.email"
        :avatar-style="authStore.user?.avatarStyle || ''"
        :size="56"
      />
    </div>
    <dl class="space-y-0 text-sm">
      <div class="flex items-center justify-between py-3 border-b border-themed">
        <dt class="text-themed-muted">{{ $t('profile.account.username') }}</dt>
        <dd class="flex flex-wrap items-center justify-end gap-2 text-themed font-medium">
          <span>{{ authStore.user?.username }}</span>
          <span
            class="inline-flex items-center rounded-full px-2 py-0.5 text-xs font-medium"
            :class="themeStore.isDark ? 'bg-gray-800 text-gray-300' : 'bg-gray-100 text-gray-600'"
          >
            {{ $t('profile.account.uid') }} {{ authStore.user?.id }}
          </span>
        </dd>
      </div>
      <div class="flex items-center justify-between py-3 border-b border-themed">
        <dt class="text-themed-muted">{{ $t('profile.account.role') }}</dt>
        <dd>
          <span :class="['badge', authStore.isAdmin ? 'badge-warning' : 'badge-success']">
            {{ authStore.isAdmin ? $t('profile.account.admin') : $t('profile.account.user') }}
          </span>
        </dd>
      </div>
      <div class="flex items-center justify-between py-3 border-b border-themed">
        <dt class="text-themed-muted">{{ $t('profile.account.email') }}</dt>
        <dd class="flex flex-wrap items-center justify-end gap-3">
          <span class="text-themed-secondary">{{ authStore.user?.email || $t('profile.account.notSet') }}</span>
          <button
            type="button"
            class="btn-secondary btn-sm"
            @click="showChangeEmailModal = true"
          >
            {{ authStore.user?.email ? $t('profile.account.changeEmail') : $t('profile.account.bindEmail') }}
          </button>
        </dd>
      </div>
      <!-- 头像风格入口 -->
      <div class="py-3">
        <button
          class="w-full flex items-center justify-between text-left"
          @click="avatarExpanded = !avatarExpanded"
        >
          <span class="text-themed-muted">{{ $t('profile.avatar.title') }}</span>
          <svg
            class="w-4 h-4 text-themed-muted transition-transform duration-200"
            :class="{ 'rotate-180': avatarExpanded }"
            fill="none"
            stroke="currentColor"
            viewBox="0 0 24 24"
          >
            <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M19 9l-7 7-7-7" />
          </svg>
        </button>
        <!-- 展开的头像选择区域 -->
        <div
          v-show="avatarExpanded"
          class="mt-4 pt-4 border-t border-themed"
        >
          <!-- 模式切换 Tab -->
          <div class="flex gap-1 mb-4 p-0.5 rounded-lg bg-gray-100 dark:bg-gray-800 w-fit">
            <button
              class="px-3 py-1.5 text-sm rounded-md transition-all"
              :class="avatarMode === 'style'
                ? 'bg-white dark:bg-gray-700 text-themed font-medium shadow-sm'
                : 'text-themed-muted hover:text-themed'"
              @click="switchToMode('style')"
            >
              🎨 {{ $t('profile.avatar.tabStyle') }}
            </button>
            <button
              class="px-3 py-1.5 text-sm rounded-md transition-all"
              :class="avatarMode === 'customUrl'
                ? 'bg-white dark:bg-gray-700 text-themed font-medium shadow-sm'
                : 'text-themed-muted hover:text-themed'"
              @click="switchToMode('customUrl')"
            >
              🖼️ {{ $t('profile.avatar.tabCustomUrl') }}
            </button>
          </div>

          <!-- DiceBear 风格模式 -->
          <template v-if="avatarMode === 'style'">
            <div class="flex flex-col sm:flex-row items-start gap-6 mb-4">
              <div class="flex-shrink-0 flex flex-col items-center gap-2">
                <UserAvatar 
                  :username="authStore.user?.username || ''"
                  :email="authStore.user?.email" 
                  :avatar-style="selectedAvatarStyle"
                  :prefer-badge="false"
                  :prefer-custom-url="false"
                  :size="96"
                />
                <span class="text-sm text-themed-secondary">{{ getStyleLabel(selectedAvatarStyle) }}</span>
              </div>
              <div class="flex-1 w-full">
                <div class="grid grid-cols-5 sm:grid-cols-6 md:grid-cols-8 lg:grid-cols-10 gap-2">
                  <button
                    v-for="style in avatarStyles"
                    :key="style"
                    class="flex flex-col items-center gap-1 p-1.5 rounded-lg border-2 transition-all"
                    :class="[
                      selectedAvatarStyle === style 
                        ? 'border-blue-500 bg-blue-500/10' 
                        : 'border-transparent hover:border-gray-300 dark:hover:border-gray-600'
                    ]"
                    :title="getStyleLabel(style)"
                    @click="selectedAvatarStyle = style"
                  >
                    <UserAvatar 
                      :username="authStore.user?.username || ''"
                      :email="authStore.user?.email" 
                      :avatar-style="style"
                      :prefer-badge="false"
                      :prefer-custom-url="false"
                      :size="32"
                    />
                  </button>
                </div>
              </div>
            </div>
            <div class="flex justify-end">
              <button
                class="btn btn-primary text-sm"
                :disabled="!hasAvatarChanged || savingAvatar"
                @click="saveAvatarStyle"
              >
                {{ savingAvatar ? $t('common.saving') : $t('common.save') }}
              </button>
            </div>
          </template>

          <!-- 自定义图片模式 -->
          <template v-if="avatarMode === 'customUrl'">
            <div class="flex flex-col sm:flex-row items-start gap-6 mb-4">
              <div class="flex-shrink-0 flex flex-col items-center gap-2">
                <UserAvatar
                  :username="authStore.user?.username || ''"
                  :email="authStore.user?.email"
                  :avatar-url="customAvatarUrl || null"
                  :prefer-badge="false"
                  :size="96"
                />
                <span v-if="authStore.user?.avatarUrl" class="text-xs text-themed-muted">
                  {{ $t('profile.avatar.currentCustom') }}
                </span>
              </div>
              <div class="flex-1 w-full space-y-3">
                <div>
                  <label class="block text-sm text-themed-secondary mb-1">
                    {{ $t('profile.avatar.urlLabel') }}
                  </label>
                  <input
                    v-model="customAvatarUrl"
                    type="url"
                    class="input w-full text-sm"
                    :class="customAvatarUrl && !isUrlValid ? 'border-red-500' : ''"
                    placeholder="https://cdn.nodeimage.com/i/example.jpeg"
                  />
                  <div v-if="customAvatarUrl && !isUrlValid" class="mt-1 text-xs text-red-500">
                    {{ $t('profile.avatar.urlInvalid') }}
                  </div>
                  <div v-if="isUrlValid" class="mt-1 text-xs text-green-500">
                    {{ $t('profile.avatar.urlValid') }}
                  </div>
                </div>
                <div class="text-xs text-themed-muted leading-relaxed">
                  {{ $t('profile.avatar.urlHint') }}
                </div>
                <div class="flex items-center gap-2">
                  <button
                    class="btn btn-primary text-sm"
                    :disabled="!hasUrlChanged || !isUrlValid || savingAvatarUrl"
                    @click="saveCustomAvatarUrl"
                  >
                    {{ savingAvatarUrl ? $t('common.saving') : $t('common.save') }}
                  </button>
                  <button
                    v-if="authStore.user?.avatarUrl"
                    class="btn-secondary btn-sm text-sm"
                    :disabled="savingAvatarUrl"
                    @click="clearCustomAvatarUrl"
                  >
                    {{ $t('profile.avatar.clearUrl') }}
                  </button>
                </div>
              </div>
            </div>
          </template>
        </div>
      </div>
    </dl>
  </div>

  <ChangeEmailModal
    :show="showChangeEmailModal"
    @close="showChangeEmailModal = false"
    @updated="handleEmailUpdated"
  />
</template>
