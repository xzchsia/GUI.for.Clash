<script setup lang="ts">
import { ref, computed, onActivated } from 'vue'
import { useI18n } from 'vue-i18n'

import { getProxyDelay, deleteGroupFixed } from '@/api/kernel'
import { BuiltInOutbound } from '@/constant'
import { ControllerCloseModeOptions, DefaultConcurrencyLimit, DefaultTestURL } from '@/constant/app'
import { ControllerCloseMode } from '@/enums/app'
import { ProxyGroupType } from '@/enums/kernel'
import { useBool } from '@/hooks'
import { useAppSettingsStore, useKernelApiStore } from '@/stores'
import { ignoredError, sleep, handleUseProxy, message, prompt, asyncPool } from '@/utils'

const expandedSet = ref<Set<string>>(new Set())
const loadingSet = ref<Set<string>>(new Set())
const filterKeywordsMap = ref<Record<string, string>>({})

const loading = ref(false)

const { t } = useI18n()
const [showMoreSettings, toggleMoreSettings] = useBool(false)
const appSettings = useAppSettingsStore()
const kernelApiStore = useKernelApiStore()

const groups = computed(() => {
  const { providers, proxies } = kernelApiStore
  if (!providers.default) return []
  return providers.default.proxies
    .concat([proxies.GLOBAL])
    .filter((v) => v.all && !v.hidden)
    .map((group) => {
      const all = group.all
        .filter((proxy) => {
          const condition1 =
            appSettings.app.kernel.unAvailable ||
            BuiltInOutbound.includes(proxy) ||
            proxies[proxy].all ||
            proxies[proxy].alive
          const keywords = filterKeywordsMap.value[group.name]
          const condition2 = keywords ? new RegExp(keywords, 'i').test(proxy) : true
          return condition1 && condition2
        })
        .map((proxy) => {
          const history = proxies[proxy].history || []
          const delay = history[history.length - 1]?.delay || 0
          return { ...proxies[proxy], delay }
        })
        .sort((a, b) => {
          if (!appSettings.app.kernel.sortByDelay || a.delay === b.delay) return 0
          if (!a.delay) return 1
          if (!b.delay) return -1
          return a.delay - b.delay
        })

      const chains = []
      if (group.type === ProxyGroupType.Relay) {
        chains.push(...group.all)
      } else {
        chains.push(group.now)
        let tmp = proxies[group.now]
        while (tmp) {
          tmp.now && chains.push(tmp.now)
          tmp = proxies[tmp.now]
        }
      }

      return { ...group, all, chains }
    })
})

const useProxyWithCatchError = (group: any, proxy: any) => {
  handleUseProxy(group, proxy).catch((err: any) => message.error(err.message || err))
}

const toggleExpanded = (group: string) => {
  if (expandedSet.value.has(group)) {
    expandedSet.value.delete(group)
  } else {
    expandedSet.value.add(group)
  }
}

const handleFilter = async (group: string) => {
  const keywords =
    (await ignoredError(prompt<string>, 'Tips', filterKeywordsMap.value[group], {
      placeholder: 'keywords',
    })) || ''
  try {
    new RegExp(keywords, 'i')
  } catch {
    message.error('Incorrect regular expression')
    await handleFilter(group)
    return
  }
  filterKeywordsMap.value[group] = keywords
}

const expandAll = () => groups.value.forEach(({ name }) => expandedSet.value.add(name))

const collapseAll = () => expandedSet.value.clear()

const isExpanded = (group: string) => expandedSet.value.has(group)

const isLoading = (group: string) => loadingSet.value.has(group)

const isFiltered = (group: string) => filterKeywordsMap.value[group]

const handleGroupDelay = async (group: string) => {
  const _group = kernelApiStore.proxies[group]
  if (_group) {
    let index = 0
    let success = 0
    let failure = 0

    const delayTest = async (proxy: string) => {
      index += 1
      update(`Testing... ${index} / ${_group.all.length}, success: ${success} failure: ${failure}`)
      const _proxy = kernelApiStore.proxies[proxy]
      try {
        loadingSet.value.add(proxy)
        const { delay } = await getProxyDelay(
          encodeURIComponent(proxy),
          appSettings.app.kernel.testUrl || DefaultTestURL,
        )
        success += 1
        _proxy.history.push({ delay })
      } catch {
        failure += 1
        _proxy.history.push({ delay: 0 })
      }
      loadingSet.value.delete(proxy)
    }
    const { update, destroy, success: msgSuccess } = message.info('Testing...', 99999)
    loadingSet.value.add(group)
    await asyncPool(
      appSettings.app.kernel.concurrencyLimit || DefaultConcurrencyLimit,
      _group.all,
      delayTest,
    )
    loadingSet.value.delete(group)
    msgSuccess(
      `Completed. ${index} / ${_group.all.length}, success: ${success} failure: ${failure}`,
    )
    await sleep(3000)
    destroy()
  }
}

const handleProxyDelay = async (proxy: string) => {
  loadingSet.value.add(proxy)
  try {
    const { delay } = await getProxyDelay(
      encodeURIComponent(proxy),
      appSettings.app.kernel.testUrl || DefaultTestURL,
    )
    const _proxy = kernelApiStore.proxies[proxy]
    _proxy.history.push({ delay })
  } catch (error: any) {
    message.error(error + ': ' + proxy)
  }
  loadingSet.value.delete(proxy)
}

const handleRefresh = async () => {
  loading.value = true
  await ignoredError(kernelApiStore.refreshConfig)
  await ignoredError(kernelApiStore.refreshProviderProxies)
  await sleep(100)
  loading.value = false
}

const locateGroup = (group: any, chain: string) => {
  collapseAll()
  if (kernelApiStore.proxies[chain].all) {
    toggleExpanded(kernelApiStore.proxies[chain].name)
  } else {
    toggleExpanded(group.name)
  }
}

const handleClearGroupFixed = async (group: string) => {
  await deleteGroupFixed(group)
  await ignoredError(kernelApiStore.refreshProviderProxies)
}

const delayColor = (delay = 0) => {
  if (delay === 0) return 'var(--level-0-color)'
  if (delay < 200) return 'var(--level-1-color)'
  if (delay < 500) return 'var(--level-2-color)'
  if (delay < 1200) return 'var(--level-3-color)'
  return 'var(--level-4-color)'
}

const handleResetMoreSettings = () => {
  appSettings.app.kernel.testUrl = DefaultTestURL
  appSettings.app.kernel.concurrencyLimit = DefaultConcurrencyLimit
  appSettings.app.kernel.controllerCloseMode = ControllerCloseMode.All
  message.success('common.success')
}

onActivated(() => {
  kernelApiStore.refreshProviderProxies()
})
</script>

<template>
  <div class="m-8 mt-0 sticky top-0 z-3">
    <div
      class="sticky flex gap-8 items-center p-8 rounded-8 backdrop-blur-sm"
      style="background-color: var(--card-bg)"
    >
      <Switch v-model="appSettings.app.kernel.autoClose">
        {{ t('home.controller.autoClose') }}
      </Switch>
      <Switch v-model="appSettings.app.kernel.unAvailable">
        {{ t('home.controller.unAvailable') }}
      </Switch>
      <Switch v-model="appSettings.app.kernel.cardMode">
        {{ t('home.controller.cardMode') }}
      </Switch>
      <Switch v-model="appSettings.app.kernel.sortByDelay">
        {{ t('home.controller.sortBy') }}
      </Switch>
      <Button @click="toggleMoreSettings" type="primary" size="small"> ... </Button>
      <div class="ml-auto">
        <Button @click="expandAll" v-tips="'home.overview.expandAll'" type="text" icon="expand" />
        <Button
          @click="collapseAll"
          v-tips="'home.overview.collapseAll'"
          type="text"
          icon="collapse"
        />
        <Button
          @click="handleRefresh"
          v-tips="'home.overview.refresh'"
          :loading="loading"
          icon="refresh"
          type="text"
        />
      </div>
    </div>
  </div>
  <div v-for="group in groups" :key="group.name" class="m-8">
    <div
      @click="toggleExpanded(group.name)"
      class="sticky z-2 flex gap-8 items-center p-8 rounded-8 backdrop-blur-sm"
      style="top: 52px; background-color: var(--card-bg)"
    >
      <div class="text-14 flex items-center gap-2">
        <img v-if="group.icon" :src="group.icon" class="w-24 h-24 mr-4" draggable="false" />
        <span class="font-bold text-18">{{ group.name }}</span>
        <span class="mx-8">
          {{ group.type }}
        </span>
        <span> :: </span>
        <template v-for="(chain, index) in group.chains" :key="chain">
          <span v-if="index !== 0" style="color: gray"> / </span>
          <Button @click.stop="locateGroup(group, chain)" type="text" size="small">
            {{ chain }}
          </Button>
        </template>
        <Button
          @click.stop="handleClearGroupFixed(group.name)"
          v-if="group.fixed"
          type="text"
          size="small"
          icon="clear"
        />
      </div>
      <div class="ml-auto">
        <Button
          @click.stop="handleFilter(group.name)"
          type="text"
          icon="filter"
          :icon-color="isFiltered(group.name) ? 'var(--primary-color)' : ''"
        />
        <Button
          @click.stop="handleGroupDelay(group.name)"
          v-tips="'home.overview.delayTest'"
          :loading="isLoading(group.name)"
          type="text"
          icon="speedTest"
        />
        <Button @click.stop="toggleExpanded(group.name)" type="text">
          <Icon
            :class="{ 'action-expand-expanded': isExpanded(group.name) }"
            class="action-expand origin-center duration-200"
            icon="arrowDown"
          />
        </Button>
      </div>
    </div>
    <Transition name="expand">
      <div v-if="isExpanded(group.name)" class="py-8 px-4">
        <Empty v-if="group.all.length === 0" />
        <div v-else-if="appSettings.app.kernel.cardMode" class="grid grid-cols-5 gap-8">
          <Card
            v-for="proxy in group.all"
            :title="proxy.name"
            :selected="proxy.name === group.now"
            :key="proxy.name"
            @click="useProxyWithCatchError(group, proxy)"
          >
            <Button
              @click.stop="handleProxyDelay(proxy.name)"
              :style="{ color: delayColor(proxy.delay) }"
              :loading="isLoading(proxy.name)"
              type="text"
              size="small"
              style="margin-left: -2px; padding-left: 2px"
            >
              <div class="text-12">
                {{ proxy.delay && proxy.delay + 'ms' }}
              </div>
            </Button>
            <div class="text-12 my-2">{{ proxy.type }} {{ proxy.udp ? ':: udp' : '' }}</div>
          </Card>
        </div>
        <div v-else class="grid grid-cols-32 gap-8">
          <div
            v-for="proxy in group.all"
            v-tips.fast="proxy.name"
            @click="useProxyWithCatchError(group, proxy)"
            :key="proxy.name"
            :style="{ background: delayColor(proxy.delay) }"
            :class="proxy.name === group.now ? 'rounded-full shadow' : ''"
            class="w-12 h-12 rounded-4 flex items-center justify-center"
          >
            <Icon v-if="isLoading(proxy.name)" icon="loading" :size="12" class="rotation" />
          </div>
        </div>
      </div>
    </Transition>
  </div>

  <Modal
    v-model:open="showMoreSettings"
    :submit="false"
    mask-closable
    cancel-text="common.close"
    title="common.more"
  >
    <template #action>
      <Button @click="handleResetMoreSettings" type="text" class="mr-auto">
        {{ t('common.reset') }}
      </Button>
    </template>

    <div class="form-item">
      {{ t('home.controller.delay') }}
      <Input
        v-model="appSettings.app.kernel.testUrl"
        :placeholder="DefaultTestURL"
        editable
        clearable
      />
    </div>

    <div class="form-item">
      {{ t('home.controller.concurrencyLimit') }}
      <Input
        v-model="appSettings.app.kernel.concurrencyLimit"
        :min="1"
        :max="50"
        type="number"
        editable
        clearable
      />
    </div>

    <div class="form-item">
      {{ t('home.controller.closeMode.name') }}
      <Radio
        v-model="appSettings.app.kernel.controllerCloseMode"
        :options="ControllerCloseModeOptions"
      />
    </div>
  </Modal>
</template>

<style lang="less" scoped>
.expand-enter-active,
.expand-leave-active {
  transform-origin: top;
  transition:
    transform 0.2s ease-in-out,
    opacity 0.2s ease-in-out;
}

.expand-enter-from,
.expand-leave-to {
  transform: scaleY(0);
}

.action-expand {
  transform: rotate(-90deg);
  &-expanded {
    transform: rotate(0deg);
  }
}

.rotation {
  animation: rotate 2s infinite linear;
}
</style>
