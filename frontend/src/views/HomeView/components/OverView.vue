<script setup lang="ts">
import { ref, onUnmounted } from 'vue'
import { useI18n } from 'vue-i18n'

import { ModeOptions } from '@/constant/kernel'
import { useEnvStore, useAppStore, useKernelApiStore } from '@/stores'
import { formatBytes, handleChangeMode, message } from '@/utils'

import { useModal } from '@/components/Modal'

import CommonController from './CommonController.vue'
import ConnectionsController from './ConnectionsController.vue'
import LogsController from './LogsController.vue'

const trafficHistory = ref<[number[], number[]]>([[], []])
const statistics = ref({
  upload: 0,
  download: 0,
  downloadTotal: 0,
  uploadTotal: 0,
  connections: [] as any[],
  inuse: 0,
})

const { t } = useI18n()
const [Modal, modalApi] = useModal({})
const appStore = useAppStore()
const envStore = useEnvStore()
const kernelApiStore = useKernelApiStore()

const handleRestartKernel = async () => {
  try {
    await kernelApiStore.restartKernel()
  } catch (error: any) {
    console.error(error)
    message.error(error)
  }
}

const handleStopKernel = async () => {
  try {
    await kernelApiStore.stopKernel()
  } catch (error: any) {
    console.error(error)
    message.error(error)
  }
}

const handleShowApiLogs = () => {
  modalApi.setProps({
    title: 'Logs',
    cancelText: 'common.close',
    width: '90',
    height: '90',
    submit: false,
    maskClosable: true,
  })
  modalApi.setContent(LogsController).open()
}

const handleShowApiConnections = () => {
  modalApi.setProps({
    title: 'home.overview.connections',
    cancelText: 'common.close',
    width: '90',
    height: '90',
    submit: false,
    maskClosable: true,
  })
  modalApi.setContent(ConnectionsController).open()
}

const handleShowSettings = () => {
  modalApi.setProps({
    title: 'home.overview.settings',
    cancelText: 'common.close',
    width: '90',
    submit: false,
    maskClosable: true,
  })
  modalApi.setContent(CommonController).open()
}

const onTunSwitchChange = async (enable: boolean) => {
  try {
    await kernelApiStore.updateConfig({ tun: { enable } })
  } catch (error: any) {
    console.error(error)
    message.error(error)
  }
}

const onSystemProxySwitchChange = async (enable: boolean) => {
  try {
    await envStore.switchSystemProxy(enable)
  } catch (error: any) {
    console.error(error)
    message.error(error)
    envStore.systemProxy = !envStore.systemProxy
  }
}

const unregisterMemoryHandler = kernelApiStore.onMemory((data) => {
  statistics.value.inuse = data.inuse
})

const unregisterTrafficHandler = kernelApiStore.onTraffic((data) => {
  const { up, down } = data
  statistics.value.upload = up
  statistics.value.download = down

  trafficHistory.value[0].push(up)
  trafficHistory.value[1].push(down)

  if (trafficHistory.value[0].length > 60) {
    trafficHistory.value[0].shift()
    trafficHistory.value[1].shift()
  }
})

const unregisterConnectionsHandler = kernelApiStore.onConnections((data) => {
  statistics.value.downloadTotal = data.downloadTotal
  statistics.value.uploadTotal = data.uploadTotal
  statistics.value.connections = data.connections || []
})

onUnmounted(() => {
  unregisterMemoryHandler()
  unregisterTrafficHandler()
  unregisterConnectionsHandler()
})
</script>

<template>
  <div>
    <div class="flex items-center rounded-8 px-8 py-4" style="background-color: var(--card-bg)">
      <Button @click="handleShowSettings" type="text" size="small">
        <Icon icon="settings" />
      </Button>
      <Switch
        v-model="envStore.systemProxy"
        @change="onSystemProxySwitchChange"
        size="small"
        border="square"
        class="ml-4"
      >
        {{ t('home.overview.systemProxy') }}
      </Switch>
      <Switch
        v-model="kernelApiStore.config.tun.enable"
        @change="onTunSwitchChange"
        size="small"
        border="square"
        class="ml-8"
      >
        {{ t('home.overview.tunMode') }}
      </Switch>
      <CustomAction :actions="appStore.customActions.core_state" />
      <Button
        @click="handleShowApiLogs"
        v-tips="'home.overview.viewlog'"
        type="text"
        size="small"
        icon="log"
        class="ml-auto"
      />
      <Button
        @click="handleRestartKernel"
        v-tips="'home.overview.restart'"
        type="text"
        size="small"
        icon="restart"
      />
      <Button
        @click="handleStopKernel"
        v-tips="'home.overview.stop'"
        type="text"
        size="small"
        icon="stop"
      />
    </div>
    <div class="flex mt-20 gap-12">
      <Card :title="t('home.overview.realtimeTraffic')" class="flex-1">
        <div class="py-8 text-12">
          ↑ {{ formatBytes(statistics.upload) }}/s ↓ {{ formatBytes(statistics.download) }}/s
        </div>
      </Card>
      <Card :title="t('home.overview.totalTraffic')" class="flex-1">
        <div class="py-8 text-12">
          ↑ {{ formatBytes(statistics.uploadTotal) }} ↓ {{ formatBytes(statistics.downloadTotal) }}
        </div>
      </Card>
      <Card
        @click="handleShowApiConnections"
        :title="t('home.overview.connections')"
        class="flex-1 cursor-pointer"
      >
        <div class="py-8 text-12">
          {{ statistics.connections.length }}
        </div>
      </Card>
      <Card :title="t('home.overview.memory')" class="flex-1">
        <div class="py-8 text-12">
          {{ formatBytes(statistics.inuse) }}
        </div>
      </Card>
    </div>
    <div class="flex">
      <div class="w-[60%]">
        <div class="py-16 font-bold" style="color: var(--card-color)">
          {{ t('home.overview.traffic') }}
        </div>
        <TrafficChart
          :series="trafficHistory"
          :legend="[t('home.overview.transmit'), t('home.overview.receive')]"
        />
      </div>
      <div class="ml-12 flex-1">
        <div class="py-16 font-bold" style="color: var(--card-color)">
          {{ t('kernel.mode') }}
        </div>
        <div class="flex flex-col gap-12">
          <Card
            v-for="mode in ModeOptions"
            :key="mode.value"
            :selected="kernelApiStore.config.mode === mode.value"
            @click="handleChangeMode(mode.value as any)"
            :title="t(mode.label)"
            class="cursor-pointer"
          >
            <div class="text-12 py-2">{{ t(mode.desc) }}</div>
          </Card>
        </div>
      </div>
    </div>
  </div>

  <Modal />
</template>
