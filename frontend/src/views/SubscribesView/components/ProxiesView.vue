<script setup lang="ts">
import { ref, computed, inject, h } from 'vue'
import { useI18n } from 'vue-i18n'
import { parse } from 'yaml'

import { ClipboardSetText, Readfile, Writefile } from '@/bridge'
import { DraggableOptions } from '@/constant'
import { useBool } from '@/hooks'
import { useSubscribesStore } from '@/stores'
import { deepClone, ignoredError, sampleID, stringifyNoFolding, message } from '@/utils'

import Button from '@/components/Button/index.vue'

import type { Menu, Subscription } from '@/types/app'

interface Props {
  sub: Subscription
}

const props = defineProps<Props>()

let editId = ''
const isEdit = ref(false)
const loading = ref(false)
const keywords = ref('')
const proxyType = ref('')
const details = ref()
const allFieldsProxies = ref<any[]>([])
const sub = ref(deepClone(props.sub))

const [showDetails, toggleDetails] = useBool(false)

const keywordsRegexp = computed(() => {
  try {
    return new RegExp(keywords.value, 'i')
  } catch {
    return keywords.value
  }
})

const filteredProxyTypeOptions = computed(() => {
  const proxyProtocols = sub.value.proxies.reduce((p, c) => {
    p[c.type] = (p[c.type] || 0) + 1
    return p
  }, {} as Recordable)
  return [{ label: 'All', value: '', count: 0 }].concat(
    Object.entries(proxyProtocols).map(([label, count]) => ({
      label: `${label}(${count})`,
      value: label,
      count,
    })),
  )
})

const filteredProxies = computed(() => {
  return sub.value.proxies.filter((v) => {
    const hitType = proxyType.value ? proxyType.value === v.type : true
    let hitName = true
    if (typeof keywordsRegexp.value === 'string') {
      hitName = v.name.toLowerCase().includes(keywordsRegexp.value.toLowerCase())
    } else {
      hitName = keywordsRegexp.value.test(v.name)
    }
    return hitName && hitType
  })
})

const menus: Menu[] = [
  {
    label: 'common.details',
    handler: async (record: Subscription['proxies'][0]) => {
      try {
        const proxy = await getProxyByName(record.name)
        details.value = stringifyNoFolding(proxy)
        isEdit.value = false
        toggleDetails()
      } catch (error: any) {
        message.error(error)
      }
    },
  },
  {
    label: 'common.copy',
    handler: async (record: Subscription['proxies'][0]) => {
      try {
        const proxy = await getProxyByName(record.name)
        await ClipboardSetText(stringifyNoFolding(proxy))
        message.success('common.copied')
      } catch (error: any) {
        message.error(error)
      }
    },
  },
  {
    label: 'common.edit',
    handler: async (record: Subscription['proxies'][0]) => {
      try {
        const proxy = await getProxyByName(record.name)
        details.value = stringifyNoFolding(proxy)
        isEdit.value = true
        editId = record.name
        toggleDetails()
      } catch (error: any) {
        message.error(error)
      }
    },
  },
  {
    label: 'common.delete',
    handler: (record: Record<string, any>) => {
      const idx = sub.value.proxies.findIndex((v) => v.name === record.name)
      if (idx !== -1) {
        sub.value.proxies.splice(idx, 1)
      }
    },
  },
]

const { t } = useI18n()
const subscribeStore = useSubscribesStore()

const handleCancel = inject('cancel') as any
const handleSubmit = inject('submit') as any

const handleSave = async () => {
  loading.value = true
  try {
    const { path, proxies, id } = sub.value
    await initAllFieldsProxies()
    const filteredProxies = allFieldsProxies.value.filter((v: any) =>
      proxies.some((vv) => vv.name === v.name),
    )
    const sortedArray = proxies.map((v) => filteredProxies.find((vv) => vv.name === v.name))
    await Writefile(path, stringifyNoFolding({ proxies: sortedArray }))
    await subscribeStore.editSubscribe(id, sub.value)
    handleSubmit()
  } catch (error: any) {
    console.log(error)
    message.error(error)
  }
  loading.value = false
}

const handleAdd = async () => {
  editId = ''
  details.value = ''
  isEdit.value = true
  toggleDetails()
}

const onEditEnd = async () => {
  let proxy: any
  try {
    proxy = parse(details.value)

    if (typeof proxy !== 'object') throw 'wrong format'
  } catch (error: any) {
    console.log(error)
    message.error(error.message || error)
    // reopen
    toggleDetails()
    return
  }

  await initAllFieldsProxies()

  const allFieldsProxiesIdx = allFieldsProxies.value.findIndex((v: any) => v.name === editId)
  const subProxiesIdx = sub.value.proxies.findIndex((v) => v.name === editId)

  if (allFieldsProxiesIdx !== -1 && subProxiesIdx !== -1) {
    allFieldsProxies.value.splice(allFieldsProxiesIdx, 1, proxy)
    sub.value.proxies.splice(subProxiesIdx, 1, {
      ...sub.value.proxies[subProxiesIdx],
      name: proxy.name,
    })
  } else {
    allFieldsProxies.value.push(proxy)
    sub.value.proxies.push({
      id: sampleID(),
      name: proxy.name,
      type: proxy.type,
    })
  }
}

const initAllFieldsProxies = async () => {
  if (allFieldsProxies.value.length) return
  const content = (await ignoredError(Readfile, sub.value!.path)) || '{}'
  const { proxies: _proxies = [] } = parse(content)
  allFieldsProxies.value = _proxies
}

const getProxyByName = async (name: string) => {
  await initAllFieldsProxies()
  const proxy = allFieldsProxies.value.find((v: any) => v.name === name)
  if (!proxy) throw 'Proxy Not Found'
  return proxy
}

const modalSlots = {
  cancel: () =>
    h(
      Button,
      {
        disabled: loading.value,
        onClick: handleCancel,
      },
      () => t('common.cancel'),
    ),
  submit: () =>
    h(
      Button,
      {
        type: 'primary',
        loading: loading.value,
        onClick: handleSave,
      },
      () => t('common.save'),
    ),
}

defineExpose({ modalSlots })
</script>

<template>
  <div class="h-full flex flex-col">
    <div class="flex items-center">
      <span class="mr-8">
        {{ t('subscribes.proxies.type') }}
        :
      </span>
      <Select v-model="proxyType" :options="filteredProxyTypeOptions" size="small" />
      <Input
        v-model="keywords"
        :placeholder="t('subscribes.proxies.name')"
        clearable
        auto-size
        size="small"
        class="mx-8 flex-1"
      />
      <Button @click="handleAdd" type="primary" size="small">
        {{ t('subscribes.proxies.add') }}
      </Button>
    </div>

    <Empty v-if="filteredProxies.length === 0" />

    <div
      v-else
      v-draggable="[sub.proxies, DraggableOptions]"
      class="grid grid-cols-4 gap-8 mt-8 overflow-y-auto"
    >
      <Card
        v-for="proxy in filteredProxies"
        :key="proxy.name"
        :title="proxy.name"
        v-menu="menus.map((v) => ({ ...v, handler: () => v.handler?.(proxy) }))"
      >
        <div class="text-12">
          {{ proxy.type }}
        </div>
      </Card>
    </div>
  </div>

  <Modal
    v-model:open="showDetails"
    :submit="isEdit"
    :mask-closable="!isEdit"
    :title="isEdit ? (details ? 'common.edit' : 'common.add') : 'common.details'"
    :on-ok="onEditEnd"
    cancel-text="common.close"
    max-height="80"
    max-width="80"
  >
    <CodeViewer v-model="details" :editable="isEdit" lang="yaml" />
  </Modal>
</template>
