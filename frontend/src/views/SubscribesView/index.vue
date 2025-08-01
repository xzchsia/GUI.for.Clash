<script setup lang="ts">
import { computed } from 'vue'
import { useI18n, I18nT } from 'vue-i18n'

import { updateProvidersProxies } from '@/api/kernel'
import { BrowserOpenURL, ClipboardSetText, Removefile } from '@/bridge'
import { DraggableOptions } from '@/constant'
import { View } from '@/enums/app'
import {
  useSubscribesStore,
  useAppSettingsStore,
  useKernelApiStore,
  usePluginsStore,
} from '@/stores'
import {
  formatBytes,
  formatRelativeTime,
  debounce,
  ignoredError,
  formatDate,
  message,
} from '@/utils'

import { useModal } from '@/components/Modal'

import ProxiesEditor from './components/ProxiesEditor.vue'
import ProxiesView from './components/ProxiesView.vue'
import SubscribeForm from './components/SubscribeForm.vue'
import SubscribeScript from './components/SubscribeScript.vue'

import type { Menu, Subscription } from '@/types/app'

const menuList: Menu[] = [
  {
    label: 'subscribes.editProxies',
    handler: (id: string) => handleEditProxies(id),
  },
  {
    label: 'subscribes.editSourceFile',
    handler: (id: string) => handleEditProxies(id, true),
  },
  {
    label: 'subscribes.copySub',
    handler: async (id: string) => {
      const sub = subscribeStore.getSubscribeById(id)!
      if (sub) {
        await ClipboardSetText(sub.url)
        message.success('common.copied')
      }
    },
  },
  {
    label: 'subscribes.script',
    handler: async (id: string) => {
      modalApi.setProps({ title: 'common.edit', width: '90' })
      modalApi.setContent(SubscribeScript, { id }).open()
    },
  },
]

const { t } = useI18n()
const [Modal, modalApi] = useModal({})
const subscribeStore = useSubscribesStore()
const appSettingsStore = useAppSettingsStore()
const kernelApiStore = useKernelApiStore()
const pluginsStore = usePluginsStore()

const generateMenus = (subscription: Subscription) => {
  const builtInMenus: Menu[] = menuList.map((v) => ({
    ...v,
    handler: () => v.handler?.(subscription.id),
  }))

  const contextMenus = pluginsStore.plugins.filter(
    (plugin) => Object.keys(plugin.context.subscriptions).length !== 0,
  )

  if (contextMenus.length !== 0) {
    builtInMenus.push(
      {
        label: '',
        separator: true,
      },
      {
        label: 'common.more',
        children: contextMenus.reduce((prev, plugin) => {
          const menus = Object.entries(plugin.context.subscriptions)
          return prev.concat(
            menus.map(([title, fn]) => {
              return {
                label: title,
                handler: async () => {
                  try {
                    plugin.running = true
                    await pluginsStore.manualTrigger(plugin.id, fn as any, subscription)
                  } catch (error: any) {
                    message.error(error)
                  } finally {
                    plugin.running = false
                  }
                },
              }
            }),
          )
        }, [] as Menu[]),
      },
    )
  }

  return builtInMenus
}

const handleShowSubForm = (id?: string) => {
  modalApi.setProps({
    title: id ? 'common.edit' : 'common.add',
    minWidth: '70',
  })
  modalApi.setContent(SubscribeForm, { id }).open()
}

const handleUpdateSubs = async () => {
  try {
    await subscribeStore.updateSubscribes()
    const { running, profile } = appSettingsStore.app.kernel
    if (running && subscribeStore.getSubscribeById(profile)?.useInternal) {
      await kernelApiStore.restartKernel()
    } else {
      await _updateAllProviderProxies()
    }
    message.success('common.success')
  } catch (error: any) {
    console.error('updateSubscribes: ', error)
    message.error(error)
  }
}

const handleEditProxies = (id: string, editor = false) => {
  const sub = subscribeStore.getSubscribeById(id)
  if (sub) {
    modalApi.setProps({
      title: sub.name,
      height: '90',
      width: '90',
      onOk: async () => {
        try {
          await _updateProviderProxies(sub.id)
        } catch (error: any) {
          console.error(error)
          message.error(error)
        }
      },
    })
    modalApi.setContent(editor ? ProxiesEditor : ProxiesView, { sub }).open()
  }
}

const handleUpdateSub = async (s: Subscription) => {
  try {
    await subscribeStore.updateSubscribe(s.id)
    const { running, profile } = appSettingsStore.app.kernel
    if (running && s.useInternal && profile === s.id) {
      await kernelApiStore.restartKernel()
    } else {
      await _updateProviderProxies(s.id)
    }
  } catch (error: any) {
    console.error('updateSubscribe: ', error)
    message.error(error)
  }
}

const handleDeleteSub = async (s: Subscription) => {
  try {
    await ignoredError(Removefile, s.path)
    await subscribeStore.deleteSubscribe(s.id)
  } catch (error: any) {
    console.error('deleteSubscribe: ', error)
    message.error(error)
  }
}

const handleDisableSub = async (s: Subscription) => {
  s.disabled = !s.disabled
  subscribeStore.editSubscribe(s.id, s)
}

const _updateProviderProxies = async (provider: string) => {
  if (appSettingsStore.app.kernel.running) {
    await kernelApiStore.refreshProviderProxies()
    if (kernelApiStore.providers[provider]) {
      await updateProvidersProxies(provider)
      await kernelApiStore.refreshProviderProxies()
    }
  }
}

const _updateAllProviderProxies = async () => {
  if (appSettingsStore.app.kernel.running) {
    await kernelApiStore.refreshProviderProxies()
    const ids = Object.keys(kernelApiStore.providers).filter(
      (v) => v !== 'default' && !kernelApiStore.proxies[v],
    )
    for (let i = 0; i < ids.length; i++) {
      await updateProvidersProxies(ids[i])
    }
    if (ids.length !== 0) {
      await kernelApiStore.refreshProviderProxies()
    }
  }
}

const noUpdateNeeded = computed(() => subscribeStore.subscribes.every((v) => v.disabled))

const clacTrafficPercent = (s: Subscription) => ((s.upload + s.download) / s.total) * 100

const clacTrafficStatus = (s: Subscription) => {
  const percent = clacTrafficPercent(s)
  if (percent > 90) return 'danger'
  if (percent > 80) return 'warning'
  return 'primary'
}

const onSortUpdate = debounce(subscribeStore.saveSubscribes, 1000)
</script>

<template>
  <div v-if="subscribeStore.subscribes.length === 0" class="grid-list-empty">
    <Empty>
      <template #description>
        <I18nT keypath="subscribes.empty" tag="div" scope="global" class="flex items-center mt-12">
          <template #action>
            <Button @click="handleShowSubForm()" type="link">{{ t('common.add') }}</Button>
          </template>
        </I18nT>
      </template>
    </Empty>
  </div>

  <div v-else class="grid-list-header">
    <Radio
      v-model="appSettingsStore.app.subscribesView"
      :options="[
        { label: 'common.grid', value: View.Grid },
        { label: 'common.list', value: View.List },
      ]"
      class="mr-auto"
    />
    <Button
      @click="handleUpdateSubs"
      :disabled="noUpdateNeeded"
      :type="noUpdateNeeded ? 'text' : 'link'"
    >
      {{ t('common.updateAll') }}
    </Button>
    <Button @click="handleShowSubForm()" type="primary" icon="add" class="ml-16">
      {{ t('common.add') }}
    </Button>
  </div>

  <div
    v-draggable="[subscribeStore.subscribes, { ...DraggableOptions, onUpdate: onSortUpdate }]"
    :class="'grid-list-' + appSettingsStore.app.subscribesView"
  >
    <Card
      v-for="s in subscribeStore.subscribes"
      :key="s.id"
      :title="s.name"
      :disabled="s.disabled"
      v-menu="generateMenus(s)"
      class="grid-list-item"
    >
      <template #title-prefix>
        <Tag v-if="s.updating" color="cyan">
          {{ t('subscribe.updating') }}
        </Tag>
      </template>

      <template #title-suffix>
        <Icon
          v-if="s.type !== 'File' && s.website"
          v-tips="'subscribe.website'"
          icon="link"
          :size="18"
          @click="BrowserOpenURL(s.website)"
          style="cursor: pointer"
        />
      </template>

      <template v-if="appSettingsStore.app.subscribesView === View.Grid" #extra>
        <Dropdown :trigger="['hover', 'click']">
          <Button type="link" size="small" icon="more" />
          <template #overlay>
            <div class="flex flex-col gap-4 min-w-64 p-4">
              <Button
                :disabled="s.disabled"
                :loading="s.updating"
                type="text"
                size="small"
                @click="handleUpdateSub(s)"
              >
                {{ t('common.update') }}
              </Button>
              <Button type="text" size="small" @click="handleDisableSub(s)">
                {{ s.disabled ? t('common.enable') : t('common.disable') }}
              </Button>
              <Button type="text" size="small" @click="handleShowSubForm(s.id)">
                {{ t('common.edit') }}
              </Button>
              <Button type="text" size="small" @click="handleDeleteSub(s)">
                {{ t('common.delete') }}
              </Button>
            </div>
          </template>
        </Dropdown>
      </template>

      <template v-else #extra>
        <Button
          :disabled="s.disabled"
          :loading="s.updating"
          type="text"
          size="small"
          @click="handleUpdateSub(s)"
        >
          {{ t('common.update') }}
        </Button>
        <Button type="text" size="small" @click="handleDisableSub(s)">
          {{ s.disabled ? t('common.enable') : t('common.disable') }}
        </Button>
        <Button type="text" size="small" @click="handleShowSubForm(s.id)">
          {{ t('common.edit') }}
        </Button>
        <Button type="text" size="small" @click="handleDeleteSub(s)">
          {{ t('common.delete') }}
        </Button>
      </template>
      <template v-if="appSettingsStore.app.subscribesView === View.List">
        <div style="margin-bottom: 8px">
          <Progress :percent="clacTrafficPercent(s)" :status="clacTrafficStatus(s)" />
        </div>
        <div>
          {{ t('subscribes.proxyCount') }}
          :
          {{ s.proxies.length }}
        </div>
        <div>
          {{ t('subscribes.upload') }}
          :
          {{ s.upload ? formatBytes(s.upload, 2) : '--' }}
          /
          {{ t('subscribes.download') }}
          :
          {{ s.download ? formatBytes(s.download, 2) : '--' }}
          /
          {{ t('subscribes.total') }}
          :
          {{ s.total ? formatBytes(s.total, 2) : '--' }}
        </div>
        <div>
          {{ t('subscribes.expire') }}
          :
          {{ s.expire ? formatDate(s.expire, 'YYYY-MM-DD HH:mm:ss') : '--' }}
          /
          {{ t('common.updateTime') }}
          :
          {{ s.updateTime ? formatDate(s.updateTime, 'YYYY-MM-DD HH:mm:ss') : '--' }}
        </div>
      </template>
      <template v-else>
        <div>
          {{ s.upload + s.download ? formatBytes(s.upload + s.download) : '--' }}
          /
          {{ s.total ? formatBytes(s.total) : '--' }}
        </div>
        <div class="traffic-diagram">
          <Progress
            :percent="clacTrafficPercent(s)"
            :status="clacTrafficStatus(s)"
            type="circle"
            :radius="20"
          />
        </div>
        <div>
          {{ t('subscribes.expire') }}
          :
          {{ s.expire ? formatRelativeTime(s.expire) : '--' }}
        </div>
        <div>
          {{ t('common.updateTime') }}
          :
          {{ s.updateTime ? formatRelativeTime(s.updateTime) : '--' }}
        </div>
      </template>
    </Card>
  </div>

  <Modal />
</template>

<style lang="less" scoped>
.traffic-diagram {
  position: absolute;
  top: 40px;
  right: 12px;
}
</style>
